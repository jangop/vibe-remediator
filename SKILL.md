---
name: vibe-remediator
description: Audit and incrementally remediate vibe-coded (LLM-generated, rapidly prototyped, AI-written) codebases. Use this skill whenever the user wants to clean up, harden, productionize, refactor, or "fix" a chaotic AI-generated codebase — including phrases like "this codebase is a mess", "AI slop", "vibe coded", "prototype that needs to scale", "make this production ready", "audit my codebase for anti-patterns", or when they mention specific symptoms like God components, N+1 queries, missing tests, silent catch-alls, type illusion, prompt drift, or async/sync hazards. Trigger it even when the user doesn't use the word "vibe" — anytime a codebase shows signs of being LLM-generated and needs systematic remediation, this skill applies.
---

# Vibe Remediator — Taming AI-Generated Codebases

When an LLM blasts out a solution, it optimizes for the next token, not the next year of maintenance. This skill is a structured workflow for finding and fixing the systemic flaws that result — without falling into the "ground-up rewrite" trap.

## Core principles (read these first)

1. **Never rewrite from scratch.** A full rewrite is almost always a trap: it loses encoded business logic, delays features, and re-introduces the same bugs in new locations. Use the **Strangler Fig pattern** — build well-architected pieces alongside the mess and migrate piece by piece.

2. **Lock down the edges first.** Vibe code fails at the boundaries. Adding strict input/output validation at API and database boundaries gives you a stable trust region inside which you can refactor safely. Adopt "parse, don't validate" — the moment data enters the system, parse it through a strict schema; downstream code then trusts the structure unconditionally.

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

This is almost always phase one of the actual fixes — it's the foundation that makes everything else safe.

- Find every place data enters the system: HTTP route handlers, queue consumers, file ingestion, CLI inputs, third-party webhooks, deserialized cache entries.
- Wrap each in a parser (Pydantic / Zod / attrs / whatever the project already uses). If nothing exists, pick one and apply it consistently.
- Do the same for outbound data — responses, database writes, external API calls.
- Replace `as MyType`, `cast()`, and `dict()` unpacking with parsed types.
- Update or remove `Any` / `dict[str, Any]` / `Record<string, unknown>` annotations once the schemas are in place.

After this phase, the *interior* of the application can be refactored without fear of mystery-shaped data.

### Phase 3 — Build the safety net

Add black-box integration tests against the *current* behavior (even the buggy parts — you're locking in observable behavior, not correctness). Test through the public interface: HTTP, CLI, or message bus. Avoid mocking internal code; mocks lock in the structure you're trying to change.

Tools: `pytest` + `httpx`/`TestClient` for Python APIs; `vitest`/`playwright`/`supertest` for JS; record-replay (`vcr.py`, `nock`, MSW) for third-party calls.

Get the suite passing on `main` before any structural changes. Now you can refactor with feedback.

### Phase 4 — Structural cleanup

Now the safer changes become possible:

- **Isolate the database.** Create a Data Access Layer (DAL) / Repository. Force all queries through it. This is where N+1 patterns become visible and fixable.
- **Componentize God files.** Split mega-files into single-responsibility units. In frontends: extract pure presentational components, then extract data hooks, leave orchestration in the page.
- **Untangle state.** In Svelte 5, migrate stores to runes (`$state`, `$derived`). In React, colocate state to the component that owns it; rip out unnecessary global contexts.
- **Push logic to the server.** Move client-side data fetching into server loaders (`+page.server.ts`, Next.js Server Components, Nuxt server routes). Ship only what the client needs.

### Phase 5 — Hygiene pass

- **Replace silent catch-alls** (`except Exception: pass`, `catch (e) {}`) with explicit handling. Either re-raise, log structured info, or convert to a domain error. Bare swallows are bugs.
- **Audit dependencies.** Remove deprecated or unused packages. Check that anything load-bearing is actually maintained.
- **Run aggressive linters in strict mode.** Ruff for Python; ESLint + TypeScript strict for JS/TS. Fix the autofixable warnings; create issues for the rest rather than disabling rules.
- **Add intent documentation.** Not "what" comments — *why* comments at non-obvious decisions. Architecture decision records (ADRs) for cross-cutting choices.

## Anti-patterns: detection signals

When triaging, look for these patterns. The "signal" column tells you what to grep / query / inspect.

| # | Anti-pattern | Detection signal |
|---|---|---|
| 1 | **Prompt drift** (different modules, different paradigms) | Compare 2-3 sibling modules: do they use the same data model, same error strategy, same naming? |
| 2 | **God components** | Find files >500 lines or with both data fetching and complex UI. In React/Svelte: `useEffect` + JSX + handlers + types all in one file. |
| 3 | **Type illusion** | Grep `Any`, `dict[str, Any]`, `: any`, `Record<string, unknown>`, `as` casts, `# type: ignore`, `@ts-ignore`. |
| 4 | **Happy-path obsession** | Look for missing `try`/`except` around I/O, no timeouts on outbound HTTP, no validation of optional fields, no handling of empty lists/None. |
| 5 | **Database query chaos** | Look for ORM calls inside loops, missing `.select_related` / `JOIN`s, missing indexes on filter columns, raw `SELECT *`. |
| 6 | **Hallucinated/bloated deps** | Audit `package.json` / `pyproject.toml`. Cross-reference each dep against npm/PyPI. Look for deprecated, archived, or single-use deps that wrap stdlib. |
| 7 | **Silent failures end-to-end** | Three places to check: code (`except Exception:`, `catch (e) {}`, `.catch(() => {})`); UX (what does the user see when an external API times out — blank screen? infinite spinner? frozen form?); observability (is there *any* error tracking — Sentry, Datadog, structured logs?). All three layers usually fail together. |
| 8 | **State mutation spaghetti** | Look for globally mutable stores, prop drilling >3 levels, `useEffect` chains, implicit subscriptions. |
| 9 | **Context window amnesia** (contradictory rules *and* duplicate registrations) | Two sub-failures: (a) similar functions in different modules enforce different constraints (drift); (b) the *same* trigger fires *two* near-identical workflows because the AI generated the feature twice under slightly different names (`sendWelcomeEmail` vs `notifyNewUser`, two `POST /signup` side-effects, two Stripe customer creations). Sort handlers/routes/cron entries alphabetically and scan for semantic near-duplicates. |
| 10 | **Sync blocking in async** | `requests.get`, sync DB drivers, `time.sleep`, blocking file I/O inside `async def`. In Node: long sync loops in handlers. |
| 11 | **Missing test harness** | Check for `tests/` or `__tests__/` directories. Run `pytest --collect-only` / `vitest --run --reporter=verbose`. |
| 12 | **Security blindspots** (incl. broken object-level authorization) | The big one in multi-tenant apps: **authentication ≠ authorization**. The AI builds login, then routes filter only by "is logged in," not by "owns this row." Test by logging in as user A, hitting an endpoint with user B's resource ID, and seeing if it returns. Also: unsanitized SQL string interpolation, missing CSRF tokens, `dangerouslySetInnerHTML`, `eval`, missing rate limits, no per-user resource scoping in queries. |
| 13 | **Intent-free documentation** | Scan comments — are they "what" (`# loop through users`) or "why" (`# users sorted by signup_date to preserve onboarding cohort`)? Check for an ADR / decision log. |
| 14 | **Webhook / event-handler state-machine gaps** | The AI implements the success event and stops. For Stripe: `checkout.session.completed` exists, but `customer.subscription.deleted`, `invoice.payment_failed`, `charge.refunded`, `customer.subscription.updated` are missing — so cancelled users keep accessing paid features, failed payments aren't surfaced. Same shape for any webhook source (PayPal, GitHub, Shopify, Twilio): list every event the upstream emits, then check coverage. |
| 15 | **Secrets pasted into AI chat history** | Vibe-coding-specific. Any secret pasted into an LLM chat (Stripe keys, DB passwords, API tokens, service role keys) now lives in a transcript on a platform whose security posture you don't control. Even if the chat is "private," it has likely been logged, used for fine-tuning consent, or leaked in a vendor breach. Check: has the user ever pasted a real key while debugging? If yes, rotate every secret that was ever pasted, and switch to placeholder-based debugging (`STRIPE_SECRET_KEY` as a variable name only). |

## Language-specific deep dives

For language-specific remediation tactics, read the relevant playbook:

- **Python** (FastAPI, Pydantic, Ruff, mypy, async hazards): `references/python-playbook.md`
- **JavaScript / TypeScript** (Zod, Next.js / SvelteKit / Nuxt, Svelte 5 runes, component extraction): `references/javascript-playbook.md`

Read these on demand — only load the playbook for the stack the codebase actually uses.

## Communication tips

- **Don't dump the full anti-pattern list at the user.** Triage on their actual codebase, report what's *there*, and skip the rest.
- **Quote real file paths and symbols** in the audit. "There's an N+1 in `services/orders.py:list_orders`" is useful; "you might have N+1 queries" is not.
- **Get explicit approval before structural changes.** Boundaries and tests can usually go in without much risk; restructuring state, splitting components, and changing DB layers deserve a checkpoint.
- **Surface tradeoffs honestly.** Some fixes (introducing a DAL, migrating to runes) have ongoing cost. Say so.

## When to push back on the user

If the user asks for a ground-up rewrite, push back — explain the Strangler Fig pattern and the costs of rewriting (lost encoded behavior, no incremental value, regressions). Only proceed with a rewrite if they confirm with full understanding of the tradeoffs.

If the user wants to fix everything in one session, propose a phased plan instead and ask them to pick the most painful phase.

---

## Self-improvement: report findings upstream

This skill is open source. You — the agent running it on a user's codebase — are *not* its maintainer. Don't edit `SKILL.md` or `references/*` in place; the user running you didn't sign up to ship skill upgrades. Instead, when you notice a way this skill could be sharper, file an issue against the upstream repo.

Upstream: `https://github.com/jangop/vibe-remediator` — use `gh issue create --repo jangop/vibe-remediator` if `gh` is available, otherwise show the user a pre-filled issue body they can paste into the browser.

### When to reflect

- **Session wrap-up** — the user signals they're done, or engagement falls off.
- **End of a remediation phase** — boundaries locked across the codebase, integration tests green, a God file fully decomposed.
- **Surprise moments** — a detection signal misfired, a remediation backfired, a step that's always skipped turned out to be load-bearing somewhere.

You don't need permission to *think* about improvements. Ask the user before *filing* anything — the issue will be public and attributed to their GitHub account.

### What's worth reporting

- **Detection signals that generalized.** A grep, query, or graph traversal that found the same anti-pattern in multiple unrelated modules — not just one suspicious line.
- **Heuristics that made decisions easier.** Rules of thumb that resolved many similar small decisions without thinking ("don't extract a component until it's used twice or exceeds ~50 lines of JSX").
- **Playbook steps that never paid off.** If a recommended action has been prescribed across several sessions and never proved useful, that's worth surfacing.
- **Repeated micro-tasks.** If the agent ends up doing the same little dance over and over across different sessions (rewriting `dict[str, Any]`, swapping `requests` for `httpx`, the same Zod-around-`fetch` wrap), that's a candidate for a bundled script or a tighter playbook example.

### What's *not* worth reporting (anti-overfitting guardrails)

- **Codebase-specific names.** No file paths, function names, package versions, or framework quirks from this one session. Strip them out before generalizing — if you can't strip them out, the lesson isn't general enough to file.
- **One-shot decisions.** Choices driven by a single user's preference or a single codebase's history don't belong in the skill. The next user's situation is different.
- **War stories.** Don't propose adding long worked examples from this session. The skill is a guide, not a portfolio.
- **Hedging caveats.** Don't propose "but sometimes you should do the opposite" unless the opposite case is genuinely common. Caveats erode every rule they attach to.

### Three tests before filing

1. **Does it apply across at least three plausibly different codebases?** If you can only picture this one, it's overfitting.
2. **Does it survive a name swap?** Mentally replace FastAPI with Flask, Pydantic with attrs, Next.js with Remix. Does the principle still hold? If not, the advice is mechanism-specific — it belongs in the relevant `references/*-playbook.md`, not the main SKILL.md.
3. **Are you proposing it because it's *valuable*, or because it's *fresh in memory*?** Recency bias drives skill bloat. If you couldn't recall this example a week from now, neither could a future user.

### Issue format

Surface the proposal to the user first, in plain language: "Here's what I noticed during this session that might generalize: [...]". If they agree it's worth filing, open an issue with this structure:

```
Title: [observation] suggests [change]

## What I noticed
A short, codebase-neutral description of the pattern or gap.

## Why it might generalize
Why this isn't a one-codebase quirk. Reference the three tests if useful.

## Proposed change
- Section of SKILL.md or references/*-playbook.md affected
- Concrete suggested edit (sketch, not final wording)
- What it would replace or remove, if anything (favor replacement over appending)

## Counter-considerations
- When this advice would be wrong
- Whether it competes with anything already in the skill
```

Encourage the user to add their own framing before submitting — they have context the agent doesn't.

### Replace and remove, don't just append

Whatever change you propose, favor *sharpening* over *adding*. The skill stays useful by getting smaller and clearer over time, not by growing. If a proposed addition doesn't subsume or replace existing text, ask yourself whether the existing text is still pulling its weight.
