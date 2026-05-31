# The Two Roles

> One AI that both plans and executes is its own reviewer — which means it isn't reviewed at all. Split it into a Strategist that designs and an Executor that ships, and let nothing cross the line between them except a written handoff.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** none — start here

---

## The wall

Hand a single AI the whole job and you get one of two failures.

Give your chat assistant filesystem access and it edits production blind — no type-check, no conventions, guessing at data it has never seen. Or let your terminal agent freelance the architecture and it commits hard to a half-formed plan, three files deep, before anyone weighed the design. A single agent that both plans and executes has no independent check on its own reasoning. That's fine until the day it's confidently wrong.

## The rule

**Two roles, one hard line.**

- The **Strategist** designs from outside the codebase and **never commits.**
- The **Executor** works inside the repo and is the **only** role that commits.
- The only thing that crosses the line is a **handoff** — a written spec the Executor implements under real tooling. Not a verbal "go ahead," not a snippet the Strategist wrote and the operator paraded in.

## How it works

The **Strategist** is your researcher and architect — strong at long-context reading, weighing tradeoffs, and writing detailed specs. It has four hard *nevers*: no editing source, no editing env/config, no database commands, no commits.

The **Executor** lives in the repo. It reads the [`AGENT.md` cascade](context-cascade.md) at startup, runs the build, tests, and DB queries, and pushes. Because it's the only role that can change the world, it's also the role that runs the [Sanity Check](sanity-check.md) — the check belongs with the eyes that can actually see production.

### The Operator: the implicit third role

Strategist and Executor are the AIs. The **Operator** is the human — the person whose repos, secrets, and reputation are on the line. They are not a fourth AI; they are the accountability layer the two AIs route around but never replace.

The Operator:

- **Approves deltas** when the Sanity Check finds a conflict. The AIs propose; the Operator decides.
- **Executes the Operator Action Block** — secrets, manual migrations, anything explicitly marked human-only in a handoff.
- **Resolves cross-role conflicts** — when Strategist and Executor disagree about scope or tier, the Operator is the tie-breaker.
- **Owns the trust dial.** Tightening or loosening Gate 3 of [Bounded Deviation](bounded-deviation.md), promoting a recurring delta into family standard — these are Operator decisions, not agent decisions.

In solo use, the Operator is *you* and the role is often invisible — you just approve as you go. In a team setting, the Operator is whoever owns the repo. Either way, naming the role explicitly is what stops the two AIs from quietly inheriting human accountability they were never built to hold.

### The Strategist's read-only inspection window

The four *nevers* above are about *writing.* They are not a prohibition on *reading.* The Strategist is encouraged to clone a repo for inspection — grep it, open files, quote real line numbers in the spec, sanity-check its own assumptions against the schema and routes before writing a handoff. A spec grounded in real file paths is dramatically cheaper for the Executor to execute than one written purely from the schema in the Strategist's head.

The limits are sharp:

- **Read only.** No `git add`, no `git commit`, no edits, no `npm install`, no `*.env` reads or writes, no DB queries.
- **No execution.** No running the test suite, no migrations, no scripts — reading is allowed because it can't change state; running can.
- **Don't substitute for the Sanity Check.** The Strategist sees the *code*; the Executor sees the *live data.* Read-only inspection improves the spec; it doesn't replace the Executor's pre-edit pass against production.

If the Strategist's inspection turns up something that would change the design — a function signature that's already different, an env var with a name collision — fold it into the handoff *before* writing it, rather than letting the Executor discover it as a delta.

**Tier routing** decides who owns a task:

| Tier | Owner |
| :---- | :---- |
| 0 Hotfix · 1 Quick fix · 2 Standard | Executor |
| 3 Strategic (architecture, cross-repo, protocol) | Strategist designs → Executor executes |

The handoff is the interface between them: file paths, signatures, env-var names, an ordered plan, acceptance criteria, and an Operator Action Block of manual steps. See [Checkpoints & Handoffs](checkpoints-handoffs.md).

**Tier is provisional.** Assignment is a starting hypothesis, not a contract. If a Tier-2 task uncovers architectural ambiguity mid-design — a new pattern, a cross-repo implication — the Executor escalates it back as Tier 3 before continuing. Going the other way, a Tier-3 spec whose Sanity Check shows the design fits cleanly can drop to Tier 2 with the Strategist's blessing. **The Executor never silently re-tiers**; every escalation or demotion is logged in the checkpoint with a one-line reason.

**Tier 0 ships first, files paper second.** A hotfix doesn't wait for a handoff — by definition there's no time. But the protocol isn't skipped, it's *backfilled*. After the fix lands and the bleed stops, the Executor writes a retroactive checkpoint at `<repo>/docs/checkpoints/<DATE>-hotfix-<slug>.md`: what was broken, how it was detected, the diff that shipped, why it couldn't wait, and a one-liner for the Strategist on whether anything here belongs in a family-wide rule. The hotfix isn't done until the backfill exists. The next Strategist session reads it the same way it reads any delta.

## Worked example

A Tier-3 task: *"add multi-tenant support."*

- The **Strategist** researches approaches, then writes a spec: the data-model changes, the migration order, the env vars required, explicit acceptance criteria, and an Operator Action Block (*"run the migration — you, not the agent"*).
- The **Executor** picks it up, sanity-checks against the live schema, implements step by step, runs the [QA gate](qa-gate.md), and pushes.

Contrast the anti-pattern: mid-chat, the Strategist says *"this is easy, I'll just edit the schema file"* and writes a migration against a data model it has never queried. That's Lesson 1 in one sentence, and it's exactly the line this split exists to hold.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| One agent plans *and* executes | No independent check on the plan | Separate the roles — even two sessions of the same tool |
| Strategist has commit access | Blind edits reach production | Strategist writes specs only; Executor commits |
| Executor invents architecture mid-task | Commits to an unreviewed design | Escalate to Tier 3; let the Strategist design first |
| "Just this once" line-crossing | The exception quietly becomes the norm | Hold the line; write the one-line spec |

## Adapt it to your setup

- **Solo:** you're the Operator. The two AI roles can be two tabs or two terminals — a planning session and an execution session. The discipline is keeping them separate, not owning two products.
- **One tool that can do both** (a terminal agent that also plans well): keep the *discipline* of the split anyway — a distinct planning pass that produces a written spec, then a distinct execution pass against it. The artifact is the point.
- **Same-modality pairings.** Two chat-based AIs (a Perplexity Strategist and a ChatGPT review pass, say) or two terminal-based ones (Claude Code planning vs. Codex executing) both work fine. The split is about *who has commit access*, not about chat-vs-terminal. As long as the planning agent has no write access and the executing agent does, the discipline holds.
- **Multiple Strategists.** Nothing in the model says you can only have one. The shop this playbook came from runs Perplexity and Claude side by side as paired strategists — different long-context strengths, different blind spots. The handoff to the single Executor is the convergence point.
- **Choosing tools:** Strategist = strong research / long-context / architecture, no repo access needed. Executor = repo-native, runs your build, commits. See the [actor table](../README.md#-works-with-any-strategist--executor).

## Related

- [Checkpoints & Handoffs](checkpoints-handoffs.md) — the artifact that crosses the line
- [File Handling](file-handling.md) — the "tempted to just fix it quick" pattern and the escape valve
- [The Sanity Check](sanity-check.md) — why the Executor, not the Strategist, validates against production
- [Bounded Deviation](bounded-deviation.md) — what the Executor may do without asking
- [Governance Sync](governance-sync.md) — how Strategist and Executor stay aligned across sessions
- [The Context Cascade](context-cascade.md) — the rules the Executor reads at startup
- [Architecture reference](architecture.md) — where this role split sits in the larger model
- Back to the [main playbook](../README.md)
