# The Sanity Check

> A blind spec meets live production *before* it runs. The Executor verifies every assumption the Strategist made against the real database and filesystem, and stops if reality disagrees.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](../README.md#foundation-3--the-two-role-framework)

---

## The wall

The Strategist writes a clean, confident spec. It reads beautifully. It is also *wrong*, because the Strategist has never seen your production data — it's reasoning from the schema and from whatever you told it, not from what's actually in the tables right now.

Here's the one that earned this protocol. A spec said, reasonably: *"Rename this billing record to match the new client name."* Sensible on paper. But that record had **already issued invoices**, and the invoice line items referenced it by name. The rename went through. The next morning, paid invoices were displaying under the wrong client. The fix was a manual data-repair job and an awkward email.

Nothing was wrong with the *plan*. The plan was wrong about the *world*. No amount of better prompting fixes this, because the Strategist structurally cannot see the world — so the check has to live with the role that can.

## The rule

**Every handoff is a design, not a law.** Before executing any spec, the Executor reconciles it against live production. If reality contradicts the spec, the Executor stops and reports a *delta* rather than following the spec off a cliff.

## How it works

The Executor runs a **pre-edit pass** — three quick checks, in order, before touching anything:

1. **Data-state check** — query the records the spec assumes. Do they exist? In the state the spec expects? *(Does this billing record exist, and has it issued anything?)*
2. **Conflict check** — naming collisions, unique constraints, foreign-key relationships, live references. *(Will this rename break a name-based reference somewhere?)*
3. **Reversibility check** — flag any step that touches something already **sent, paid, or deployed.** These are the steps that hurt when they're wrong.

If all three pass, the Executor proceeds normally. If any fails, it emits a **delta** and waits:

```text
## Sanity Delta — Handoff H-042, Step 3

SPEC SAYS:     Rename billing record "Acme" → "Acme Holdings LLC"
PRODUCTION:    Record id=4471 has 6 issued invoices; 2 are paid.
               Invoice line items reference the record by display name.
RISK IF RUN:   Paid invoices re-render under the new name → client-facing
               discrepancy on already-settled documents.
CORRECTION:    Add a display-name alias instead of mutating the record;
               leave issued invoices bound to the original name.
NEW CRITERIA:  New invoices use "Acme Holdings LLC"; the 6 issued
               invoices are unchanged. Verified by re-rendering invoice #5.

→ Awaiting operator approval before proceeding.
```

The **operator** approves, adjusts, or rejects. Only then does the work run. The delta — and the decision — gets logged in the repo's `AGENT.md` changelog, so the reasoning survives the session.

## Worked example

**Before (spec as written):** mutate record `4471` in place.
**Pre-edit pass:** data-state check finds 6 issued invoices on `4471`; reversibility check flags 2 as paid.
**After (delta applied):** add an alias, leave issued invoices bound to the original name, new invoices pick up the new name.
**Outcome:** the rename the Strategist *intended* happens for everything going forward, and nothing already settled silently changes underneath the client.

The Strategist's goal was met. The Strategist's *method* was corrected by the only role that could see why it was unsafe.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Treating the spec as authoritative | It was written blind; production is the source of truth | Reconcile against live data first |
| Silent deviation | The operator can't trust results they can't see reasoned | Always emit a delta; never "fix it quietly" |
| Deviating to "improve" the spec | Scope creep dressed up as diligence | Deviate only to *reduce risk* (see [Bounded Deviation](bounded-deviation.md)) |
| Running the check but skipping the log | The next session repeats the mistake | Record the delta in the changelog |

## Adapt it to your setup

- **Solo, one repo:** you're the operator. The check is just a habit — before your Executor runs anything destructive, have it print "here's what the spec assumes vs. what I actually see" and eyeball it.
- **No database:** the same three checks apply to files, config, and deploy state — does the file exist, will this collide, is it already shipped?
- **Higher trust over time:** as patterns recur, promote the safe corrections into your family `AGENT.md` so the Executor applies them automatically (within the [Bounded Deviation](bounded-deviation.md) limits) instead of pausing every time.

## Related

- [Bounded Deviation](bounded-deviation.md) — the exact limits on when the Executor may deviate without asking
- [The Two Roles](../README.md#foundation-3--the-two-role-framework) — why the check belongs to the Executor, not the Strategist
- Back to the [main playbook](../README.md)
