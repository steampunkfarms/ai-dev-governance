# Checkpoint: H-PUB-001 — Public governance repo setup
**Date:** 2026-05-24
**Status:** BLOCKED — Sanity Check mismatches; halted per handoff protocol.

## Sanity Delta — what the handoff assumes vs. what production shows

The handoff is explicit: "If ANY condition differs, STOP, write a checkpoint
describing the mismatch, and do nothing else." Multiple conditions differ.

### Condition-by-condition

1. **`git remote -v` + branch `main`** — MATCHES. Remote is
   `https://github.com/steampunkfarms/ai-dev-governance.git`, branch `main`.
   *Caveat:* there was no local clone when the handoff was invoked. I cloned
   to `/Users/ericktronboll/Projects/ai-dev-governance/` as a precursor
   (minimal, reversible action). Logging here for transparency.

2. **Working tree contains the dropped-in files; no other unexpected uncommitted
   changes** — DIFFERS. The six files are at
   `/Users/ericktronboll/Downloads/files 2/`, not in the repo working tree. The
   operator prerequisite ("drop them into the local repo clone") was not done.

3. **Existing `README.md` is the earlier placeholder/v1 — will be overwritten**
   — DIFFERS. The committed `README.md` is **byte-identical** to the new
   `/Users/ericktronboll/Downloads/files 2/README.md` (`diff` exit 0,
   25,250 bytes). It was already updated in commits `6173814` / `777bba2`. The
   handoff would overwrite it with the same bytes — a no-op, but the premise
   is wrong.

4. **All 6 files present at listed paths** — DIFFERS.
   - `README.md` at root: present (identical to new — see #3).
   - `CONTRIBUTING.md` at root: **missing**.
   - `docs/sanity-check.md`: **wrong path.** A `sanity-check.md` is committed
     at the repo **root**, not under `docs/`. Byte-identical to the new
     `docs/sanity-check.md` (commits `d6f1263` create, then a delete and
     recreate, merged via PR #1 = `289231a`).
   - `docs/bounded-deviation.md`: **missing**.
   - `docs/_deep-dive-template.md`: **missing**.
   - The `docs/` directory itself did not exist until this checkpoint was
     written.

5. **No `LICENSE` file exists yet** — MATCHES.

### Why production diverged from the spec

Recent commits (PR #1, `289231a`) shipped a partial setup: the final `README.md`
and `sanity-check.md` were committed, but `sanity-check.md` landed at the repo
root instead of `docs/sanity-check.md`. CONTRIBUTING, bounded-deviation, the
deep-dive template, and the `assets/` scaffolding never landed.

The new README.md links to `docs/sanity-check.md` and `docs/bounded-deviation.md`.
Those links are currently dangling on the live public page — the repo is in a
"published but broken-links" state.

## Risk if the handoff were followed as written

- Step 2 ("the final README overwrites the placeholder") would be a no-op
  rewrite — harmless but misleading.
- Step 2 placement of `docs/sanity-check.md` would create a *second* copy at
  the correct path while leaving the stale root copy committed — divergence.
- The root `sanity-check.md` is reachable via the existing repo structure;
  blindly placing a new `docs/sanity-check.md` without removing the root copy
  would leave two files that could drift.

## Adjusted plan I recommend (operator approval needed before executing)

Same end-state as the handoff; smaller, safer diff:

1. `git mv sanity-check.md docs/sanity-check.md` (preserves history; removes
   the dangling-link root copy).
2. Copy `CONTRIBUTING.md`, `docs/bounded-deviation.md`,
   `docs/_deep-dive-template.md` into place from
   `/Users/ericktronboll/Downloads/files 2/`.
3. Create `assets/.gitkeep`.
4. Comment out the hero `<img>` block in `README.md` (Step 3 of the handoff).
   Currently uncommented; live page may show a broken image.
5. Verify internal links resolve, then commit + push (one commit:
   `docs: complete public-repo setup — move sanity-check, add deep-dive docs and CONTRIBUTING`).
6. Update this checkpoint to COMPLETE with shipped summary.

This is bounded deviation: file-anchored evidence (commit hashes above,
`diff` exit 0); minimal and risk-reducing (uses `git mv` instead of duplicating;
no rewrite of identical bytes); no scope expansion (same final tree).

## Context for next session

- Repo cloned at `/Users/ericktronboll/Projects/ai-dev-governance/` on `main`,
  clean working tree (except this checkpoint).
- Source files for the remaining additions are at
  `/Users/ericktronboll/Downloads/files 2/`.
- Pre-authorization in the handoff is conditional on all checks matching;
  multiple do not, so the pre-auth does **not** apply. Operator approval of
  the adjusted plan is needed before proceeding.

## Files modified this session

- `docs/checkpoints/2026-05-24-public-repo-setup.md` — this file (only change).
- (Cloned the repo locally — no commits, no remote changes.)
