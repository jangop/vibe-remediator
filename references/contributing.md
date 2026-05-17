# Contributing to vibe-remediator

This skill is meant to evolve based on patterns observed across many real codebases. The contribution path is:

1. The agent running the skill notices a potential improvement at session wrap-up, end of a remediation phase, or after a surprise moment (a detection signal misfired, a remediation backfired, a step that's always skipped turned out to matter).
2. Surface the proposal to the user in plain language.
3. Apply the three anti-overfitting tests below.
4. If it survives all three, file an issue with the template below.

PRs that don't reference an issue are welcome, but the issue-first path catches overfitting earlier and keeps the skill from drifting toward whichever codebase was in front of it last.

## Three anti-overfitting tests

Before filing, check the proposed change against all three:

1. **Does it apply across at least three plausibly different codebases?** If you can only picture this one codebase benefiting, it's overfitting.
2. **Does it survive a name swap?** Mentally replace FastAPI with Flask, Pydantic with attrs, Next.js with Remix. Does the principle still hold? If it only works with the named tools, it belongs in `references/*-playbook.md`, not the main SKILL.md.
3. **Are you proposing it because it's *valuable*, or because it's *fresh in memory*?** Recency bias drives skill bloat. If you couldn't recall this example a week from now, neither could a future user.

## What's worth reporting

- **Detection signals that generalized.** A grep, query, or graph traversal that found the same anti-pattern in multiple unrelated modules.
- **Heuristics that made decisions easier.** Rules of thumb that resolved many similar small decisions ("don't extract a component until it's used twice or exceeds ~50 lines of JSX").
- **Playbook steps that never paid off.** If a recommended action has been prescribed across several sessions and never proved useful.
- **Repeated micro-tasks** that could become a bundled script or a sharper playbook example.

## What's *not* worth reporting

- **Codebase-specific names.** No file paths, function names, package versions, or framework quirks from one session. If you can't strip them out, the lesson isn't general enough.
- **One-shot decisions** driven by a single user's preference or a single codebase's history.
- **War stories.** The skill is a guide, not a portfolio of past wins.
- **Hedging caveats.** "But sometimes you should do the opposite" erodes every rule it attaches to.

## Replace, don't append

The skill stays useful by getting smaller and clearer, not by growing. If a proposed addition doesn't subsume or replace existing text, ask whether the existing text is still pulling its weight.

## Issue template

```
Title: [observation] suggests [change]

## What I noticed
A short, codebase-neutral description of the pattern or gap.

## Why it might generalize
Why this isn't a one-codebase quirk. Reference the three tests if useful.

## Proposed change
- Section of SKILL.md or references/*-playbook.md affected
- Concrete suggested edit (sketch, not final wording)
- What it would replace or remove, if anything

## Counter-considerations
- When this advice would be wrong
- Whether it competes with anything already in the skill
```
