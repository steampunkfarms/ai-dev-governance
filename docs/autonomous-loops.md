# Autonomous Loops

> The Strategist and Executor are bounded by sessions: a human starts them, a human approves their deltas, a human watches the deploy. **Autonomous loops aren't bounded by sessions.** A cron job, a webhook handler, an inbound-event consumer, a long-running agent — these run when no one is looking, and they're the part of the model most likely to silently corrupt state.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The QA Gate](qa-gate.md) · **See also:** [Foundation 1 in the architecture reference](architecture.md#foundation-1--repo-schema--routing)

---

## The wall

You ship a cron that posts to social, or a webhook that mutates a database row, or an agent that triages incoming email. It works in testing. It works the first night. Then a week later you discover one of three things:

- It ran every hour for three days against the wrong account, and nobody noticed because nobody was watching.
- It silently stopped two weeks ago and you only found out because a customer asked why their thing didn't happen.
- It's still running fine, but you have no way to tell — there are no logs, no metrics, no last-success-at timestamp. You're trusting that no news is good news.

The pattern under all three: **the loop has no operator in the room.** Everything else in this playbook assumes a Strategist designs, an Executor implements, and a human approves. Autonomous loops break that assumption. The discipline that protects the rest of the system — checkpoints, Sanity Check, QA gate — was built for human-paced cycles. Loops run on their own clock.

This deep dive is about the three things every autonomous loop needs that a session-based task doesn't: **a kill switch, observability that's checked, and an owner.**

## The rule

1. **Every autonomous loop has a documented kill switch the operator can hit in under a minute, from a phone, without an agent.**
2. **Every autonomous loop emits a heartbeat — and something checks the heartbeat.** An unwatched log is not observability.
3. **Every autonomous loop has a named owner — the repo or family it belongs to — and shows up on a single shared health surface.** No anonymous infrastructure.
4. **Loops follow the same tier discipline as everything else.** A loop that mutates production state is Tier 2 at minimum; a loop that touches money, external sends, or shared infrastructure is Tier 3 and needs Strategist sign-off on the design.
5. **The first deploy of a new loop runs in dry-run mode for one cycle.** Real schedule, real triggers, real logs — fake side effects. You watch one full cycle complete in dry-run before flipping the switch.

## How it works

### Rule 1 — The kill switch

A kill switch is not a feature; it's a precondition for shipping. Before a loop goes live, the operator must know — and have documented, in the repo's `AGENT.md` — exactly how to stop it. The bar:

- **One action, under a minute, from a phone.** A `DISABLED=true` env var the operator can set in the hosting console qualifies. A code-level `if (!enabled) return` behind a config flag readable from outside the repo qualifies. A `git revert` + redeploy does *not* qualify — that's a recovery procedure, not a kill switch.
- **Tested before first run.** Set the switch, trigger the loop, confirm nothing happens, unset, confirm normal operation. Document the verification in the `AGENT.md`. Untested kill switches usually don't work — they fail closed in unexpected ways, or the flag is read at startup and the running process ignores it.
- **Discoverable without context.** A future operator (or future-you at 2am) should be able to find the kill switch from the loop's symptom alone. If the only way to find it is to read the source, it's not a kill switch — it's a debugging exercise.

The pattern that consistently works: a single `<LOOP_NAME>_ENABLED` env var per loop, read on every invocation (not at boot), defaulting to `false`. Shipping a loop means flipping it to `true` in the hosting console. Killing a loop means flipping it back. The `AGENT.md` lists the loop name, the env var, the hosting console URL, and the verification steps.

### Rule 2 — Heartbeats that get checked

Most loops have logs. Logs are not observability. Observability is **a signal something watches and alerts on.** The minimum:

- **A success heartbeat.** At the end of each successful run, the loop records a `last_success_at` timestamp somewhere durable — a database row, a key in a metrics store, a status file in shared storage. Not a log line.
- **A staleness check.** A second loop — or a health endpoint, or an external uptime service — reads the heartbeat and alerts when it's older than the loop's expected interval plus a tolerance. A daily cron with a 6-hour tolerance; an hourly cron with a 15-minute tolerance.
- **A failure heartbeat.** Failed runs record `last_failure_at` and a one-line reason. The alert fires on either staleness *or* repeated failure.

The three failure modes this catches: silent stoppage (no heartbeat update), silent degradation (heartbeat updates but failure rate climbs), and the harder one — **silent success-with-wrong-side-effect** (the loop runs, the heartbeat updates, but the work it did was wrong). Rule 5 below addresses that one.

The principle: *if there is no signal that fires when the loop is unhealthy, the loop is invisible.* Invisible loops always end up doing the wrong thing for longer than you'd expect.

### Rule 3 — Named owner, shared surface

Every loop belongs to exactly one repo or family. That repo's `AGENT.md` lists it: name, what it does, schedule, kill switch, heartbeat location. No exceptions for "shared utilities" — pick a home, name it, document it.

In addition, every loop in the enterprise appears on **one shared health surface** — a single page, dashboard, or document the operator can open to see "are all my loops alive?" The surface lists each loop, its owner repo, its expected interval, its last-success-at, and its current status. Most enterprises will end up with three to twenty loops; without a single surface, no one ever has the full picture.

The surface itself can be modest. A markdown file in the governance repo, updated weekly. A spreadsheet. A tiny dashboard widget. What matters is that it exists and that the operator looks at it on a fixed cadence — every Monday, every release, whatever. A health surface no one reads is the same as no health surface.

### Rule 4 — Tier discipline for loops

A loop's tier is set by *blast radius*, not by code complexity:

| Loop shape | Tier | What that means |
| :---- | :---- | :---- |
| Read-only ingestion (poll an RSS feed, mirror a public file) | **1** | Executor ships it; QA gate applies. No Strategist needed. |
| Internal-state mutation (recompute a cache, tidy a table) | **2** | Executor ships with Sanity Check against production; full QA gate. |
| External side effects (post to social, send email, charge a card) | **3** | Strategist designs the loop *and* the failure-handling envelope. Operator approves. |
| Touches shared infrastructure (writes to the orchestrator, edits the registry) | **3** | Strategist always. Affects every other loop. |

Tier 3 loops also need a written **failure envelope**: what happens if this loop fails for one cycle? Three cycles? A week? The envelope tells the operator whether a stale heartbeat is "wake me up" or "look at it on Monday."

### Rule 5 — Dry-run the first cycle

Before flipping a new loop to live, run it on its real schedule with side effects faked. The pattern:

1. Add a `<LOOP_NAME>_DRY_RUN=true` env var alongside the enabled flag.
2. In dry-run mode, every action that has a side effect — a write, a send, an API call — is replaced with a log line that says *what would have happened.* The heartbeat still updates. The Sanity Check still runs.
3. Ship the loop in dry-run for at least one full cycle. Read the logs. Confirm the actions the loop intended match what you actually want.
4. *Then* flip dry-run off. Watch the first live cycle by hand.

This is the single discipline most often skipped, and it's the one that catches the silent-success-with-wrong-side-effect failure mode. A loop that *runs* correctly but *acts* incorrectly is the most expensive bug class in autonomous-loop work, because the heartbeat is green the whole time. Dry-run is how you see what the loop would have done before it actually does it.

## When a loop misfires

Recovery has its own protocol — the same shape as the [error-recovery rule for file handling](file-handling.md#rule-5--error-recovery), applied to a loop instead of a session.

1. **Hit the kill switch first.** Stop the bleed before you investigate. A misbehaving loop continues to misbehave on its schedule while you debug.
2. **Read the heartbeats backward.** Find the last known-good cycle. Everything between that cycle and now is suspect.
3. **Inventory the side effects.** For loops that mutate state or send things, list every action between the last-good and now. This is the cleanup surface.
4. **Decide: reverse, leave, or compensate.** Some actions are reversible (delete a row, recall a draft). Some are not (an email was sent, a payment cleared). For the irreversible ones, the operator decides whether to compensate (send a correction) or leave it.
5. **Backfill a checkpoint.** Treat the incident exactly like a Tier 0 hotfix: write a retroactive checkpoint at `docs/checkpoints/<DATE>-loop-incident-<slug>.md` with the trigger, the diff (or config change) that shipped, and one line for the Strategist on whether the failure should become a new standard.

The principle is the same as for sessions: **the plan is intent; the heartbeat is fact.** When they diverge, trust the heartbeat.

## Worked example

A small shop adds a cron that posts a daily summary to a Facebook page. The naive path: write the function, set a cron expression, deploy. The first week it works. The second week, the page's access token expires silently, the cron logs an error, no one reads the logs, the page goes dark for nine days, and the operator finds out from a customer.

The disciplined path:

1. **Tier assignment.** Loop has external side effects (it posts to a page). Tier 3. Strategist designs the failure envelope: "if no post for 48 hours, alert the operator." Operator signs off.
2. **Kill switch.** `FACEBOOK_POSTER_ENABLED` env var, read on every invocation, defaulting to `false`. Tested by setting `false`, triggering, confirming no post; setting `true`, confirming a post. Documented in the repo's `AGENT.md` with the hosting console URL.
3. **Heartbeats.** Each successful post writes `last_success_at` to a `loop_heartbeats` table. A second cron — the *staleness checker* — runs every 2 hours and emits an alert if `last_success_at` is older than 26 hours. Failed runs write `last_failure_at` plus the error message.
4. **Health surface.** The loop is added to the shared `governance/health-surface.md` table: name, owner repo, schedule, kill switch, last-success-at link, expected interval.
5. **Dry-run.** Deployed with `FACEBOOK_POSTER_DRY_RUN=true` for two full cycles. Logs reviewed; the would-be posts looked right. Dry-run flipped off, first live post watched manually.
6. **Failure envelope.** Week three, the token expires. The post fails. `last_failure_at` updates with `OAuthException: token expired`. Two hours later, the staleness checker fires the alert. Operator hits the kill switch from a phone, then opens a CC handoff to rotate the token. Total dark time: zero — because the next-scheduled post never happened in a broken state.

The discipline traded a 30-minute up-front setup for a 9-day silent outage averted. That ratio is typical.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Ship a loop with no kill switch | A misfiring loop runs on its schedule while you scramble for a fix | Document and *test* the kill switch before first live run |
| "It logs errors, that's enough" | Logs no one reads are invisible | Heartbeats + an alert that fires on staleness or failure |
| Use a `git revert` as the kill switch | Slow, risky, and unavailable from a phone | A runtime env var read on every invocation |
| Skip dry-run because "it's a small loop" | Side-effect bugs are silent and compound | Dry-run for at least one full cycle, every time |
| One alert channel for all loops, no owner | When something fires, no one knows whose loop it is | Each loop names its owner repo; the alert includes the owner |
| Treat loops as infrastructure, not as code that ships | Loops drift out of the governance model the rest of your code follows | Loops get tiers, Sanity Checks, QA gates, and checkpoints — same as everything else |
| Read the kill-switch flag once at boot | A long-running process never sees the change | Read on every invocation; for long-running processes, on every iteration of the main loop |
| Health surface that no one looks at | Same as no health surface | Pair the surface with a fixed cadence (Monday, every release) |

## Adapt it to your setup

- **Solo, one or two loops:** the "shared health surface" is a five-row table in a single markdown file. The kill switch is still a runtime env var. The dry-run is still mandatory. You don't need anything fancier than that.
- **Cron-as-code (Vercel, Cloudflare, GitHub Actions):** the kill switch can be a config-level "disable schedule" toggle, *if* you've confirmed it stops the currently-queued execution and not just future ones. If the platform queues runs in advance, layer a runtime flag on top.
- **Long-running agents (a triage bot, a watcher daemon):** the heartbeat is more frequent — every cycle of the agent's main loop, not just every "completion." The kill switch is read on every iteration so the agent can exit cleanly mid-task.
- **Webhook handlers:** the schedule is "whenever a webhook fires," but the discipline is identical. Heartbeat per invocation. Kill switch on every invocation. Dry-run for one production webhook before going live.

## Related

- [The QA Gate](qa-gate.md) — the post-execution checklist that applies to loop deploys just as it does to feature deploys
- [The Sanity Check](sanity-check.md) — runs inside the loop body for state-mutating loops, exactly as it would inside a Tier 2 handoff
- [Foundation 1 in the architecture reference](architecture.md#foundation-1--repo-schema--routing) — where the orchestrator/shared-resource pattern lives
- Back to the [main playbook](../README.md)
