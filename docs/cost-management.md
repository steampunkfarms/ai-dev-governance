# Cost Management

> AI coding is a metered habit. Token spend, infrastructure spend, and tool spend each follow the same shape — a small number of high-cost activities account for most of the bill, and the bill is invisible until it isn't. **The discipline isn't to spend less; it's to spend deliberately.** This file is about making cost a first-class signal in the governance loop.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md) · **See also:** [Autonomous Loops](autonomous-loops.md), [The QA Gate](qa-gate.md)

---

## The wall

The cost story breaks the same way every time:

- **Month one:** the AI coding habit feels free. The model you're using has a generous monthly cap and you haven't hit it. You're not measuring.
- **Month three:** you've added two more repos, two more agents, a few crons that call an LLM, and a cloud hosting tier you upgraded "for the deploys." You're vaguely aware spend is up but you couldn't put a number on which line item drove it.
- **Month six:** a single bill arrives that's three to ten times what you expected. The cause is almost always one of: an autonomous loop calling an expensive model in a tight cycle, a runaway agent that opened a long context window on every request, a hosting tier auto-upgraded by traffic from a single misbehaving cron, or a single Strategist session that used 200K tokens because no one was looking.
- **Month seven:** you cut hard, lose some capability you actually wanted, and start over with a budget you'll forget about by month nine.

The root failure is the same in each case: **cost is invisible at the moment of decision.** When the Strategist picks a model, when the Executor opens a long context, when an Operator wires a cron to run hourly — the dollar implications aren't visible in the loop. The discipline below makes them visible without making them painful.

## The rule

1. **Every recurring spend has a named owner and a monthly budget.** No anonymous bills.
2. **Cost decisions are explicit, not emergent.** Pick the model for the task; pick the hosting tier for the load; pick the cron cadence for the value. Don't default upward.
3. **Spend is reviewed on a fixed cadence** — monthly at minimum — against the budgets, and the review produces a one-line decision: keep, cut, or escalate.
4. **The expensive paths have circuit breakers.** Autonomous loops that call paid APIs have spend caps. Hosting platforms that bill by usage have monthly limits or alerts. No path can spend without bound.
5. **Cost overruns trigger the same backfill discipline as a Tier 0 hotfix.** An incident, a checkpoint, a one-line note for the Strategist on whether the spend pattern should become a new rule.

## How it works

### Rule 1 — Named owners and budgets

Every paid line in the stack — every AI subscription, every hosting account, every metered API — belongs to a named owner: a person, a repo, or a family. The owner is responsible for the spend on that line. A monthly budget is set against the line, in dollars, and recorded in one place — a `budgets.md` file in the governance repo is enough.

The list usually fits on one page:

| Line | Owner | Monthly budget | Trend | Notes |
| :---- | :---- | :---- | :---- | :---- |
| Strategist subscription (per AI) | <name> | $20 | flat | One per AI; expect 2-3 |
| Executor subscription (per AI) | <name> | $200 | rising | Usage-priced beyond cap |
| Hosting (per provider) | <family / repo> | $50 | flat | Per project, where possible |
| Metered APIs (Stripe, Twilio, email) | <repo> | $variable | spiky | Spike alerts at 2× monthly avg |
| LLM API direct calls (per repo) | <repo> | $X | depends | The most volatile line |

What this table does, that nothing else does: it makes the bill *legible*. A glance tells you whether spend is concentrated on Strategists (lots of thinking), Executors (lots of building), hosting (lots of traffic), or APIs (lots of automation). Each of those signals routes to a different intervention.

### Rule 2 — Explicit decisions, not defaults

The biggest cost gains come from refusing the default. Three patterns to watch:

- **Model selection.** The Strategist defaults to "the most capable model available." For most tasks, that's overspend. Pick the model deliberately by tier: the cheap fast model for routine generation, the mid model for normal design work, the top model only when reasoning depth actually matters. The exact picks change quarterly as vendors release new tiers; the *discipline* of picking explicitly does not.
- **Context size.** A long context window is a budget item. The Strategist that opens 40 files before designing pays for 40 files' worth of tokens on every reasoning step. The discipline: load the [architecture graph](architecture-graphs.md), the relevant `AGENT.md`, and the *specific* files the design touches — not everything that might be relevant.
- **Loop cadence.** A cron that runs every 15 minutes is 96 invocations a day. A cron that runs hourly is 24. Most loops want to run "as often as makes sense," which silently becomes "as often as the cheapest tier allows." Pick a cadence by the value of each run, not by the available bound.

The five-second test for any new line of spend: *if I had to defend this expense in a sentence to the operator, what would I say?* If the sentence is "I didn't think about it," reroute.

### Rule 3 — Monthly review

Once a month, the operator opens the budgets table and the most recent invoice from every line. The review is short — 20 minutes is plenty — and produces one of three outcomes per line:

- **Keep.** Spend matches budget within tolerance; continue.
- **Cut.** Spend is rising without value; identify the driver, set a circuit breaker, or move to a cheaper tier.
- **Escalate.** Spend is rising *and* tracks rising value; raise the budget intentionally and note the rationale in the changelog.

The output of the review is a one-line entry per line in a `cost-log.md` in the governance repo: date, line, status, action. A six-month-old `cost-log.md` is the single most valuable artifact when planning a stack change — it tells you, in seconds, where the bill has been concentrated and which interventions actually moved the curve.

Review cadence tightens when the bill is volatile (weekly during a launch month, or when a new loop is in dry-run). It loosens to quarterly when spend has been flat for two consecutive months. The discipline is to *have* a cadence, not to enforce a specific one.

### Rule 4 — Circuit breakers

Every path that can spend without bound needs a cap. The patterns:

- **LLM API calls from code.** A per-day or per-month spend cap, configured on the vendor's dashboard, with an alert *before* the cap. The cap stops new calls; the alert gives you time to react before the cap stops them.
- **Hosting autoscale.** A maximum-instance count, or a maximum-bill alert. Most hosting platforms support both; turn them on at provisioning, not after the first surprise bill.
- **Metered third-party APIs.** Same shape — vendor-side cap plus an alert at a fraction of the cap.
- **Autonomous loops.** The kill switch from [Autonomous Loops](autonomous-loops.md) is also a cost circuit breaker. A loop that's spending too much is a loop you can stop in under a minute.

Caps fail closed: when hit, the system stops doing the expensive thing. Alerts fail open: they let you decide. Use both. Caps without alerts feel like outages; alerts without caps feel like discovery of next month's overrun.

### Rule 5 — Overrun is an incident

When a bill comes in materially over budget — say, 50% over, or any unexplained doubling — treat it the same way you treat a Tier 0 hotfix: stop the bleed first, investigate second, paper-trail third.

1. **Find the driver.** Vendor dashboards usually break spend down by day, route, or model. Find the top line and read its dates.
2. **Stop the source.** Hit the kill switch on the loop, downgrade the model tier, cut the cadence, whatever makes the spend stop today. The *right* fix can wait; the bleed cannot.
3. **Write a checkpoint** at `docs/checkpoints/<DATE>-cost-overrun-<slug>.md` with the budget, the actual, the driver, the immediate intervention, and one line on what the long-term fix looks like.
4. **Update the budget table.** Either the budget was wrong (raise it intentionally and document why) or the spend was wrong (the immediate intervention should bring it back into bounds).
5. **One line for the Strategist.** If the failure mode is novel — a new way for cost to spiral — it gets promoted to a rule, possibly a new entry in this file or a new prohibition in `AGENT.md`.

The principle: **a surprise bill is a governance failure, not a vendor failure.** Treat it the same way you'd treat a deploy that broke production — fix, document, learn.

## When the Strategist is tempted to "use the best model for everything"

Same shape as the Strategist's other temptations: feels efficient, costs more, and there's almost no situation where the most expensive model is actually required for the task at hand.

The pattern instead:

- **Default to the mid-tier model** for most Strategist work. Architecture, spec writing, handoff drafting, code review — all work at mid tier.
- **Escalate to the top tier deliberately** for genuine reasoning depth: novel architectural problems, hard debugging, multi-step planning where the cheap model's first draft will need three rounds of correction anyway.
- **Drop to the cheap-fast tier** for routine generation: writing a checkpoint template, summarizing a meeting, generating boilerplate. The savings compound across a session.

The five-second test: *what would I lose if I dropped one tier here?* If the honest answer is "nothing important," drop it.

## Worked example

A solo developer with two repos and an AI habit hits month six and discovers $480/month going out, of which they can account for $180 (two subscriptions, one hosting plan). They run the discipline:

1. **Owners and budgets.** Spend the 30 minutes building the budgets table. Six lines, total budget $250. The $230 gap is two surprises: an LLM API key used by a single cron in the orchestrator repo ($170), and a hosting tier auto-upgraded by that same cron's traffic ($60).
2. **Identify the driver.** The cron runs every 5 minutes, calls a top-tier model with a 4K-token prompt on each run. 288 runs/day × $0.12/run × 30 days = ~$1,037/month worst case; actually $170 because half the runs short-circuit. The cron is doing useful work but the cadence was set to "every 5 minutes" with no reason.
3. **Cut.** Drop the cron to hourly (12× cheaper), drop the model to mid tier ($0.02/run instead of $0.12, 6× cheaper). New steady-state: ~$2.40/month. Set a vendor-side cap at $20/month. Set an alert at $10/month.
4. **Downgrade the hosting tier.** The traffic spike that triggered the upgrade was the cron itself; with cron at hourly, the tier downgrades to free. Save $60.
5. **Checkpoint.** Write the incident at `docs/checkpoints/2026-05-30-cost-overrun-orchestrator.md`. New rule in `AGENT.md`: any new cron must declare cadence + model + estimated monthly cost in its handoff.
6. **Monthly review.** First review is two weeks later. New steady-state: $190/month, $60 under budget. The cost-log records the intervention so the next time a cron starts overspending, the playbook is one paragraph away.

The discipline traded a $300/month surprise for a 30-minute exercise that, repeated monthly, never lets that surprise recur.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "I'll watch the bills informally" | Informal attention drifts; surprises arrive monthly | Fixed review cadence, written budget, written log |
| Default to the most capable model on every call | Most tasks don't need it; cost compounds invisibly | Pick the model tier deliberately per task |
| Set a cron cadence to "as often as makes sense" | "Sense" rises to the bound the cheapest tier allows | Pick cadence by per-run value; document the reasoning |
| Autoscale with no cap | One viral moment becomes a five-figure bill | Cap + alert at provisioning, not after the first surprise |
| Spend ownership shared across "the team" | No one watches the bill | One named owner per line |
| Cut hard after an overrun, restore quietly when memory fades | Same overrun in three months | Update the budget *and* leave the circuit breaker in place |
| Open every relevant file into the Strategist's context "just in case" | Token spend per reasoning step balloons | Load the architecture graph + the specific files the design touches |
| Treat a surprise bill as the vendor's problem | The vendor's job is to bill; yours is to govern | Treat overruns as Tier 0 incidents with paper trails |

## Adapt it to your setup

- **Solo, low spend:** the budgets table can live in your password manager's secure note alongside the credentials. The monthly review is 5 minutes. The principle is identical at every scale.
- **Team:** every line still has an owner; the owners are humans with names. Spend reviews happen in a standing meeting; the cost log is shared.
- **Heavy LLM API usage:** add a per-repo line for direct API spend. Loops that call paid APIs declare an estimated cost in their handoff (rule 4 in [Autonomous Loops](autonomous-loops.md)).
- **Enterprise / multi-family:** budgets nest. Each family has a family-level budget that's the sum of its repo-level budgets plus a buffer. The shared-resource lines (orchestrator, registry) have their own line. Review cadence and circuit-breaker discipline are unchanged at scale; only the table is longer.

## Related

- [Autonomous Loops](autonomous-loops.md) — the highest-leverage cost surface; kill switches are also cost circuit breakers
- [The Two Roles](two-roles.md) — model-tier discipline rides on the Strategist/Executor split
- [Architecture Graphs](architecture-graphs.md) — the antidote to "load every file into context just in case"
- [The QA Gate](qa-gate.md) — where a "declared cost" check for new loops can live
- Back to the [main playbook](../README.md)
