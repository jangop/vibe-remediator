# JavaScript / TypeScript Remediation Playbook

## 1. Schema-driven network boundaries

TypeScript's `as` keyword is a lie — it asserts a type at compile time without checking anything at runtime. Vibe-coded code is full of:

```ts
const data = await res.json() as User[];  // checks nothing
```

If the API returns `{ error: "..." }`, the code crashes deep inside the UI with a confusing message.

**Fix: parse with Zod** (or Valibot, ArkType, io-ts — pick one and stick with it):

```ts
import { z } from "zod";

const User = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});
type User = z.infer<typeof User>;

async function fetchUsers() {
  const res = await fetch("/api/users");
  return z.array(User).parse(await res.json());
}
```

Apply this to:
- API responses
- Form submissions (`zodResolver` with react-hook-form, or `superforms` in SvelteKit)
- URL query parameters
- Cookies and localStorage reads
- Webhook payloads

## 2. Push logic to the server

AI reaches for client-side `useEffect` / `onMount` fetching even in meta-frameworks designed to avoid it. Signs to look for:

- `useEffect` that fetches data on mount
- `onMount` in Svelte components doing network calls
- Loading spinners on every page

**Where data fetching belongs by framework:**

| Framework | Place |
|---|---|
| **Next.js App Router** | Server Components — `async function Page()` with `await`. Client Components only for interactivity. |
| **SvelteKit** | `+page.server.ts` `load` function; `+server.ts` for API routes. |
| **Nuxt** | `useFetch` in `<script setup>` (auto-runs on server). |
| **Remix** | `loader` function in the route module. |

Move the fetch to the server boundary; serialize only what the client needs. Example (Next.js App Router):

```tsx
export default async function UserPage({ params }) {
  const user = User.parse(await fetchUser(params.id));
  return <UserCard user={user} />;
}
```

The loading spinner disappears, SEO improves, and the failure mode shifts from "broken UI" to "broken page" (handle with `error.tsx`).

## 3. Untangle state

### Svelte 5 — migrate to runes

If the codebase has stores (`writable`, `readable`, `derived`) sprinkled across files, migrate to runes. Reactivity becomes explicit and local.

```ts
// before — invisible reactivity via store subscription
export const cart = writable<CartItem[]>([]);

// after — explicit reactivity, scoped to a class
class CartState {
  items = $state<CartItem[]>([]);
  total = $derived(this.items.reduce((s, i) => s + i.price, 0));
}
export const cart = new CartState();
```

### React — colocate, don't globalize

AI tends to hoist state to global contexts even when one component owns it. Audit each context:

- Used in more than one subtree? If no → push it down.
- Is it actually shared, or is it props in disguise? → just pass props.
- Is it server state? → React Query / SWR / `use()` with cache, not a custom context.

When global state is genuinely needed (auth, theme, feature flags), use a small library (Zustand, Jotai) over a hand-rolled provider chain.

## 4. Break up God components

Find the largest files first:

```bash
find . -name "*.tsx" -o -name "*.svelte" | xargs wc -l | sort -rn | head -20
```

Typical God components mix imports, types, data fetching, form state, event handlers, helpers, and JSX all in one file. Extract in this order (least risky first):

1. **Pure UI primitives.** `Button`, `Card`, `Badge` — props in, markup out. Zero behavior.
2. **Pure utility functions.** Anything not touching React/Svelte state. Move to `lib/` or `utils/`.
3. **Custom hooks / composables.** Bundle of related state + effects (`useCart`, `useFormValidation`).
4. **Feature sub-components.** Sections that own their own state (`<UserProfileSection>`).

What's left should be a page that wires sub-components to data — no business logic, no rendering details.

## 5. Silent catch-alls

Detection:

```bash
grep -rnE 'catch\s*\([^)]*\)\s*\{\s*\}|\.catch\(\(\)\s*=>\s*\{\}\)' --include="*.ts" --include="*.tsx" --include="*.js" .
```

Replacement strategies are in `SKILL.md` Phase 5. JS-specific channels for the "log and re-raise" option: surface to UI via error boundary / toast / form error, or to monitoring via Sentry / Datadog.

## 6. Linting and strict TS

Bare-minimum `tsconfig.json` strictness:

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

`noUncheckedIndexedAccess` is the most impactful — it forces handling `undefined` from array/object access, which catches a huge class of vibe-coded crashes. `exactOptionalPropertyTypes` is also valuable but notoriously thorny to retrofit; add it after the simpler flags are clean.

ESLint with `@typescript-eslint/strict-type-checked` + `@typescript-eslint/stylistic-type-checked` covers most of the rest.

## 7. Auditing dependencies

Ask each dependency to earn its place. Common candidates worth questioning:

- `moment` — `date-fns` or stdlib `Intl.DateTimeFormat` / `Temporal` usually suffice.
- `lodash` used only for `pick` / `omit` / `groupBy` — ES2023 array/object methods cover most cases.
- `axios` in a modern codebase — `fetch` is now native and capable.
- `uuid` for simple ID generation — `crypto.randomUUID()` is built in.
- Multiple state libraries (Redux + Zustand + Context) or multiple form libraries (Formik + react-hook-form) — consolidate on whichever already has wider coverage.
- Unused Babel/Webpack plugins after a Vite/Turbopack migration.

Don't strip a dependency because it's "old-fashioned" — only because the project would be simpler without it.

## 8. N+1 in client-side land

Frontend N+1 is real:

```tsx
{users.map(user => (
  <div key={user.id}>
    <UserPosts userId={user.id} />  {/* each renders its own fetch */}
  </div>
))}
```

Fix by hoisting the fetch:

```tsx
const users = await fetchUsersWithPosts();  // server-side, one query

{users.map(user => (
  <div key={user.id}>
    <UserPosts posts={user.posts} />  {/* no fetch in child */}
  </div>
))}
```

When a single round-trip isn't possible, batch with the DataLoader pattern on the server.
