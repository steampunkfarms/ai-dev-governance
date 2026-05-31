<h1 align="center">AI Dev Governance</h1>

<p align="center">
  <strong>A field-tested model for running AI coding agents across many repos — a planner that can't commit, an executor that can, and the file discipline that keeps them honest.</strong>
</p>

<p align="center">
  <em>Actor-agnostic. Pair any strategist-style AI with any executor-style agent.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-living%20document-brightgreen" alt="Status: living document">
  <img src="https://img.shields.io/badge/works%20with-any%20strategist%20%2B%20executor-blue" alt="Works with any strategist + executor">
  <img src="https://img.shields.io/badge/read-~10%20min-orange" alt="Read time ~10 min">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey" alt="License CC BY 4.0"></a>
</p>

<!-- Add your Runway hero banner at assets/hero.png, then this renders. Delete these two lines if you don't want a banner. -->
<p align="center">
  <img src="assets/hero.png" alt="AI Dev Governance — a Strategist plans, an Executor ships, files keep them in sync" width="820">
</p>

---

**Your AI agent forgot everything from yesterday. It just overwrote a file that was working. And the "plan" it confidently executed was based on a database that hasn't looked like that in weeks.**

If you've felt any of that, you've hit the wall this repo is about. It's not a tool to install — it's the governance model a small shop evolved while running two AI actors across ~20 repositories, written down so you can skip the painful parts. Every pattern here is the scar tissue of a specific mistake.

> **The goal isn't to make AI write more code faster — it's to make sure the code it writes is the code you actually wanted, every time, without you watching it type.**

**Who this is for:** solo developers and small teams using AI coding agents — usually a chat assistant *plus* a terminal agent — across more than one repo. New devs get a structure to model from before the walls show up; experienced devs hitting those walls get a map out.

**This README is the manifesto.** The working reference — foundations, layered protocols, lifecycle — lives in [`docs/architecture.md`](docs/architecture.md). If you'd rather skip the theory and adopt the minimum viable version today, jump to [Getting Started](docs/getting-started.md) — a 30-minute on-ramp. Each facet has its own deep-dive in [`docs/`](docs/).

---

## 🚀 How to use this repo

You don't read this like a tutorial. You hand it to your AI and let it adapt the model to *your* setup.

**1. Point your Strategist at this page.** Paste the prompt below into whatever chat-based AI you use, with this README in context (paste the raw file, or give it the repo URL).

```text
You are acting as my Strategist. Read this entire page top to bottom,
then read docs/architecture.md.

Then, given my setup:
- Repos / projects: <e.g. 3 Next.js apps + a shared package>
- My Strategist tool: <e.g. ChatGPT in the web app>
- My Executor tool: <e.g. Codex, Claude Code, Cursor, or Grok Build in my terminal>
- Where my files live: <e.g. ~/dev, GitHub org "acme">

Assess how to adapt this governance model to MY environment. Tell me:
1. Which of my tools should play Strategist vs Executor, and why.
2. What my AGENT.md cascade should look like (global → family → project).
3. The single smallest first step I can take this week.
Flag anything in this model that doesn't fit my situation — don't force it.
```

**2. Map the two roles to your tools** (see below — the roles matter more than the brands).

**3. Adopt the minimum core first:** the two-role split and checkpoints. That alone kills most of the pain.

**4. Grow into the rest** — the cascade, the Sanity Check, the roadmap index — as your repo count climbs. The [architecture reference](docs/architecture.md) and the [deep dives](#-deep-dives) cover each facet in isolation.

---

## 🧩 Works with any Strategist + Executor

This model is **actor-agnostic.** It describes two *roles*, not two products. Slot in whatever you already use.

| Role | What it needs to be good at | Example tools |
| :---- | :---- | :---- |
| **Strategist** | Research, architecture, writing specs. Works *outside* the repo. Needs no commit access — and shouldn't have it. | Perplexity, ChatGPT, Claude (web/app), Gemini, Grok |
| **Executor** | Lives *inside* the repo. Runs the build, tests, and DB queries. Commits and pushes. | Codex, Claude Code, Cursor, GitHub Copilot, Gemini CLI, Grok Build, Aider |

**Example pairings** — all equally valid:

| Strategist | Executor |
| :---- | :---- |
| Perplexity | Claude Code |
| ChatGPT | Codex |
| Claude (app) | Cursor |
| Gemini (web) | Gemini CLI |
| Grok | Grok Build |

…or any combination. Pick whatever you already pay for; the discipline is in the *handoff between the roles*, not the logos. The shop this playbook came from runs Perplexity and Claude as paired strategists with Claude Code as executor — the model genuinely doesn't care.

---

## The six lessons at a glance

1. The planning AI is blind and amnesiac — so don't let it touch code.
2. Sessions die mid-task — so write to disk constantly and verify.
3. A spec written blind will collide with production — so reality-check before executing.
4. Rules conflict across scopes — so define a precedence order up front.
5. "Roadmaps" multiply and drift — so stop expecting them to sync.
6. "Done" means shipped and verified — not "the code is written."

---

## Part I — Six lessons that shaped everything

Each of these began as a painful mistake. The protocols in the [architecture reference](docs/architecture.md) exist to make each mistake hard to repeat.

### 1. The planning AI is blind and amnesiac — so don't let it touch code

A chat-based Strategist can't run your type-checker, doesn't read your repo's rules at session start, and remembers nothing from the last session. It's excellent at research, architecture, and spec-writing — and risky the moment it edits a real file, because it's guessing at production state.

**The rule:** the Strategist never writes source, never edits env/config, never runs the database, never commits. When it's tempted to "just fix it quick," it writes a **handoff spec** instead, and the Executor applies that spec with type-checking and conventions in force. Persistence lives in files, never in the assumption that an agent will "remember."

### 2. Sessions die mid-task — so write to disk constantly and verify

AI sessions time out, and a cloud agent's working filesystem is ephemeral — it resets when the session ends. Batching all your writes "for the end" turns a timeout into total loss.

**The rule:** write directly to the persistent (operator's) filesystem, incrementally, one file per completed step. Then **read the file back** — a "success" response from a write tool is not proof on its own. And drop a **checkpoint** at the start of any multi-step task, after each major step, and at the end of every session, so the next session resumes exactly where this one stopped.

### 3. A spec written blind will collide with production — so reality-check before executing

The Strategist designs without seeing live data. Treat **every handoff as a design, not a law.** Before implementing one, the Executor runs a quick reality check against production: existing records, naming collisions, foreign-key relationships, and anything already sent, paid, or deployed.

This rule was bought the hard way. Early on, a spec renamed a billing record that had *already issued invoices* — and those invoices then displayed under the wrong name. A reality check that queried the live billing records first would have caught it. That's why specs are now validated against production before a single line lands. → [Full walkthrough](docs/sanity-check.md)

### 4. Rules conflict across scopes — so define a precedence order up front

In a multi-repo enterprise, a global rule, a shared-infrastructure rule, a team-wide convention, and a project-specific override can all apply to the same task. Without a declared order, the agent picks one at random.

**The rule:** a written precedence cascade decides which wins. More specific scopes override more general ones for project details; more general scopes still govern routing and cross-project behavior. → [The Context Cascade](docs/context-cascade.md)

### 5. "Roadmaps" multiply and drift — so stop expecting them to sync

A real codebase grows several "what's left to build" surfaces: a product roadmap (decisions, pricing, version planning), an execution roadmap per repo (what ships next), and often a database-backed dashboard. People assume these stay in lockstep. They don't, and trying to auto-sync them is a trap.

**The rule:** keep the genres separate on purpose, accept that mismatches are normal, and maintain **one index** as the single entry point that catches drift *on audit* rather than pretending everything is always aligned.

### 6. "Done" means shipped and verified — not "the code is written"

The cheapest place to catch a regression is before "complete" is declared.

**The rule:** Tier-2-and-up work isn't done until it passes a fixed gate — type-check clean, diff shows only intended files, no hardcoded secrets, acceptance criteria verified, pushed to main, and the host's build confirmed green. The deploy *is* the review.

---

## Part II — The architecture (reference)

The six lessons above tell you *what failed* and the shape of the discipline that prevents each failure. The **[architecture reference](docs/architecture.md)** turns that into a working manual: four foundations, four layered protocols, the end-to-end lifecycle of one phase, and the scaling-down rules so you don't over-engineer a one-repo setup. Read it second, after the lessons above have given you the *why*.

At a glance:

- **Foundations:** repo schema & routing · context-file cascade · two-role framework · file-handling protocol
- **Layered protocols:** the Sanity Check + Bounded Deviation · post-execution QA gate · Governance Sync · the roadmap system
- **Lifecycle:** the eleven-step walkthrough of one Tier-2 phase from handoff to push to next-session resume

→ Read the full [architecture reference](docs/architecture.md).

---

## ⚠️ When you've adopted too much of this

The model scales down on purpose. A one-repo solo dev doesn't need everything below; an enterprise running 30 repos probably does. The fastest way to lose value here is to copy the maximal version of the playbook onto a setup that doesn't have the failure modes it was built to prevent.

You've over-engineered this if:

- You're maintaining a four-level cascade for a single repo.
- You're writing handoffs to yourself for a session you'll execute ten minutes later.
- You're keeping a roadmap *index* when you only have one roadmap.
- You're running the full QA gate on Tier-0 hotfixes.
- You're treating the Strategist/Executor split as different *products* when one tool could play both roles with discipline.

**The transferable core is small:** two roles, files as the only memory, reality-check before executing, declared precedence, accept drift and catch it on audit. Everything else earns its place by surviving a real failure first.

The full scale-down table is in the [architecture reference](docs/architecture.md#when-youre-over-engineering-this).

---

## 📚 Deep dives

Each facet of the model, presented in isolation with worked examples. Start with whichever wall you're hitting.

| Deep dive | What it covers |
| :---- | :---- |
| [Getting Started](docs/getting-started.md) | The smallest useful adoption: two roles, one `AGENT.md`, one checkpoint — in 30 minutes |
| [The Two Roles](docs/two-roles.md) | Choosing your Strategist and Executor; drawing the line between them; the implicit Operator role |
| [The Context Cascade](docs/context-cascade.md) | Building an `AGENT.md` hierarchy that resolves conflicts cleanly |
| [File Handling](docs/file-handling.md) | Which tool, which filesystem, verify by reading back, recover from crashes |
| [Checkpoints & Handoffs](docs/checkpoints-handoffs.md) | Templates, the session-resume protocol, and the state cascade when your governance file outgrows one page |
| [The Sanity Check](docs/sanity-check.md) | Validating a blind spec against live production before it runs |
| [Bounded Deviation](docs/bounded-deviation.md) | The exact rule for when an Executor may stray from the spec |
| [The Delta Log](docs/delta-log.md) | Counting deltas across families until a pattern becomes a standard |
| [Governance Sync](docs/governance-sync.md) | Keeping Strategist and Executor in sync through files; handling sync conflicts |
| [The Roadmap System](docs/roadmap-system.md) | Two genres, the index, and why surfaces are allowed to drift |
| [The QA Gate](docs/qa-gate.md) | The fixed checklist that defines "done" |
| [Architecture Graphs](docs/architecture-graphs.md) | Generated maps of repo structure that turn the Strategist from blind to fluent |
| [Autonomous Loops](docs/autonomous-loops.md) | Kill switches, heartbeats, and the discipline for code that runs when no one is watching |
| [Secrets and Rotation](docs/secrets-and-rotation.md) | Credential hygiene with two AIs in the room; rotation and leak protocols |
| [Dependency Management](docs/dependency-management.md) | Lockfile-as-truth; cadences for patch / minor / major; monthly audit |
| [Cost Management](docs/cost-management.md) | Budgets, monthly review, circuit breakers, and treating overruns as incidents |

> Adding a deep dive? Copy [`docs/_deep-dive-template.md`](docs/_deep-dive-template.md) so every facet reads the same way.

---

## Adapting this to your setup

You don't need three families, a dashboard, or an orchestrator to use most of this. The transferable core is small:

- **Split the AI into a planner that can't commit and an executor that can.** Most failures trace back to a blind planner editing live files.
- **Make files the only memory.** Checkpoints, handoffs, and changelogs — not the assumption that an agent remembers.
- **Treat every plan as a hypothesis and reality-check it against production before executing.**
- **Write down which rule wins when rules conflict, before you need the answer.**
- **Let your tracking surfaces drift, and keep one index that catches the drift.**

Start with the two-role split and checkpoints. Add the cascade, the Sanity Check, and the roadmap index as your repo count grows and the failure modes above start to bite.

---

## 🔗 New here?

If the manifesto resonates and you want the shortest path from "interesting" to "in production":

1. **[Getting Started](docs/getting-started.md)** — adopt two roles, one `AGENT.md`, and one checkpoint on your next task. 30 minutes.
2. **Hit a wall.** Use the trigger table in that doc to pick the next deep-dive when a specific failure mode arrives.
3. **Graduate to the [architecture reference](docs/architecture.md)** when the model has earned your trust on real failures.

---

## 🤝 Contributing

Found a wall we didn't cover, or a cleaner way to handle one we did? Open an issue describing the failure mode and how your approach prevents it. New facet docs should follow the [deep-dive template](docs/_deep-dive-template.md). See [CONTRIBUTING.md](CONTRIBUTING.md).

## Using these patterns

Borrow it, adapt it, make it yours. Prose is licensed [CC BY 4.0](LICENSE); attribution is appreciated.

---

<p align="center"><strong>Stop babysitting your agents. Start governing them.</strong></p>
