# vibe-remediator

> A Claude Code skill for taming AI-generated codebases.
>
> <sub>(Yes, **VIB**e-**R**emedi-**ATOR**. The acronym is intentional. It's the only joke in the repo.)</sub>

When an LLM blasts out a solution, it optimizes for the next token, not the next year of maintenance. `vibe-remediator` is a [Claude](https://www.anthropic.com/claude-code) [skill](https://docs.claude.com/en/docs/claude-code/skills) that gives Claude a structured workflow for auditing and incrementally remediating the systemic flaws that result — without falling into the "ground-up rewrite" trap.

## What it does

Triggers whenever you describe a codebase as "vibe coded", "AI slop", "a mess", "prototype that needs to scale", or ask Claude to "make this production ready" / "audit my codebase".

Claude will then:

1. **Triage** — walk a checklist of anti-patterns common in LLM-generated codebases (God components, type illusion, N+1 queries, silent failures, async/sync hazards, prompt drift, broken row-level authorization, webhook gaps, secrets in chat history, etc.) and produce a prioritized audit with concrete file paths.
2. **Plan** — group findings into phases (boundaries → safety net → structural → polish) and pause for your approval before changing code.
3. **Fix incrementally** — apply the Strangler Fig pattern: lock down edges with strict parsing, write black-box integration tests as a safety net, then refactor structure.

The skill is **guide-informed, pragmatic**: if the codebase already uses Joi instead of Zod, attrs instead of Pydantic, or Drizzle instead of SQLAlchemy, it adapts to what's there rather than dogmatically prescribing one stack.

Language-specific deep dives live in `references/` and are loaded on demand based on the stack at hand:

- `references/python-playbook.md` — Ruff, Pydantic, mypy/pyright strict, FastAPI async hazards, Repository pattern.
- `references/javascript-playbook.md` — Zod, server-side data fetching in Next.js / SvelteKit / Nuxt, Svelte 5 runes, React state colocation, God component extraction.

## Installation

### Option 1 — clone into your global skills directory

```bash
git clone https://github.com/jangop/vibe-remediator.git ~/.claude/skills/vibe-remediator
```

### Option 2 — clone anywhere and symlink

```bash
git clone https://github.com/jangop/vibe-remediator.git
ln -s "$(pwd)/vibe-remediator" ~/.claude/skills/vibe-remediator
```

Then in Claude Code, run `/reload-plugins` (or restart the session). The skill should appear in your available skills list as `vibe-remediator`.

## Usage

Just talk to Claude normally. Trigger phrases like:

- "This codebase was vibe coded — can you clean it up?"
- "Audit my FastAPI service for anti-patterns."
- "I inherited this Next.js app and it's a mess. Where do I start?"
- "Make this prototype production-ready."
- "There are N+1 queries somewhere in here, find them."

…should all activate the skill. You don't need to invoke it explicitly.

## Philosophy

This isn't a linter. It doesn't try to autofix everything. The remediation workflow is designed around five core principles:

1. **Never rewrite from scratch.** Strangler Fig — build well-architected pieces alongside the mess and migrate piece by piece.
2. **Lock down the edges first.** Parse, don't validate. Strict schemas at every system boundary create a trust region the interior can refactor safely.
3. **Tests are the safety net, not the goal.** Write high-level black-box integration tests against current behavior — then refactor.
4. **Be pragmatic, not dogmatic.** Adapt to existing tooling. Consistency at boundaries matters more than which library.
5. **One pass at a time.** Don't try to fix every anti-pattern simultaneously. Highest-leverage first.

## Contributing

Issues and pull requests welcome. If you've encountered a vibe-coding anti-pattern this skill doesn't catch, open an issue with a representative example — most useful additions come from real codebases.

## License

MIT.
