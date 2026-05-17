# JavaScript / TypeScript Remediation Playbook

The frontend ecosystem is uniquely vulnerable to two failure modes: God components (everything in one `.tsx`/`.svelte` file) and cascading state failures. Both compound with the third frontend hazard — implicit trust in API response shapes.

## 1. Schema-driven network boundaries

TypeScript's `as` keyword is a lie — it asserts a type at compile time without checking anything at runtime. Vibe-coded code is full of:

```ts
const data = await res.json() as User[];  // 🚨 this checks nothing
```

If the API returns `{ error: "..." }`, your code crashes deep inside the UI with a confusing message.

**Fix: parse with Zod (or Valibot, ArkType, io-ts — pick one and stick with it).**

```ts
import { z } from "zod";

const User = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

const UserList = z.array(User);
type User = z.infer<typeof User>;

async function fetchUsers() {
  const res = await fetch("/api/users");
  return UserList.parse(await res.json());  // 🔒 trust everything downstream
}
```

Apply this to:
- API responses (above)
- Form submissions (`zodResolver` with react-hook-form, or `superforms` in SvelteKit)
- URL query parameters
- Cookies and localStorage reads
- Webhook payloads

## 2. Push logic to the server

AI loves `useEffect`. Loves it. It will reach for client-side fetching every time, even in meta-frameworks designed to avoid it.

**Signs to look for:**
- `useEffect` that fetches data on mount
- `onMount` in Svelte components doing network calls
- Loading spinners everywhere because every page double-fetches

**Fix by framework:**

| Framework | Where data fetching belongs |
|---|---|
| **Next.js App Router** | Server Components (default). Use `async function Page()` and `await` directly. Only use Client Components for interactivity. |
| **SvelteKit** | `+page.server.ts` `load` function. Pass data to the page via `data` prop. Use `+server.ts` for API routes. |
| **Nuxt** | `useFetch` in `<script setup>` (auto-runs on server). Avoid `$fetch` inside `onMounted` unless it must be client-only. |
| **Remix** | `loader` function in the route module. |

Move the fetch out of `useEffect` / `onMount` and into the server boundary. Serialize only what the client needs.

**Before:**
```tsx
export default function UserPage({ params }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch(`/api/users/${params.id}`).then(r => r.json()).then(setUser);
  }, [params.id]);
  if (!user) return <Spinner />;
  return <UserCard user={user} />;
}
```

**After (Next.js App Router):**
```tsx
export default async function UserPage({ params }) {
  const user = User.parse(await fetchUser(params.id));
  return <UserCard user={user} />;
}
```

The loading spinner disappears, the SEO improves, the failure mode shifts from "broken UI" to "broken page" (much easier to handle with `error.tsx`).

## 3. Untangle state

### Svelte 5 — migrate to runes

If the codebase has stores (`writable`, `readable`, `derived`) sprinkled across files, migrate to runes. Runes make reactivity explicit and local.

**Before (Svelte 4 store):**
```ts
// stores/cart.ts
import { writable, derived } from "svelte/store";
export const cart = writable<CartItem[]>([]);
export const total = derived(cart, ($cart) => $cart.reduce((s, i) => s + i.price, 0));
```

**After (Svelte 5 runes):**
```ts
// state/cart.svelte.ts
class CartState {
  items = $state<CartItem[]>([]);
  total = $derived(this.items.reduce((s, i) => s + i.price, 0));

  add(item: CartItem) { this.items.push(item); }
  remove(id: string) { this.items = this.items.filter(i => i.id !== id); }
}

export const cart = new CartState();
```

Reactivity is now explicit (the `$state` rune marks what's reactive). No more invisible subscriptions.

### React — colocate, don't globalize

AI tends to hoist state to global contexts even when only one component needs it. Audit each context:

- Is it used in more than one subtree? If no → push it down to where it's used.
- Is it actually shared state, or is it props in disguise? → use prop passing.
- Is it server state (data from an API)? → React Query / SWR / `use()` with cache, not a custom context.

When global state is truly needed (auth, theme, feature flags), use a small library (Zustand, Jotai) instead of a hand-rolled provider chain.

## 4. Break up God components

Find your largest component files first:

```bash
find . -name "*.tsx" -o -name "*.svelte" | xargs wc -l | sort -rn | head -20
```

A typical God component looks like:
```
- imports (50 lines)
- type definitions (30 lines)
- data fetching (40 lines)
- form state (60 lines)
- event handlers (100 lines)
- helper functions (50 lines)
- JSX (200 lines)
```

**Extraction order** (least risky first):

1. **Pure UI primitives.** `Button`, `Card`, `Badge` — anything that takes props and returns markup. Zero behavior. Easy to extract, instant readability win.
2. **Pure utility functions.** Anything that doesn't touch React/Svelte state. Move to `lib/` or `utils/`.
3. **Custom hooks / composables.** Bundle of related state + effects. Move `useCart`, `useFormValidation`, etc. out.
4. **Feature sub-components.** Sections of the page that have their own state (`<UserProfileSection>`, `<OrderSummary>`).
5. **The orchestrator.** What's left should be a page that wires sub-components together with data — no business logic, no rendering details.

## 5. Silent catch-alls

Search and rewrite:

```bash
grep -rn "catch (e) {}" --include="*.ts" --include="*.tsx" --include="*.js" .
grep -rn "\.catch(() => {})" --include="*.ts" --include="*.tsx" --include="*.js" .
```

Replace with one of:
- Narrow handler that knows what to do (`if (err instanceof TimeoutError) { retry(); }`)
- Surface to UI via error boundary / toast / form error
- Log to monitoring (Sentry, Datadog) and re-throw

Same rule as Python: never swallow silently.

## 6. Linting and strict TS

Bare minimum `tsconfig.json` strictness:

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true
  }
}
```

`noUncheckedIndexedAccess` is the most impactful — it forces you to handle `undefined` from array/object access, which catches a huge class of vibe-coded crashes.

ESLint with `@typescript-eslint/strict-type-checked` + `@typescript-eslint/stylistic-type-checked` configs catches the rest.

## 7. Common bloated dependencies to audit

- `moment` → `date-fns` or stdlib `Intl.DateTimeFormat` / `Temporal`
- `lodash` for trivial uses → ES2023 array/object methods
- `axios` → `fetch` (now native and good)
- `uuid` for simple ID generation → `crypto.randomUUID()`
- Multiple state libraries (Redux + Zustand + Context) — consolidate
- Multiple form libraries (Formik + react-hook-form) — pick one
- Unused Babel/Webpack plugins after a Vite/Turbopack migration

## 8. N+1 in client-side land

Frontend N+1 is real:

```tsx
{users.map(user => (
  <div key={user.id}>
    {user.name}
    <UserPosts userId={user.id} />  {/* 🚨 each renders its own fetch */}
  </div>
))}
```

Fix by hoisting the fetch:

```tsx
const users = await fetchUsersWithPosts();  // server-side, one query

{users.map(user => (
  <div key={user.id}>
    {user.name}
    <UserPosts posts={user.posts} />  {/* 🔒 no fetch in child */}
  </div>
))}
```

Or, when a single round-trip isn't possible, batch with DataLoader pattern on the server.
