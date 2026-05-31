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

## Where state goes when the governance file grows

A single-file governance store (one `state.md` for the whole enterprise) works for the first few months. By the time you're carrying a dozen active projects, three families of standards, six months of decisions, and a backlog of deferred items, that file becomes unreadable — and every session pays a re-read tax for context it doesn't need.

The answer is a **state cascade**: one quick-read index file, with the bulk of the state sharded into named files that the session loads on demand.

```
governance/
├── state.md                       ← quick-read index + cascade map
└── state/
    ├── decisions-log.md           ← durable decisions, append-only
    ├── deferred-items.md          ← anything that's been pushed forward
    ├── architecture-notes.md      ← cross-cutting design notes
    ├── active-work.md             ← what's in flight right now
    ├── projects/
    │   ├── <project-slug>.md      ← per-project state, one file each
    │   └── …
    └── completed-YYYY-MM.md       ← monthly archive of finished work
```

**The routing rule for session-end updates:**

| Update | Goes in |
| :---- | :---- |
| One-line current-status summary, pointer to where the detail lives | `state.md` |
| A durable decision (a rule, a precedent, a "we're not doing X") | `state/decisions-log.md` |
| Something pushed forward without a fixed date | `state/deferred-items.md` |
| Per-project status, blockers, next-step | `state/projects/<slug>.md` |
| Work shipped this month | `state/completed-YYYY-MM.md` |
| A cross-cutting design observation that doesn't belong to one project | `state/architecture-notes.md` |

**The routing rule for session-start reads:**

1. Always read `state.md` — the index and cascade map.
2. Read the downstream files in `state/` only when the request touches their subject. A request about project X loads `state/projects/x.md`; a request that touches a precedent loads `state/decisions-log.md`. The cascade map in `state.md` tells you which downstream files to load when.
3. Read alerts and global context files (the [`AGENT.md` cascade](context-cascade.md)) regardless.

The payoff: `state.md` stays under a page; per-session context loads stay under a few KB; nothing is lost because the bulk lives in the named files, addressable by topic. The pattern scales from one repo to thirty without changing shape — only the per-project file count grows.

**When to adopt the cascade:** when `state.md` reaches a length that makes you scroll to find anything, or when sessions repeatedly load context they don't need. Until then, a single file is enough.

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
