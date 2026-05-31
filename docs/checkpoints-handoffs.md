# Checkpoints & Handoffs

> Two gaps swallow work: **time** (a session ends mid-task) and **roles** (the Strategist's intent must reach the Executor intact). Two written artifacts bridge them. Files are the only memory.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md)

---

## The wall

AI sessions time out, and the agent's working memory evaporates with them. The next session starts cold and either redoes finished work or — worse — half-redoes it and leaves the repo in a state no one intended (a migration applied twice, a file half-refactored).

The same gap appears across roles, in space rather than time: the Strategist's careful reasoning decays into *"uh, do the thing we talked about"* by the time the Executor acts on it.

## The rule

- A **checkpoint** carries state across **time** — session to session.
- A **handoff** carries state across **roles** — Strategist to Executor.

Both are files on disk. Neither relies on an agent "remembering."

## How it works

**Checkpoints.** Write one at the start of any multi-step task (the plan, all steps unchecked), after each major step, when a timeout feels close, and at session end. Store them at `<repo>/docs/checkpoints/<DATE>-<slug>.md`. The session-start protocol: read the latest checkpoint → read the [`AGENT.md` cascade](context-cascade.md) → read the execution roadmap → resume, skipping completed work → write a fresh checkpoint noting where you picked up.

```
# Checkpoint: <Task Name>
**Date:** YYYY-MM-DD
**Status:** IN PROGRESS | BLOCKED | COMPLETE

## Plan
1. [done] Step one — notes
2. [in progress] Step two — got through X
3. [ ] Step three

## Context for next session
- What was the active task when this ended?
- Which file was mid-edit?
- What decisions aren't obvious from the code?
- What should the next session do FIRST?

## Files modified this session
- path/to/file — what changed and why
```

**Handoffs.** The Strategist writes them to the governance repo. A complete handoff carries: file paths, function signatures, env-var names, an ordered plan, explicit acceptance criteria, and an **Operator Action Block** listing manual steps (secrets, migrations) the agent must *not* do. The Executor implements it, then logs the result to the project changelog.

```
# Handoff: <ID> — <Title>
**Tier:** <0–3>   **Repo:** <name>

## Objective
<what and why, in two sentences>

## Steps
1. <path> — <change>
...

## Acceptance criteria
- [ ] <PASS/FAIL condition>

## Operator Action Block (human-only)
1. <secret / migration / anything the agent must not run>
```

## Worked example

A migration task times out after step 3 of 6. Without a checkpoint, the next session re-runs steps 1–3 and double-applies the migration. With this checkpoint, it doesn't:

```
## Plan
1. [done] Add tenant_id column
2. [done] Backfill existing rows
3. [done] Apply migration   ← DB migration ALREADY applied; do NOT re-run
4. [ ] Update the ORM model
5. [ ] Wire the tenant middleware
6. [ ] QA gate + push

## Context for next session
- Migration is live in prod. Start at step 4.
```

The next session reads it, skips to step 4, and finishes cleanly. Nothing is lost, and nothing is done twice.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "I'll remember where I was" | The session won't | Checkpoint every multi-step task |
| Saving all files at the end | A timeout means total loss | Write each file as it's completed |
| Vague checkpoints ("worked on stuff") | Useless to the next session | Record the next action + non-obvious state |
| Handoff with no acceptance criteria | "Done" is undefined | List PASS/FAIL criteria |
| No Operator Action Block | Manual steps get lost, or the agent oversteps | List every human-only step explicitly |

## Adapt it to your setup

- **Solo:** checkpoints are notes to your future self; handoffs become your own session-to-session plan even with a single tool.
- **Team:** the handoff is your async interface — the Strategist and Executor may be different people on different days.

## Related

- [File Handling](file-handling.md) — the mechanics that make incremental checkpointing actually work
- [The Two Roles](two-roles.md) — what the handoff connects
- [The Sanity Check](sanity-check.md) — the Executor's first move on receiving a handoff
- [Governance Sync](governance-sync.md) — how the handoff file gets safely shared between roles
- [The file-handling protocol — in context](architecture.md#foundation-4--the-file-handling-protocol)
- Back to the [main playbook](../README.md)
