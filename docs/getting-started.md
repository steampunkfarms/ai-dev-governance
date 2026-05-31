# Getting Started

> The README is the manifesto. The architecture reference is the working manual. This file is the **first thirty minutes** — the smallest possible adoption that proves the model in your own repo before you commit to anything bigger.

**Part of:** [AI Dev Governance](../README.md) · **Read this when:** you've skimmed the README, agree the failure modes are real, and want a concrete on-ramp instead of a wall of theory.

---

## The wall

Most teams that find this repo bounce off it the same way: they read the manifesto, nod at the six lessons, scroll the deep-dive table, and quietly close the tab. The model looks right, but the *adoption path* is invisible. Where do you actually start on a Monday morning when you've got one repo, one chat assistant, one terminal agent, and a Tuesday deadline?

If you copy the whole playbook into a one-repo solo project, you'll over-engineer it in an afternoon — four-level cascades, governance sync, a Delta Log with two entries. You'll feel ridiculous within a week and abandon the model.

The right move is the opposite: **adopt the smallest piece that survives a real failure**, prove it on one task, then add the next piece only when a new failure mode actually bites. The first thirty minutes below get you to the minimum viable governance for any setup, in any pairing of tools.

## The rule

The smallest useful adoption is **two roles + a checkpoint**. Everything else in the playbook is layered on top of that core and earns its place by surviving a specific failure. If you adopt those two things on your next task, you have already eliminated the majority of the pain the rest of the playbook is reacting to.

## How it works

### Step 1 — Pick your two tools (5 minutes)

You almost certainly already have both. The roles matter more than the brands.

- **Strategist** — the chat-based AI you use for thinking out loud, research, and writing specs. Lives *outside* your repo. Should not have commit access even if it could.
- **Executor** — the agent that lives *inside* your repo: runs the build, edits files, commits. Usually a terminal agent or an IDE-integrated one.

Examples — all equally valid pairings:

| If your Strategist is… | A natural Executor is… |
| :---- | :---- |
| Perplexity | Claude Code |
| ChatGPT | Codex |
| Claude (web/app) | Cursor |
| Gemini (web) | Gemini CLI |
| Grok | Grok Build |

You can mix vendors freely. The shop this playbook came from runs two Strategists in parallel and pairs them with a single Executor — the model genuinely doesn't care.

**Write down which tool plays which role.** That single sentence is your governance for the day.

### Step 2 — Drop one `AGENT.md` at the repo root (5 minutes)

Create `AGENT.md` (or `CLAUDE.md`, `AGENTS.md`, `.cursorrules` — whatever your Executor loads at startup) in your repo root. Five lines is enough for day one:

```markdown
# Project rules

- Strategist: <which tool>, no commits, never edits source.
- Executor: <which tool>, owns commits, runs type-check before "done".
- Commit messages: <your convention, e.g. conventional commits>.
- Type-check command: <e.g. `npx tsc --noEmit`>.
- Branch model: <e.g. trunk-based, push directly to main>.
```

That file is your context cascade for now. You'll grow it as the rules earn their place. Most one-repo setups never need more than one of these.

### Step 3 — Write your first checkpoint (5 minutes)

Pick the next task you're about to give your Executor — anything non-trivial, even a single feature. Before the Executor touches code, *the Strategist* writes a file at `docs/checkpoints/<YYYY-MM-DD>-<slug>.md`:

```markdown
# Checkpoint: <Task Name>
**Date:** YYYY-MM-DD
**Status:** PLANNED

## Plan
1. [ ] Step one
2. [ ] Step two
3. [ ] Step three

## Context for next session
- The active task is <one sentence>.
- The first file to open is <path>.
- Non-obvious decisions: <anything not visible in the code>.

## Files expected to change
- path/to/file — what changes
```

That file is the whole handoff. Hand it to your Executor verbatim. When the Executor finishes a step, it ticks the box and adds a one-line note. When the session ends, it updates the *Status* line and the *Context for next session* block.

This is the single highest-leverage habit in the playbook. It is also the one most people skip on small tasks — and small tasks are exactly where session timeouts hurt most, because all the context lived in chat scrollback.

### Step 4 — Run the task end to end (15 minutes)

The discipline:

1. **Strategist** writes the handoff (the checkpoint above), grounded in any files it can read in your repo. It does not edit anything.
2. **Executor** opens the checkpoint, opens the files, makes the changes, runs the type-checker, commits, pushes.
3. **Executor** updates the checkpoint with what shipped and any deviations from the plan.
4. **You** glance at the diff and the checkpoint. Done.

That's the loop. On the first run it will feel like overhead. By the third run, the checkpoint *is* how you think about the task — and the first time a session times out mid-task, the file on disk will save you the entire context.

## When to add the next piece

Adopt nothing else until you feel the matching failure. The triggers:

| If you start hitting… | Adopt next | Deep dive |
| :---- | :---- | :---- |
| "The plan looked right but production didn't match" | **The Sanity Check** — Executor reality-checks against live data before executing | [sanity-check.md](sanity-check.md) |
| "The Executor wrote to the wrong filesystem and lost work" | **File handling** — audit your tool surface, verify by reading back | [file-handling.md](file-handling.md) |
| "Rules conflict between this repo and that repo" | **The Context Cascade** — declared precedence across scopes | [context-cascade.md](context-cascade.md) |
| "The Executor strayed from the spec and we don't know why" | **Bounded Deviation** — the three-gate rule for when the Executor may stray | [bounded-deviation.md](bounded-deviation.md) |
| "Shipping happened but nobody can tell from the git log what changed" | **The QA Gate** — the fixed checklist that defines 'done' | [qa-gate.md](qa-gate.md) |
| "Roadmap A says one thing, roadmap B says another" | **The Roadmap System** — two genres, one index, drift caught on audit | [roadmap-system.md](roadmap-system.md) |
| "Strategist and Executor keep over-writing each other's governance files" | **Governance Sync** — fast-forward-only with a tie-breaker rule | [governance-sync.md](governance-sync.md) |
| "Same fix keeps appearing in three different repos" | **The Delta Log** — count deltas until a pattern becomes a standard | [delta-log.md](delta-log.md) |

The pattern: a doc earns its place by preventing a failure you can name. If you can't name the failure, you don't need the doc yet.

## Worked example

A solo developer with one Next.js app and Claude + Claude Code adopts the model on a Monday:

- **Roles:** Claude (web) is Strategist. Claude Code is Executor.
- **`AGENT.md`:** five lines at the repo root, naming the two tools, the type-check command, and the convention that the Executor must run `npx tsc --noEmit` before committing.
- **First task:** add a settings page. Claude writes a checkpoint at `docs/checkpoints/2026-05-30-settings-page.md` listing four steps and the two files expected to change.
- **Execution:** Claude Code opens the checkpoint, makes the changes, ticks each box as it goes, commits, pushes. Total wall time: 40 minutes, of which 5 minutes was reading and updating the checkpoint.

Two weeks in, the developer hits the wall where the Executor edits the wrong env file and a deploy breaks. *That's* the trigger to adopt the Sanity Check — and the failure mode is concrete enough that the protocol feels obvious instead of bureaucratic. Two months in, a second repo joins the setup, and the context cascade earns its first override. By the time the developer is at four repos, the full architecture reference reads like a description of what they were already doing.

That's the intended adoption curve: **discipline first, structure later, and only the structure that real failures justify.**

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Adopt the full playbook on day one | Most of it is solving failures you haven't hit; you'll abandon the whole thing | Adopt two roles + a checkpoint; add the rest on real triggers |
| Skip the `AGENT.md` because "it's just me" | Future-you isn't current-you; neither is the Executor on Tuesday | Write the five lines. They take two minutes and prevent the "wait, which command?" loop |
| Treat the Strategist/Executor split as different *products* | The split is about *discipline*, not branding; one tool can play both with care | Run a planning pass, *then* an execution pass, even in one tool |
| Skip checkpoints on "small" tasks | Small tasks are exactly where lost context hurts most | Always checkpoint. The file is your only memory |
| Read every deep dive before starting | Analysis paralysis; you'll never start | Read this page, ship one task, then read the next deep dive when the trigger fires |

## Adapt it to your setup

- **Solo, one repo, one tool:** the two-role split becomes a planning pass and an execution pass in the same chat. The checkpoint is still a real file on disk. That's enough.
- **Solo, one repo, two tools:** exactly the example above. The handoff between Strategist and Executor is the checkpoint file.
- **Team of two-to-five, one repo:** the Strategist may be a human + AI pair; the Executor may be a human running an agent. The checkpoint becomes your async interface.
- **Multiple repos:** the moment you have two repos copy-pasting a rule, that rule moves up the cascade. That's the trigger to read the [Context Cascade](context-cascade.md) deep dive — not before.

## Related

- [The Two Roles](two-roles.md) — the role definitions you just adopted
- [Checkpoints & Handoffs](checkpoints-handoffs.md) — the full templates, when the five-line version stops being enough
- [The Architecture reference](architecture.md) — the working manual, for when the model has earned your trust
- Back to the [main playbook](../README.md)
