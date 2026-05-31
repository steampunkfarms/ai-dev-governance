# Dependency Management

> The Strategist designs in current-state assumptions. The Executor ships against the lockfile. The gap between those two — the packages you depend on, their versions, the transitive graph beneath them — is where AI-assisted teams quietly lose more time than to any other single category. **Lockfiles are governance.** This file is about treating them that way.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The QA Gate](qa-gate.md) · **See also:** [Secrets and Rotation](secrets-and-rotation.md), [Architecture Graphs](architecture-graphs.md)

---

## The wall

Dependencies break in four shapes, and AI workflows amplify each one:

- **The phantom import.** The Strategist writes a handoff that imports `@vendor/sdk` because that's the package name in the vendor's docs. The Executor runs the handoff. The package isn't installed in this repo. The Strategist guessed because the chat-based AI was trained on the docs, not on this `package.json`. (This is the same failure mode that produced commit `1e7d027` in the public history of this very repo — a Strategist spec referenced a package that wasn't in the lockfile.)
- **The version mismatch.** Three repos all depend on `@vendor/sdk`. Two are on v3, one is on v4. The Strategist writes a handoff that uses the v4 API surface. The Executor ships it. The two v3 repos still type-check because the import lives behind a feature flag — but at runtime, the call fails. There was no warning because nothing in the system knew the versions had diverged.
- **The silent upgrade.** A dependency releases a minor version with a breaking change that doesn't show in the changelog. CI passes because the test suite doesn't exercise the affected surface. Production fails three days later when a real user hits the path.
- **The dead dependency.** A package hasn't been touched in 14 months. The maintainer abandons it. A CVE drops. Nothing in your toolchain tells you you're now running unpatched code; you find out from a security advisory you happen to read.

Each failure has a different root, but the discipline is shared: **upgrade cadence is a Strategist concern, lockfile integrity is an Executor concern, and the operator owns the audit cadence that catches what both miss.**

## The rule

1. **The lockfile is the source of truth.** `package.json` (or `requirements.txt`, `Cargo.toml`, `go.mod`) lists intent; the lockfile lists fact. The Strategist designs against the lockfile, not against the manifest.
2. **A package the Strategist references in a handoff must be in the lockfile already, or the handoff explicitly says "add this dependency."** No phantom imports.
3. **Upgrades happen on a declared cadence,** not opportunistically. Each repo's `AGENT.md` names the cadence and the upgrade discipline.
4. **A single audit pass per month** — across every repo — produces a list of dependencies that are outdated, abandoned, or vulnerable. The list is the input to the next month's upgrade cadence.
5. **Major upgrades are Tier 3.** Strategist designs; Executor implements; Operator approves. No major upgrade slides in as a side effect of "let me just update the lockfile."

## How it works

### Rule 1 — Lockfile is truth

The lockfile (`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `requirements.lock`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, …) is committed to the repo, and is what gets installed in CI and production. The manifest tells you which packages were *requested*; the lockfile tells you which versions actually shipped, transitively.

For the Strategist, this means: when you need to know whether a package exists in a repo, *read the lockfile*, not the `package.json`. A package in `package.json` but not in the lockfile is half-installed; a package in the lockfile but not in `package.json` is a transitive dependency the repo doesn't control directly. Both situations require Executor follow-up before a handoff can rely on them.

For the Executor: never run an install that updates the lockfile as a side effect of unrelated work. If a handoff is supposed to be "add a route," and the install step bumps 47 transitive dependencies, the handoff has grown a second job. Split it: ship the route on the existing lockfile, then file a separate upgrade handoff for the bumps.

This discipline is what makes the lockfile a stable read for the Strategist. If the lockfile churns on every commit, it stops being a source of truth and starts being noise.

### Rule 2 — No phantom imports

The Strategist's biggest dependency-shaped failure is referencing a package that isn't installed. The discipline that prevents it:

- **The Strategist reads the lockfile** for any handoff that touches third-party packages. Reading is cheap; assumption is expensive.
- **The architecture graph** ([architecture-graphs.md](architecture-graphs.md)) includes the package list and version per repo. The Strategist's first read on a Tier 3 change includes the graph, which means the package list is already in context.
- **Handoffs that introduce a new dependency say so explicitly.** A line at the top of the handoff: `New dependencies: @vendor/sdk@^4.1.0 (Executor to add).` The Executor installs as a discrete step, runs the QA gate, and ships. The new package is now in the lockfile for next time.
- **The Executor flags phantom-import bugs in the Sanity Check.** If the spec references `@vendor/sdk` and the package isn't installed, the Executor emits a delta: "spec assumes `@vendor/sdk`, not present; options are (a) add the dep, (b) use the already-installed `@othervendor/client`, (c) raw `fetch()`." Operator picks; Executor implements.

The five-second test for the Strategist: *did I read the lockfile before referencing this package?* If no, the handoff has a hole.

### Rule 3 — Cadence, not opportunism

Each repo's `AGENT.md` declares its upgrade cadence. A typical pattern:

- **Patches** — applied weekly, by a small dependabot-style automation, auto-merged if CI passes. Patches are mostly safe and the cost of falling behind is mostly security debt.
- **Minor versions** — applied monthly, in a single batched upgrade handoff. The Executor runs the upgrade, runs the QA gate, ships if green. Bumps a release-notes review into the changelog so anything notable is recorded.
- **Major versions** — never automatic. A major version is a Tier 3 change that requires a Strategist-designed migration handoff (see rule 5).

The cadence is *declared*, not aspirational. A repo that doesn't run its patch cadence for three months has a debt problem that compounds — and the longer the debt sits, the more likely an upgrade hits a breaking change buried under five other breaking changes. Monthly is the slowest reasonable cadence for minors; weekly is the fastest reasonable cadence for patches.

For repos that genuinely don't change much, the cadence can be quarterly — but the cadence must still exist. "Whenever we get around to it" is not a cadence.

### Rule 4 — Monthly audit pass

Once a month, the operator runs a cross-repo audit:

- **Outdated.** `npm outdated`, `pip list --outdated`, `cargo outdated`, etc., across every repo. Output goes into a single `dependency-audit.md` in the governance repo.
- **Abandoned.** For any dependency that hasn't released in 18 months, flag it. Some of these are stable (a mature library that doesn't need to change); some are abandoned (the maintainer has moved on). The audit doesn't decide which; it flags for the Strategist to triage.
- **Vulnerable.** Run `npm audit`, `pip-audit`, `cargo audit`, or whatever your ecosystem provides. Critical findings escalate immediately as Tier 1 fixes; high findings go into the next month's upgrade cadence.
- **Diverged.** For dependencies present in multiple repos, flag any version divergence. Divergence isn't always wrong — sometimes two repos legitimately need different versions — but it should be deliberate, not accidental.

The audit produces a one-page summary that goes into the monthly cost/dependency review. Each finding gets one of three dispositions: *upgrade now* (next sprint), *upgrade later* (declared deadline), *accept* (with a one-line reason in the audit log).

### Rule 5 — Major upgrades are Tier 3

A major version bump (`v3 → v4` in semver, or any release with explicit breaking-change notes) is never a routine update. The discipline:

- **Strategist reads the release notes** end to end. Identifies every breaking change that touches surfaces this repo uses.
- **Strategist writes a migration handoff.** Per breaking change: which files are affected, the before/after of the API surface, any data migrations required, any environment changes.
- **Operator approves** the handoff before the Executor begins. Major upgrades have a non-trivial chance of needing a rollback; the operator should be aware before the rollback becomes necessary.
- **Executor implements** with the standard QA gate, plus a smoke-test pass against the upgraded surface in staging if the repo has one.
- **Changelog entry** records the migration: from-version, to-version, breaking changes addressed, deltas encountered.

The slowest you can do a major upgrade safely is one per sprint per repo. The fastest you can do one without overlooking something is also about that fast. The discipline isn't about speed; it's about ensuring that no major version slides in unobserved.

## When the Strategist is tempted to "let me just run `npm install`"

Same shape as the other Strategist temptations: feels like a small operational step, has Executor-shaped side effects, breaks the model. An `npm install` that updates the lockfile is a code change. It belongs in a handoff and in the QA gate, not in a Strategist's chat session.

The pattern instead:

- The Strategist *reads* the lockfile to confirm what's installed.
- If something needs to be installed or upgraded, the Strategist writes a handoff that explicitly says so, with the version target and the rationale.
- The Executor runs the install, the type-check, and the QA gate, and ships the lockfile change as part of the handoff's commit.

The five-second test: *will this `install` change `package-lock.json`?* If yes, it's a handoff, not a quick fix.

## Worked example

A small shop runs four Next.js repos, each with `next`, `@vendor/auth`, `@vendor/db`, and `@vendor/sdk`. The Strategist is asked to add a new feature to repo A.

Without dependency discipline: the Strategist drafts a handoff that imports `@vendor/sdk@^4` because the v4 docs are what shows in chat. Repo A is on v3. The Executor type-checks, fails, kicks the handoff back. Round trip wasted, half a day lost.

With dependency discipline:

1. **Strategist reads the architecture graph** for repo A. Sees `@vendor/sdk` is at `3.2.1`. Notes that repos B and C are at `3.2.1`; repo D is at `4.0.5`.
2. **Strategist writes the handoff against the v3 API.** Notes at the top: "Repo A is on `@vendor/sdk@3.2.1`. Use the v3 API. Migration to v4 is tracked separately in the audit log."
3. **Executor implements.** Type-checks clean; ships.
4. **Audit pass on the first of the month.** `@vendor/sdk` flagged as diverged (3 repos on v3, 1 on v4). Disposition: *upgrade later, target Q3.* One-line entry in `dependency-audit.md` explaining why now isn't the right time.
5. **Q3.** Strategist reads the v4 release notes, designs a migration handoff per repo, sequences them (repo D's v4 is already in production and stable, so its handoff is shortest). The migration ships across three sprints, one repo at a time, with deltas logged. Divergence resolves to "all four on v4" by end of quarter.

The discipline traded a half-day round trip for a 10-minute lockfile read, and turned a six-month-deferred upgrade into a planned migration instead of an emergency.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Strategist references packages from memory of the vendor's docs | Memory drifts; the lockfile is fact | Read the lockfile or the architecture graph before writing the handoff |
| Treat `package.json` as the source of truth | Manifest is intent; lockfile is fact | The lockfile is the contract |
| Run `npm install` in a chat-based Strategist session | Side-effect change to the lockfile, no QA gate | Write a handoff that says "add this dependency"; let the Executor install |
| Upgrade dependencies opportunistically | Whatever happens to be at the top of the list, in whatever sprint has slack | A declared cadence per release-tier (patch / minor / major) |
| Auto-merge major upgrades | Breaking changes ship without anyone reading the release notes | Major upgrades are Tier 3 with a migration handoff |
| Let dependency versions diverge across repos without flagging | The Strategist will write to whichever version it last saw | Audit pass flags divergence; the audit log records why it's tolerated when it is |
| Treat `npm audit` warnings as noise | Critical vulns become exploits while you're not looking | Critical = Tier 1 fix; high = next month's upgrade cadence |
| Skip the lockfile read because "this is a small change" | Phantom-import bugs are usually the smallest-looking changes | Always read; reading is cheap |
| Bundle a dependency upgrade into an unrelated feature handoff | Two failures in one PR, harder to revert | Upgrade is its own handoff |

## Adapt it to your setup

- **Solo, one repo:** the audit is `npm outdated` plus `npm audit` once a month. The cadence is whatever you'll actually do — quarterly is fine for a stable solo project. The principle (read the lockfile, never `install` in a planning session) is identical.
- **Few repos:** the audit becomes a single command run across all repos, with output concatenated. Tools like `pnpm -r outdated` or shell loops do this in seconds.
- **Polyglot stack:** each language gets its own audit command and its own cadence. The discipline is identical across languages; only the tooling differs.
- **Monorepo:** the lockfile is one file but the audit is per-package. Major upgrades may have to migrate every package simultaneously; the migration handoff is correspondingly larger.
- **Compliance-bound:** the audit log feeds into the security review; vulnerable-dependency dispositions become evidence. Cadences may tighten (weekly audit, immediate patch on any critical).

## Related

- [The QA Gate](qa-gate.md) — where install/lockfile-change validations live in the per-change checklist
- [Architecture Graphs](architecture-graphs.md) — the per-repo package list and version data the Strategist reads first
- [Secrets and Rotation](secrets-and-rotation.md) — same shape: declared cadence, audit log, escalation discipline
- [Cost Management](cost-management.md) — the monthly review where dependency-audit findings get prioritized alongside spend
- Back to the [main playbook](../README.md)
