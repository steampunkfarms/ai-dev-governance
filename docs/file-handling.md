# File Handling

> The agent's working filesystem is ephemeral. *Yours* is permanent. Every rule below exists because that distinction was learned the hard way — usually by losing a session's work to a timeout, or by trusting a write tool that lied about which disk it had touched.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [Checkpoints & Handoffs](checkpoints-handoffs.md) · **See also:** [Foundation 4 in the architecture reference](architecture.md#foundation-4--the-file-handling-protocol)

---

## The wall

Most AI coding tools run in a container, a sandbox, or some other ephemeral environment. That environment has its own filesystem. Your repo lives on a *different* filesystem — your laptop, your server, your remote volume. The two look interchangeable from inside the agent until they aren't:

- A session times out mid-task. Anything written only to the agent's container vanishes when the container resets.
- A write tool returns *"success"* — but it wrote to the wrong filesystem, and you don't notice until the next session starts cold and can't find the file.
- A planning-shaped agent uses a write tool that *looks* generic (a `create_file`, a `save`, a `write`) but is actually a sandbox-scoped tool that never touches your disk.
- You batch ten file writes "for the end of the session," and the session dies on file seven. You lose seven, not three.

Each of these is the same root failure: **the agent and the operator do not share a filesystem by default; they share one only when the right tool is used and the result is verified.** The rules below close that gap.

## The rule

1. **Plant every file on the operator's filesystem, not the agent's container.**
2. **Verify the write by reading the file back.** A "success" response is not proof.
3. **Know which of your tools cross the filesystem boundary, and which don't.** Test them before you trust them.
4. **Write incrementally.** One file per completed step. No "save it all at the end."
5. **On any timeout or crash, verify on disk before resuming.** Don't trust the plan; trust the files.

## How it works

### Rule 1 — Plant on the operator's filesystem

Use the tool that writes to the persistent path that the operator owns — typically a workspace under their home directory, a mounted volume, or a synced cloud drive. **The destination path being correct is necessary but not sufficient** — the tool itself has to be the one that targets that filesystem. Passing a `/Users/.../Projects/myrepo/...` path to a sandbox-scoped tool can still land the file in the sandbox. The path argument doesn't constrain the destination; the *tool* does.

If you must stage a complex file in the agent's container first (large generation, intermediate processing), copy it to the operator's filesystem **immediately** after staging — not at session end.

### Rule 2 — Verify the write

After every write, do a small read of the same path to confirm:

- The file exists at the operator's path (not just inside the agent).
- The first few lines are what you expected.
- The file isn't truncated.

If the read fails (ENOENT, "no such file," empty response), the write went somewhere you didn't intend. Redo it via a known-good tool and re-verify. **Never trust a write tool's "succeeded" response on its own** — the success bit and the destination-correct bit are two different signals.

### Rule 3 — Audit your tool surface before you trust it

The single rule that turns this from theory into practice: **before relying on a write tool for repo work, prove out where it writes.** Do this once per tool, per agent, at the start of any new setup.

The test:

1. Pick a unique throwaway filename (e.g. `__filesystem_probe_<timestamp>.txt`).
2. Write a one-line marker (e.g. `"probe at 2026-05-30 21:25 PDT"`) to a path under your real workspace using the tool you're auditing.
3. **In a different surface** — your terminal, your editor's file browser, `ls` from your shell — open or list that file.
4. If it's not there, the tool is sandbox-scoped. Flag it as **never use for repo writes**. If it is there, the tool crosses the boundary and is safe to trust until proven otherwise.

The tools-to-suspect list (ones that have historically surprised people):

| Tool-name shape | Common surprise |
| :---- | :---- |
| Generic-sounding names (`create_file`, `save`, `write_to_file`) | Often sandbox-scoped even when the `path` argument looks like a real disk path |
| "Artifact" or "canvas" tools | Almost always sandbox-scoped — they're for display, not persistence |
| Anything that takes a path but never errors on permission | Suspicious; a real-disk write would fail noisily on a bad path |
| Anything that "uploads" a file | Often writes to a temp area you can't `cd` into |

Tools that *usually* cross the boundary, but still verify the first time:

- A filesystem-namespaced tool family (anything prefixed with `Filesystem:`, `Desktop`, `Workspace:`, or the agent's branded equivalent).
- A shell tool that runs `bash`/`zsh` against the operator's actual shell — verify it's not a containerized shell.
- An editor integration that writes through the IDE's filesystem (Cline, Cursor's edit tools, etc.).

**The transferable lesson isn't "always use tool X."** It's: *the tool name does not tell you where it writes. Test once, document the answer in your project's `AGENT.md`, and stop guessing.*

### Rule 4 — Write incrementally

For any task that produces more than one file:

- Write each file the moment its step completes.
- Verify it (rule 2) before moving to the next step.
- Update the checkpoint with that step marked done (see [Checkpoints & Handoffs](checkpoints-handoffs.md)).
- Then start the next step.

The pattern to avoid is "I'll generate all eight files, then write them at the end." If the session dies on file seven, you lose seven *and* the in-memory work for file eight. Incremental writes mean a crash at file seven costs you file seven, not the whole batch.

This also makes large files easier. A 500-line file written in one tool call often hits tool limits or timeout windows; the same file written as 20 chunks of 25 lines each rarely does.

### Rule 5 — Error recovery

When a session times out or crashes, the next session must **not** assume the previous one finished what its checkpoint said it would. The plan represents *intent*, the files represent *fact*. They diverge whenever a timeout lands between an intent and a file write.

The recovery protocol:

1. **Read the latest checkpoint.** See what the previous session *intended* to do.
2. **Verify each "done" step against the filesystem.** For every step the checkpoint marks complete, open the file(s) it claimed to write. Confirm they exist, are non-empty, and match what the step described.
3. **Clean up partial writes.** If a file is half-written, either complete it explicitly or delete it — don't leave the next step to discover it accidentally.
4. **Resume from the last confirmed-good state**, not from the last *claimed*-good state. If step 3 of 6 was marked done but the file isn't on disk, the resume point is step 3, not step 4.
5. **Write a new checkpoint** noting what you verified and where you actually resumed.

This rule is the bridge between [Checkpoints & Handoffs](checkpoints-handoffs.md) and the [Sanity Check](sanity-check.md): one tells you what to write down, the other tells you to validate against reality before trusting it. Error recovery is just the same discipline applied to your own previous session.

## When the Strategist is tempted to "just fix it quick"

This is the single behavioral pattern that breaks the model most often. The Strategist sees a one-line bug, knows the fix, and the path of least resistance is to edit the file. Don't.

Stop. Write the fix into a handoff document instead. The pattern:

```markdown
## Fix for Executor

In `src/lib/<area>/<file>.ts`, line 11:
- Remove: `import <thing> from "<package>";`
- The package is not in package.json. Replace the import-based call
  with a raw fetch() against <vendor>'s REST endpoint.

Full replacement code:
```ts
// ... 8 lines of replacement
```

## Acceptance
- [ ] `npx tsc --noEmit` clean
- [ ] No new dependencies added
```

What the Executor does with that handoff that the Strategist *can't* do directly: it runs the type-checker, reads the repo's `AGENT.md` to apply project conventions (the `.trim()` on env vars, the auth wrapper on cron routes, the commit-message format), commits with a clean message, and pushes. A "quick" Strategist edit skips all of that — and every single time it has been tried, the Executor has had to do a cleanup pass to put the conventions back. The "quick" path is slower.

**The five-second test:** if you're about to edit a file from a chat-only or planning-only context, ask *"will this run through type-check and the project's conventions before it commits?"* If the answer is no, it's a handoff, not an edit.

## Worked example

A Strategist session is generating eight new docs for a project. The naive path: hold them all in conversation context, then write them in a burst at the end.

What goes wrong: session times out after doc 5 is fully generated but only doc 3 has been written to disk. Docs 4 and 5 exist only in the chat scrollback; the operator has to copy-paste them out by hand. Docs 6–8 are unstarted and would have been finished if the writes hadn't been batched.

The disciplined path:

1. Generate doc 1, write it to `<workspace>/docs/doc-1.md`, read the first 5 lines back to verify, update the checkpoint.
2. Generate doc 2, write, verify, update checkpoint.
3. … (repeat for each.)

When the session times out after doc 5, the next session reads the checkpoint, sees docs 1–5 on disk and verified, and starts at doc 6. No copy-paste, no rework, no lost content. The only "extra" work was reading five small files back — total cost, maybe 10 seconds. The savings, on a real timeout, is hours.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Treat the agent's container as the primary filesystem | Resets on timeout; work vanishes silently | Write to the operator's filesystem; verify by read |
| Trust a write tool's "success" response | It can mean "wrote to the sandbox successfully" | Always read the file back from the intended path |
| Use a write tool you haven't audited | The tool name doesn't tell you where it writes | Run the probe-file test once before trusting a tool |
| Pass the right path to the wrong tool | The path argument doesn't constrain the destination — the tool does | Match tool to filesystem, not path to filesystem |
| Batch all writes for end of session | A timeout means total loss | Write each file as its step completes |
| Single 500-line file write | Tool limits + longer window for a timeout | Chunk large writes; prefer many small files |
| Resume a crashed session from the plan | The plan is intent; the files are fact | Verify each "done" step against the disk first |
| Strategist edits source to save a round trip | Bypasses type-check, conventions, commit log | Write a handoff for the Executor — every time |
| Strategist edits env/config "just this once" | Misnamed vars, missed `.trim()`, secrets in chat history | List the vars in a handoff; let the Executor apply |

## Adapt it to your setup

- **Solo, single tool:** rules 1, 2, 4, 5 still apply — they protect you from your own tool's container behavior. The Strategist/Executor handoff in the last section becomes a planning-pass → execution-pass discipline within the same tool.
- **Cloud-only agents (no local filesystem):** the operator's filesystem might be a remote workspace, a synced cloud folder, or a mounted volume. The rule is unchanged: figure out which path your agent's tools persist to, write there, and verify.
- **Sandboxed agents by design (e.g. browser-based code interpreters):** stage in the sandbox, but copy out to a persistent destination — a download, a sync, a commit — at every checkpoint, not just at session end.
- **New agent or new tool:** run the probe-file test once and add a note to your project's `AGENT.md` saying which tools cross the boundary and which don't. This is a one-time cost that pays back the first time you'd otherwise have trusted a sandbox-scoped tool.

## Related

- [Checkpoints & Handoffs](checkpoints-handoffs.md) — the artifacts that make incremental writing useful
- [The Two Roles](two-roles.md) — why the Strategist's "quick fix" is structurally wrong
- [The Sanity Check](sanity-check.md) — the same disciplines (verify against reality, don't trust intent) applied to production state
- [Architecture reference: Foundation 4](architecture.md#foundation-4--the-file-handling-protocol) — where this protocol sits in the larger model
- Back to the [main playbook](../README.md)
