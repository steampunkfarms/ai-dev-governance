# Secrets and Rotation

> Two AIs and a human share a codebase. **None of them should ever see a production secret in plaintext.** The discipline below is what keeps that true — not just on day one, when everyone is careful, but two years in, when the operator has rotated 40 keys and onboarded three new repos.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md), [The QA Gate](qa-gate.md) · **See also:** [File Handling](file-handling.md)

---

## The wall

Secrets leak in five ways, in roughly this order of frequency:

1. **Pasted into chat.** The Strategist asks "what's my Stripe key?" while debugging a webhook. The operator pastes it. The chat log now lives in the AI provider's system, the operator's local cache, and possibly a shared session transcript.
2. **Committed to a repo.** A developer adds a "temporary" `.env` to a commit, or an Executor writes a config file with a value the operator pasted earlier in the session. `git push --force-with-lease` does not delete the value from GitHub's history.
3. **Logged.** A debug `console.log(process.env)` ships to production. The value appears in the host's log aggregator, viewable by anyone with access.
4. **Echoed into a handoff.** The Strategist writes a handoff that says "set `STRIPE_SECRET_KEY=sk_live_…`" with the actual value. The handoff goes into the governance repo, which the Executor reads from, which is checked into git.
5. **Forgotten on rotation.** A key is rotated in the hosting console but not in the local `.env.local`, or vice versa, and the discrepancy is only discovered when something silently fails.

Each one is recoverable individually. The compound problem is that AI workflows multiply every leak surface: more chat transcripts, more handoff documents, more agent containers with their own filesystems, more places a value can end up resident in memory or on disk. The discipline below is shaped specifically for that environment — it assumes two AIs and a human, and it puts every secret behind a discipline that survives the *next* leak vector you haven't thought of yet.

## The rule

1. **Secret values never appear in chat, handoffs, checkpoints, or any file in any repo.** Names appear; values do not.
2. **The operator is the only role that pastes a secret into anything.** Not the Strategist, not the Executor.
3. **Secrets live in exactly one place per environment** — the hosting console for production, a single local file (`.env.local` or equivalent) for development — and are referenced everywhere else by name.
4. **A rotation playbook exists for every secret, lives in the repo's `AGENT.md`, and is exercised at least once before the secret is needed in anger.**
5. **A leak protocol exists for every secret category** — what to rotate, who to notify, how to confirm the leak window is closed — and is rehearsed on adoption, not on the incident.

## How it works

### Rule 1 — Names, not values

Every reference to a secret across the playbook uses the *name* — the env-var identifier — not the value. The Strategist writes a handoff that says:

> Set `STRIPE_SECRET_KEY` in the hosting console for this project. The value is in the operator's `1Password → Stripe → Live mode → Restricted key`. Do not paste the value into this handoff or into chat.

That's it. The Executor reads the name, the operator pastes the value into the hosting console, and the value is never resident in any AI's context, any chat transcript, or any committed file.

The same rule applies to `AGENT.md` files, checkpoints, the Delta Log, and every other artifact the playbook creates. If you find yourself about to type a value, stop and re-route through the operator.

### Rule 2 — The operator's monopoly on paste

Only the operator pastes secret values. In practice this means three workflows:

- **Local development.** The operator maintains `.env.local` (or equivalent) by hand, by copying values out of a password manager into a file the AI never reads. The Executor reads the env at runtime; it does not edit the file.
- **Production / hosting console.** The operator sets values in the hosting provider's UI (Vercel, Cloudflare, Render, etc.). The AI cannot reach these consoles directly — and if you've wired an AI integration that could, treat that integration itself as a security-sensitive surface and audit it under the same discipline.
- **Shared services (databases, third-party APIs).** Same pattern. The operator manages credentials inside the third-party's UI; the AI references them by env-var name.

This monopoly is the single most load-bearing rule in the file. Every other rule is enforceable mechanically; this one is enforceable only by discipline, and the discipline is easy to drop in a hurry. The five-second test: *if I'm about to type a value, am I the operator or an AI?* If AI, stop.

### Rule 3 — One source of truth per environment

Per environment, the values live in exactly one place:

| Environment | Source of truth | Read by |
| :---- | :---- | :---- |
| Production / staging | Hosting provider's env-var UI | The deployed app at runtime |
| Local development | A single ignored file (`.env.local`) at the repo root | The local dev server |
| Shared / cross-repo | A secrets manager (1Password, AWS Secrets Manager, Doppler, Infisical) | The operator, manually copying out |

Anything else — a `.env.example` checked into the repo, a `config.local.ts`, an init script — contains *names only* (and optionally placeholder values like `STRIPE_SECRET_KEY=sk_test_REPLACE_ME`). The repo never contains a real value, in any branch, in any commit, ever.

A short script (`scripts/check-secrets.sh` or similar) runs in CI on every PR and greps the diff for high-entropy strings and known prefixes (`sk_live_`, `whsec_`, `xoxb-`, AWS access key shapes, `pk_live_`, `eyJ` JWT shapes, etc.). The script fails the PR if it finds one. This is the mechanical backstop for rule 2.

### Rule 4 — Rotation, in advance

Every secret has a documented rotation procedure that lives in the repo's `AGENT.md`. Minimum content:

- **Where to generate the new value** — vendor console URL, role/permission to use.
- **Where to set the new value** — hosting consoles for every environment, secrets manager.
- **What to test after rotation** — a smoke test that confirms the app works with the new value.
- **What to revoke** — the old value's location in the vendor's console, and the timing rule for revocation (e.g. "wait 24h after the new value is live, then revoke the old one").

The procedure is rehearsed *before* it's needed in anger. Pick a non-critical secret, rotate it on a calm afternoon, walk the procedure, fix anything that didn't work, update the `AGENT.md`. The rehearsal is the only way to discover that step 4 actually requires admin access the operator doesn't have, or that the smoke test takes 20 minutes when you assumed it took 2.

Rotation cadence varies by category. A reasonable starting point:

- **Long-lived API keys (Stripe, Twilio, SendGrid):** rotate annually, or immediately on any suspected exposure.
- **OAuth client secrets:** rotate annually.
- **Service-account credentials:** rotate semi-annually, or as the platform requires.
- **Database passwords:** rotate semi-annually, or as the platform requires.
- **Internal HMAC / signing secrets:** rotate annually, unless the algorithm is broken and forces sooner.

Rotation events go in the changelog of the repo that uses the secret, with the rotation date and the new key's last-four (for identification, not for use).

### Rule 5 — A leak protocol per category

A leak is when a secret value lands somewhere it shouldn't — a chat transcript, a public log, an accidental commit, an over-shared screenshot. When a leak happens, the playbook is:

1. **Revoke immediately.** Not "after we figure out scope." The first action is to make the leaked value useless. Revoking takes seconds; reasoning about blast radius can take hours.
2. **Generate the replacement.** Follow the rotation procedure from rule 4.
3. **Inventory the leak surface.** Where did the value end up? Chat? Git? A log file? Each surface is a separate cleanup task.
   - Chat: clear the conversation if the platform allows; otherwise note that the value was once visible there.
   - Git: rewrite history if the value is in a few commits and the repo is small; otherwise rotate and accept the historical exposure (the revocation makes it harmless).
   - Logs: scrub if the platform allows; otherwise note in the incident.
4. **Document the incident.** A checkpoint at `docs/checkpoints/<DATE>-leak-<slug>.md` with the trigger, the value's location-history, the rotation timeline, and one line for the Strategist on whether the leak path should become a new prohibition.
5. **If the leak is third-party-scoped (a vendor key)** — notify the vendor's security team. Most vendors care more than you'd expect and will sometimes accelerate the rotation on their end.

The principle: **the cost of an unnecessary rotation is small; the cost of a delayed rotation is unbounded.** When in doubt, rotate.

## When the Strategist needs to "test" something

This is the equivalent of the [Strategist's "just fix it quick" temptation in file handling](file-handling.md#when-the-strategist-is-tempted-to-just-fix-it-quick). The Strategist is debugging a webhook signature mismatch and asks the operator for the signing secret so it can compute a hash by hand. Don't.

The pattern instead:

- The Strategist writes a handoff that asks the **Executor** to compute the hash inside the repo, with the value pulled from env at runtime.
- The Executor runs the test, returns the result (the *result*, not the secret), and the Strategist reasons forward from the result.
- The operator confirms the result matches what the vendor's tooling produces.

This adds a 5-minute round trip and removes a vector that has bitten real teams. The five-second test: *would I be comfortable if this secret value ended up in tomorrow's chat log?* If no, don't put it there today.

## Worked example

A small team adds support for sending transactional email via Resend. The naive path: get the API key, paste it into a chat to ask why the SDK is failing, paste it into a `.env` that gets accidentally committed, force-push to remove it, congratulate yourself.

The disciplined path:

1. **Setup.** Operator generates a Resend API key in the Resend console; stores it in 1Password under "Resend → Production." Adds `RESEND_API_KEY` to the Vercel project's env vars (production + preview). Adds `RESEND_API_KEY=re_test_REPLACE_ME` to `.env.example` and commits that. Adds `RESEND_API_KEY=<real test key>` to local `.env.local`, which is gitignored.
2. **`AGENT.md` entry.** A four-line rotation procedure: where to generate, where to set, smoke test (`curl -X POST https://api.resend.com/emails --auth …` → expect 200 with test-mode response), revocation timing.
3. **Strategist handoff.** Writes "the handler reads `RESEND_API_KEY` from env." Does not include the value. The handoff is committed to the governance repo without risk.
4. **Executor implements.** Reads env at runtime, ships, the deploy works.
5. **Six months later, rotation.** Operator generates a new key on a Tuesday afternoon. Walks the rotation procedure from `AGENT.md`: new value in Vercel for production *and* preview, smoke test passes, old key revoked 24h later. Two-minute log line in the changelog.
6. **A year in, a leak.** A debug log inadvertently echoed the env to the host's log aggregator for ten minutes. Operator revokes immediately, rotates, scrubs the logs, writes a checkpoint at `docs/checkpoints/2027-04-12-leak-resend.md`. The Strategist updates `AGENT.md` to add a one-line prohibition on `console.log(process.env)` in handlers, and the post-deploy QA grep is updated to flag it. Total time from discovery to resolution: 15 minutes, of which 10 were waiting for the new key to propagate.

The discipline traded a 30-minute setup for a leak resolution that took 15 minutes instead of half a day.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Paste a secret into chat "just for debugging" | The transcript is durable; the value is now in the AI provider's system | Reference by name; have the Executor compute the test |
| Put real values in `.env.example` | `.env.example` is committed; the value is now public | Names + placeholder values only (`sk_test_REPLACE_ME`) |
| Hand-edit the hosting console's env every time | Drift between environments; no rotation log | A rotation playbook in `AGENT.md`; rehearse it |
| Commit `.env.local`, then `git revert` | `git revert` doesn't remove from history; rotation is required regardless | Gitignore from the start; never commit |
| "We'll rotate after the launch" | Launch never produces a calm moment; the rotation gets indefinitely deferred | Rotate on a fixed cadence regardless of launch state |
| Strategist writes "set `STRIPE_SECRET_KEY=sk_live_…`" in a handoff | Value is now in the governance repo, even after edit | Handoff says only the name and points to the operator's password manager |
| Log `process.env` for debugging | Production logs are searchable by anyone with access | Log only the values you need, by name, redacted by length |
| Revoke "once we understand the scope" | Scope-reasoning takes hours; revocation takes seconds | Revoke first, then investigate |
| One shared API key used by every repo | A single leak forces a rotation that breaks every repo simultaneously | One key per repo or per service boundary; rotate independently |

## Adapt it to your setup

- **Solo, no secrets manager:** a single passworded `secrets.md` file in an iCloud-synced folder or a password manager's secure note is enough. The principle (names in code, values in operator-only storage) is identical.
- **Team:** add a secrets manager (1Password Shared Vault, Doppler, Infisical, AWS Secrets Manager). The operator-paste discipline is enforced by team norm, backed by the CI grep for known prefixes.
- **Compliance-bound (HIPAA, SOC 2, PCI):** rotation cadences shorten, leak protocols become legally significant, and the audit trail in the changelog becomes the basis for an evidence record. The shape of the discipline is identical; the cadences and documentation tighten.
- **Heavy third-party integration (many vendors):** the rotation playbook becomes a per-vendor table in the repo's `AGENT.md`, not a paragraph. Generate-test-revoke steps are identical per vendor; the URLs differ.

## Related

- [The Two Roles](two-roles.md) — defines the operator's monopoly on pasting values
- [File Handling](file-handling.md) — the same "verify before trusting" discipline applied to writes
- [The QA Gate](qa-gate.md) — where the post-deploy secret scan lives
- [Autonomous Loops](autonomous-loops.md) — loops are a high-leak surface because they read secrets unattended
- Back to the [main playbook](../README.md)
