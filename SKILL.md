---
name: vibe-remediator
description: Audit and incrementally remediate vibe-coded (LLM-generated, rapidly prototyped, AI-written) codebases. Use this skill whenever the user wants to clean up, harden, productionize, refactor, or "fix" a chaotic AI-generated codebase — including phrases like "this codebase is a mess", "AI slop", "vibe coded", "prototype that needs to scale", "make this production ready", "audit my codebase for anti-patterns", or when they mention specific symptoms like God components, N+1 queries, missing tests, silent catch-alls, type illusion, prompt drift, or async/sync hazards. Trigger it even when the user doesn't use the word "vibe" — anytime a codebase shows signs of being LLM-generated and needs systematic remediation, this skill applies.
---

When an LLM blasts out a solution, it optimizes for the next token, not the next year of maintenance. This skill is a structured workflow for finding and fixing the systemic flaws that result — without falling into the "ground-up rewrite" trap.

## Core principles

1. **Never rewrite from scratch.** A full rewrite is almost always a trap: it loses encoded business logic, delays features, and re-introduces the same bugs in new locations. Use the **Strangler Fig pattern** — build well-architected pieces alongside the mess and migrate piece by piece.

2. **Lock down the edges first.** Vibe code fails at boundaries. Adopt "parse, don't validate" — the moment data enters the system, parse it through a strict schema; downstream code then trusts the structure unconditionally. This creates a trust region inside which you can refactor safely.

3. **Tests are the safety net, not the goal.** You cannot unit-test spaghetti — discrete "units" of logic don't exist. Write high-level black-box integration tests (payload X in, response Y out) **before** refactoring. They give you the confidence to change internals.

4. **Be pragmatic, not dogmatic.** This skill recommends Pydantic, Zod, Ruff, Repository pattern, etc. because they fit common stacks. But if the codebase already uses Joi, attrs, Drizzle, or some other equivalent — extend what's there. The goal is *consistent boundaries*, not specific libraries.

5. **One pass at a time.** Don't try to fix every anti-pattern simultaneously. Pick the highest-leverage one for the user's current pain, fix it across the codebase, then move to the next.

## Workflow

### Phase 1 — Triage (audit)

Map the blast radius before touching anything. Produce a written audit, share it with the user, and get alignment on priorities **before** making code changes.

**Use the codebase graph tools** (`mcp__codebase-memory-mcp__*`) as the primary discovery mechanism — they surface call chains, fan-out, and cross-service edges that text search cannot. If the repo is not indexed, run `index_repository` first. Fall back to `Grep`/`Read` only for configs and non-code files.

Walk the [anti-pattern checklist](#anti-patterns-detection-signals) below. For each, record:

- **Present?** (yes / no / partial)
- **Severity** (blocker / high / medium / low) — based on user-facing impact, not aesthetic
- **Blast radius** (a few files / a module / cross-cutting) — drives the remediation strategy
- **Concrete examples** — quote 1-3 specific file paths or symbols so the user can verify

End the triage with a **prioritized remediation plan**:

1. Group findings into 3-5 phases (boundaries → safety net → structural → polish)
2. State the dependency order ("we can't refactor X until tests exist around Y")
3. Estimate effort in rough buckets (hours / day / week)
4. Identify what *not* to fix yet (out of scope or low-leverage)

Then **pause and ask the user**: which phase do they want to tackle first, and how much code change are they comfortable with this session?

### Phase 2 — Lock down the edges

- Find every place data enters the system: HTTP route handlers, queue consumers, file ingestion, CLI inputs, third-party webhooks, deserialized cache entries.
- Wrap each in a parser (Pydantic / Zod / attrs / whatever the project already uses). If nothing exists, pick one and apply it consistently.
- Do the same for outbound data — responses, database writes, external API calls.
- Replace `as MyType`, `cast()`, and `dict()` unpacking with parsed types.
- Update or remove `Any` / `dict[str, Any]` / `Record<string, unknown>` annotations once schemas are in place.

### Phase 3 — Build the safety net

Add black-box integration tests against the *current* behavior (even the buggy parts — you're locking in observable behavior, not correctness). Test through the public interface: HTTP, CLI, or message bus. Avoid mocking internal code; mocks lock in the structure you're trying to change.

Tools: `pytest` + `httpx`/`TestClient` for Python APIs; `vitest`/`playwright`/`supertest` for JS; record-replay (`vcr.py`, `nock`, MSW) for third-party calls.

### Phase 4 — Structural cleanup

- **Isolate the database.** Create a Data Access Layer (DAL) / Repository. Force all queries through it. This is where N+1 patterns become visible and fixable.
- **Componentize God files.** Split mega-files into single-responsibility units. In frontends: extract pure presentational components, then extract data hooks, leave orchestration in the page.
- **Untangle state.** In Svelte 5, migrate stores to runes (`$state`, `$derived`). In React, colocate state to the component that owns it; rip out unnecessary global contexts.
- **Push logic to the server.** Move client-side data fetching into server loaders (`+page.server.ts`, Next.js Server Components, Nuxt server routes). Ship only what the client needs.

### Phase 5 — Hygiene pass

- **Replace silent catch-alls** (`except Exception: pass`, `catch (e) {}`) with explicit handling — narrow the exception type, re-raise as a domain error, or log structured info and re-raise at the top level. Bare swallows are bugs.
- **Audit dependencies.** Remove deprecated or unused packages. Check that anything load-bearing is actually maintained.
- **Run aggressive linters in strict mode.** Ruff for Python; ESLint + TypeScript strict for JS/TS. Fix the autofixable warnings; create issues for the rest rather than disabling rules.
- **Add intent documentation.** Not "what" comments — *why* comments at non-obvious decisions. Architecture decision records (ADRs) for cross-cutting choices.

## Anti-patterns: detection signals

When triaging, look for these patterns. The longer-form failure modes (7, 9, 12, 14, 15) are detailed in [Notable failure modes](#notable-failure-modes) below the table.

| # | Anti-pattern | Detection signal |
|---|---|---|
| 1 | **Prompt drift** | Compare 2-3 sibling modules — same data model, error strategy, naming? |
| 2 | **God components** | Files >500 lines, or `useEffect` + JSX + handlers + types in one file. |
| 3 | **Type illusion** | Grep `Any`, `dict[str, Any]`, `: any`, `Record<string, unknown>`, `as` casts, `# type: ignore`, `@ts-ignore`. |
| 4 | **Happy-path obsession** | Missing `try`/`except` around I/O, no HTTP timeouts, no handling of `None`/empty lists. |
| 5 | **Database query chaos** | ORM calls inside loops, missing `.select_related` / `JOIN`s, missing indexes, `SELECT *`. |
| 6 | **Hallucinated/bloated deps** | Cross-reference each dep in `package.json` / `pyproject.toml`; look for deprecated, archived, or stdlib-overlapping packages. |
| 7 | **Silent failures end-to-end** | See [below](#7-silent-failures-end-to-end). |
| 8 | **State mutation spaghetti** | Global mutable stores, prop drilling >3 levels, `useEffect` chains, implicit subscriptions. |
| 9 | **Context window amnesia** | See [below](#9-context-window-amnesia). |
| 10 | **Sync blocking in async** | `requests.get`, sync DB drivers, `time.sleep`, blocking file I/O inside `async def`. |
| 11 | **Missing test harness** | Check for `tests/` or `__tests__/`. Run `pytest --collect-only` / `vitest --run`. |
| 12 | **Broken authorization** | See [below](#12-broken-authorization). |
| 13 | **Intent-free documentation** | Comments explain "what" not "why". No ADR / decision log. |
| 14 | **Webhook state-machine gaps** | See [below](#14-webhook-state-machine-gaps). |
| 15 | **Secrets in AI chat history** | See [below](#15-secrets-in-ai-chat-history). |

### Notable failure modes

#### 7. Silent failures end-to-end
Failures usually break in three layers at once, so check all three:
- **Code:** `except Exception:`, `catch (e) {}`, `.catch(() => {})`.
- **UX:** what does the user see when an external API times out — blank screen? infinite spinner? frozen form?
- **Observability:** is there *any* error tracking — Sentry, Datadog, structured logs?

#### 9. Context window amnesia
Two sub-failures:
- **Drift:** similar functions in different modules enforce different constraints.
- **Duplicate registrations:** the *same* trigger fires *two* near-identical workflows because the AI generated the feature twice under slightly different names (`sendWelcomeEmail` vs `notifyNewUser`, two `POST /signup` side-effects, two Stripe customer creations). Sort handlers / routes / cron entries alphabetically and scan for semantic near-duplicates.

#### 12. Broken authorization
The big one in multi-tenant apps: **authentication ≠ authorization**. The AI builds login, then routes filter only by "is logged in," not by "owns this row." Test by logging in as user A, hitting an endpoint with user B's resource ID, and seeing if it returns. Also: unsanitized SQL string interpolation, missing CSRF tokens, `dangerouslySetInnerHTML`, `eval`, missing rate limits.

#### 14. Webhook state-machine gaps
The AI implements the success event and stops. For Stripe: `checkout.session.completed` exists, but `customer.subscription.deleted`, `invoice.payment_failed`, `charge.refunded`, `customer.subscription.updated` are missing — so cancelled users keep accessing paid features and failed payments aren't surfaced. Same shape for any webhook source (PayPal, GitHub, Shopify, Twilio): list every event the upstream emits, then check coverage.

#### 15. Secrets in AI chat history
Vibe-coding-specific. Any secret pasted into an LLM chat (Stripe keys, DB passwords, API tokens, service-role keys) now lives in a transcript on a platform whose security posture you don't control. Even if the chat is "private," it has likely been logged or could leak via a vendor breach. Ask the user whether they've pasted a real key while debugging. If yes, rotate every secret that was ever pasted, and switch to placeholder-based debugging — the chat doesn't need the actual value to write working code.

## Language-specific deep dives

Load the relevant playbook on demand:

- **Python** (FastAPI, Pydantic, Ruff, mypy, async hazards): `references/python-playbook.md`
- **JavaScript / TypeScript** (Zod, Next.js / SvelteKit / Nuxt, Svelte 5 runes, component extraction): `references/javascript-playbook.md`

## Communication tips

- **Don't dump the full anti-pattern list at the user.** Triage on their actual codebase, report what's *there*, and skip the rest.
- **Quote real file paths and symbols** in the audit. "There's an N+1 in `services/orders.py:list_orders`" is useful; "you might have N+1 queries" is not.
- **Get explicit approval before structural changes.** Boundaries and tests can usually go in without much risk; restructuring state, splitting components, and changing DB layers deserve a checkpoint.
- **Surface tradeoffs honestly.** Some fixes (introducing a DAL, migrating to runes) have ongoing cost. Say so.
- **Push back on rewrites and "fix everything now" requests.** Re-explain the Strangler Fig pattern and propose a phased plan — pick the most painful phase first.

---

## Self-improvement: report findings upstream

This skill is open source. You — the agent running it on a user's codebase — are *not* its maintainer. Don't edit `SKILL.md` or `references/*` in place; the user running you didn't sign up to ship skill upgrades. Instead, when you notice a way this skill could be sharper, file an issue against the upstream repo.

**Upstream:** `https://github.com/jangop/vibe-remediator` — use `gh issue create --repo jangop/vibe-remediator` if `gh` is available; otherwise show the user a pre-filled issue body they can paste in.

**When to reflect:** session wrap-up, end of a remediation phase, or a "surprise moment" (a signal misfired, a step you always skip turned out to matter). Ask the user before filing — the issue is public and attributed to their account.

**Before filing, see `references/contributing.md`** for the anti-overfitting tests and the issue template. Most observations don't pass those tests, which is fine — the skill stays useful by getting *sharper*, not bigger.
