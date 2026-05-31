# Checkpoint: Gap-fill pass — five new deep dives + cross-links
**Date:** 2026-05-30
**Status:** COMPLETE
**Session:** Perplexity Computer (CChat), as Contributor

## Plan
1. [done] Write `docs/getting-started.md` — 30-minute on-ramp; trigger table for next-doc adoption
2. [done] Write `docs/autonomous-loops.md` — kill switches, heartbeats, dry-run, named owner, leak protocol
3. [done] Write `docs/architecture-graphs.md` — generated structure maps as a first-class Strategist input
4. [done] Write `docs/secrets-and-rotation.md` — names-not-values, operator monopoly on paste, rotation + leak playbooks
5. [done] Write `docs/cost-management.md` — budgets, monthly review, circuit breakers, overrun-as-incident
6. [done] Write `docs/dependency-management.md` — lockfile-as-truth, per-tier cadence, monthly audit, major-as-Tier-3
7. [done] Add escalation-routing matrix to `architecture.md` (Foundation 3) — officer closest to the battlefield
8. [done] Add state-cascade routing pattern to `checkpoints-handoffs.md` — for governance files that outgrow one page
9. [done] Expand `architecture.md` Layered Protocols section with a fifth subsection pointing into the five new deep dives
10. [done] Add the five new dives to the README deep-dive table; add a "New here?" section pointing to getting-started.md
11. [done] Verify all links; commit and push

## Sanity Delta (against the prior session's gap analysis)

The prior session identified 11 gaps in four buckets. This session shipped:

| Gap (from prior analysis) | Bucket | Shipped as |
| :---- | :---- | :---- |
| Missing "Getting started" path | D (structural) | `docs/getting-started.md` + README "New here?" + getting-started entry at top of deep-dive table |
| Knowledge graphs as Strategist input | A (no public equivalent) | `docs/architecture-graphs.md` + new Layered Protocols subsection in `architecture.md` |
| Kill switches + observability for autonomous loops | A | `docs/autonomous-loops.md` |
| Secret/credential rotation discipline | A | `docs/secrets-and-rotation.md` |
| Cost discipline for token + infra spend | A | `docs/cost-management.md` |
| Dependency upgrade cadence + lockfile discipline | A | `docs/dependency-management.md` |
| Escalation routing ("officer closest to the battlefield") | B (worth deepening) | New `#### Escalation routing — who handles what` table in `architecture.md` Foundation 3 |
| State cascade for growing governance | B | New `## Where state goes when the governance file grows` section in `checkpoints-handoffs.md` |

Two gaps from the prior analysis were intentionally deferred:

- **A separate "alerts/" surface convention.** Touched obliquely by the heartbeat discipline in Autonomous Loops; promoting it to its own doc was judged premature for a public repo. Revisit if the autonomous-loops doc generates questions about how alerts surface across many repos.
- **A "knowledge management" doc covering pillar/strategy memory.** This is BTS-internal-shaped (Stazia's domain expertise routing, pillar strategy across the BFOS family) and resists de-identification cleanly. Better left to the [Delta Log](../delta-log.md) and the changelogs as patterns, rather than a standalone doc that would feel hollow without the BTS specifics.

One item from the prior analysis was deliberately *not* shipped:

- **Tier-0-style "system overview" hand-tour of the orchestrator.** Public repo doesn't carry an orchestrator — the orchestrator is a BTS-specific shared resource. The existing `Foundation 1 — Repo schema & routing` already establishes the *pattern* of a shared orchestrator; documenting one team's specific shape would push the model from "vendor-agnostic" toward "describes BTS's setup."

## Deviations from the plan

None. The plan was executed as written. One mid-session decision: the new Layered Protocols subsection in `architecture.md` is positioned as "supporting disciplines" rather than as a sixth foundation, to preserve the "four foundations + four layered protocols" spine that the README and the architecture reference both build on. Adding the five new disciplines as a *fifth* layered-protocol subsection keeps the spine intact while making the new dives visible at the architecture-reference level.

## Files modified this session

| Path | Change |
| :---- | :---- |
| `docs/getting-started.md` | NEW. 30-minute on-ramp; trigger table for adopting subsequent deep dives. |
| `docs/autonomous-loops.md` | NEW. Five rules: kill switch / heartbeat / owner / tier / dry-run. Misfire recovery protocol. |
| `docs/architecture-graphs.md` | NEW. Five rules: contents / generated-not-written / when-Strategist-reads / cross-repo / staleness-disqualifies. |
| `docs/secrets-and-rotation.md` | NEW. Five rules: names-not-values / operator-monopoly / one-source-of-truth / rotation-playbook / leak-protocol. |
| `docs/cost-management.md` | NEW. Five rules: owners-and-budgets / explicit-decisions / monthly-review / circuit-breakers / overrun-as-incident. |
| `docs/dependency-management.md` | NEW. Five rules: lockfile-is-truth / no-phantom-imports / declared-cadence / monthly-audit / majors-are-Tier-3. |
| `docs/architecture.md` | Added `#### Escalation routing — who handles what` table in Foundation 3 (between Tier 0 backfill and "What the Strategist must never do"). Added new Layered Protocols subsection pointing into the five new dives. Updated Related footer with all new links. |
| `docs/checkpoints-handoffs.md` | Added `## Where state goes when the governance file grows` section with state-cascade routing tables. |
| `README.md` | Added pointer to `getting-started.md` in the manifesto block. Added `getting-started.md` row at top of deep-dive table; appended five new dives. Added new "🔗 New here?" section between scale-down and Contributing. Updated `checkpoints-handoffs.md` table row to mention the state cascade. |

## Voice discipline applied (per Operator feedback)

The Operator preferred this contributor's voice over the alternative Strategist's on prior commits. Voice notes for this pass:

- Every new dive uses the canonical six-section shape (Wall / Rule / How it works / Worked example / Anti-patterns / Adapt-it / Related), matching `file-handling.md` as the reference exemplar.
- Each "rule" section is numbered, with a short prose body — no bullet-list-only sections; bullets accompany prose, not replace it.
- Five-second tests are introduced where they apply (cost decisions, secret pastes, lockfile-changing installs, Strategist quick-fix temptations) to give the reader a memorable in-flight check.
- Vendor-agnostic discipline preserved: examples cycle across Perplexity, ChatGPT, Claude, Gemini, Grok (Strategist) and Codex, Claude Code, Cursor, Gemini CLI, Grok Build (Executor). No vendor leads twice consecutively in any list.
- "Scar tissue" tone applied where appropriate — e.g. the autonomous-loops worked example walks the OAuth-token-expiration failure backwards; the cost-management worked example walks the $480/month surprise.
- One self-reference: the dependency-management doc points to commit `1e7d027` of this very repo as a real-world phantom-import example (the package referenced in a Strategist spec was not in the lockfile). This makes the doc honest about being shaped by recent live failures.

## Cross-link audit

All new docs link to at least three existing docs and are linked from at least two existing docs:

- `getting-started.md` ↔ README, two-roles, checkpoints-handoffs, architecture
- `autonomous-loops.md` ↔ README, architecture (foundation 1 + new layered subsection), qa-gate, file-handling, sanity-check, secrets-and-rotation, architecture-graphs
- `architecture-graphs.md` ↔ README, architecture (new layered subsection), two-roles, context-cascade, qa-gate, autonomous-loops
- `secrets-and-rotation.md` ↔ README, architecture (new layered subsection), two-roles, file-handling, qa-gate, autonomous-loops
- `cost-management.md` ↔ README, architecture (new layered subsection), autonomous-loops, two-roles, architecture-graphs, qa-gate
- `dependency-management.md` ↔ README, architecture (new layered subsection), qa-gate, architecture-graphs, secrets-and-rotation, cost-management

## Operator Action Block

None. This was a docs-only pass with no env vars, no migrations, no manual smoke tests required. The Operator's next action is at their discretion:

1. (Optional) Add `getting-started`, `autonomous-loops`, `architecture-graphs`, `secrets-rotation`, `dependency-management`, `cost-management` to the GitHub repo's Topics list — the prior set was 19/20; any of these could replace the least-relevant existing topic if the Operator wants the new dives discoverable by topic search.
2. (Optional) Pin `getting-started.md` as the first issue's reference or in the GitHub repo's About → Description, since it is now the recommended on-ramp.
3. No CC handoff required; this session's work was self-contained.

## Next-session context

- The reinstated standing rule (no direct edits to project repos by the Strategist) automatically resumes after this session. The two-task suspension that covered this work is exhausted.
- The public repo now has 16 dive documents (the original 10, plus `_deep-dive-template.md`, plus 5 new). Future additions should continue using the deep-dive template and the six-section voice shape.
- Two gaps were deferred (alerts surface, knowledge-management doc) and noted above — they are *not* lost, just judged premature for a public-repo de-identified shape. Revisit on operator request or if an issue from the public surfaces them.
