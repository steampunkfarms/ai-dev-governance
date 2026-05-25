# Bounded Deviation

> The [Sanity Check](sanity-check.md) lets the Executor correct a blind spec. Bounded Deviation is the precise, narrow license that says *when it may do so on its own* — and when it must stop and ask.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Sanity Check](sanity-check.md)

---

## The wall

Once you accept that "the spec is a design, not a law," you've opened a dangerous door. If the Executor can deviate from the plan, what stops it from *improving* things you didn't ask for? Refactoring a file it happened to open? "Fixing" an unrelated bug? Expanding a one-line change into an afternoon?

Unbounded deviation turns a helpful guardrail into an unpredictable collaborator — the exact thing the two-role split was supposed to prevent. The Sanity Check is only safe if the freedom it grants is tightly fenced.

## The rule

The Executor may deviate from a handoff **without pausing for approval** only when **all three gates** pass:

1. **Evidence** — the reason is anchored in a file or query result you could point to and reproduce. Not a hunch, not taste.
2. **Minimal & risk-reducing** — the deviation is the *smallest* change that removes a *concrete* risk. It is not an enhancement.
3. **Scope-stable** — it doesn't materially expand which files or systems the task touches.

If **any** gate fails, the Executor emits a [delta](sanity-check.md) and waits. And **every** deviation — even a fully-authorized one — is logged in the project's `AGENT.md` changelog. The license to deviate is never a license to deviate *silently*.

## How it works

For each step where production disagrees with the spec, the Executor runs the three gates in order. The moment one fails, it stops evaluating and pauses.

- **Gate 1 fails** → "I think this is better" with nothing reproducible behind it. Pause.
- **Gate 2 fails** → the change is an improvement, not a risk reduction. Pause (and log it as a roadmap suggestion, not a deviation).
- **Gate 3 fails** → fixing it would pull in new files, new systems, or new surface area. Pause.

**Default when uncertain: pause.** A paused task that leaves a clear delta costs minutes. A confident deviation that was actually scope creep costs trust — and trust is the only reason you let the agent run unattended at all.

## Worked example

**Allowed.** Continuing the billing-rename case from the [Sanity Check](sanity-check.md): the spec said *mutate the record*; the Executor proposes *add a display alias* instead.

- Gate 1 — a query shows 6 issued invoices bound to the record by name. ✅ file-anchored
- Gate 2 — the alias is the minimal change that avoids re-rendering settled invoices. ✅ risk-reducing
- Gate 3 — it still touches only the billing record and its display layer; no new systems. ✅ scope-stable

All three pass → the Executor proceeds and logs the deviation.

**Not allowed.** While implementing, the Executor notices the invoice component is messy and wants to refactor it.

- Gate 1 — the code smell is real and visible. ✅
- Gate 2 — a refactor is an *enhancement*, not a risk reduction. ❌

Gate 2 fails, so the Executor does **not** touch it. Instead it notes "invoice component could use a refactor" as a roadmap suggestion in the changelog and moves on. The improvement might be worth doing — but as its own handoff, with its own review, not smuggled into this one.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "While I'm here" refactors | Classic scope creep; fails Gate 3 | Log it as a separate roadmap item |
| Deviating on taste | No reproducible evidence; fails Gate 1 | Follow the spec or emit a delta |
| Expanding to fix unrelated bugs | Turns one reviewable change into many | File the bug; keep the task atomic |
| Deviating silently | The operator can't trust unseen reasoning | Always log the deviation, even when authorized |
| Pausing on everything | Defeats the point of an autonomous Executor | Trust the three gates; proceed when all pass |

## Adapt it to your setup

- **Solo:** you're the approver, so "pause" means *stop and leave a clear note* (a checkpoint or a delta in plain sight) rather than guessing. When you're back, you decide.
- **Team:** deltas route to whoever owns the repo. Recurring, always-approved corrections get promoted into the family `AGENT.md` so the Executor stops asking about settled questions.
- **Tuning the gates:** Gate 3 is your trust dial. Early on, keep it tight — any new file means pause. As the Executor proves itself on a codebase, you can widen what counts as "scope-stable."

## Related

- [The Sanity Check](sanity-check.md) — where deviations originate and how a delta is written
- [The Two Roles](../README.md#foundation-3--the-two-role-framework) — why this freedom belongs to the Executor
- Back to the [main playbook](../README.md)
