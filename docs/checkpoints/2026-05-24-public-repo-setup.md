# Checkpoint: H-PUB-001 — Public governance repo setup

**Date:** 2026-05-24
**Status:** COMPLETE — adjusted plan executed after operator approval of the Sanity Delta below.
**Shipped commit:** `711701b` on `main` (pushed).

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

Initial halt-checkpoint commit (this file alone, status BLOCKED), then —
after operator approval of the adjusted plan — commit `711701b` shipped:

- `sanity-check.md` → `docs/sanity-check.md` (rename via `git mv`, history preserved).
- `CONTRIBUTING.md` (new, root).
- `docs/bounded-deviation.md` (new).
- `docs/_deep-dive-template.md` (new).
- `assets/hero.png` (new — operator dropped at `~/Downloads/hero-image.png`).
- `assets/.gitkeep` (new — kept literally per handoff Step 1, redundant once
  `hero.png` is present but harmless).
- `docs/checkpoints/2026-05-24-public-repo-setup.md` (this file).

## Acceptance criteria — final

- [x] Final `README.md` at repo root.
- [—] Hero `<img>` block commented out → **deviated.** Operator delivered
  `hero-image.png` mid-session, so the asset exists and the block is left
  uncommented. Operator-approved.
- [x] `CONTRIBUTING.md` at root.
- [x] `docs/` contains `sanity-check.md`, `bounded-deviation.md`,
  `_deep-dive-template.md`.
- [x] `assets/.gitkeep` present (and `assets/hero.png`).
- [x] No dangling internal links (verified by grep on real markdown links;
  `docs/roadmap.md` in README is inline-code descriptive text, not a link).
- [x] Clean commit pushed to `main`; working tree clean.
- [x] Final checkpoint written (this file).

## Operator Action Block (Erick — still required, NOT delegated)

1. **Add a `LICENSE`** on GitHub web UI. Match the README badge (CC BY 4.0)
   or update the badge.
2. **Set repo About + topics** on GitHub web UI per handoff text.
3. **Hero banner** — already installed by this session. No further action
   unless you want to replace it.
4. **(Optional)** Enable Discussions.

## Bounded-deviation log (post-execution)

Two deviations from the handoff-as-written, both operator-approved and
within the three-prong test:

1. **Cloned the repo before running** — handoff assumed a pre-existing local
   clone. Evidence: `gh repo view` confirmed remote; no local clone existed.
   Minimal (single `gh repo clone`), risk-reducing (alternative was a hard
   STOP with operator absent at first). No scope change.
2. **Used `git mv` instead of placing a fresh file** — handoff Step 2 said
   "place the files." Production had `sanity-check.md` at root from PR #1.
   `git mv` preserves history and eliminates the duplicate that "place"
   would have left behind. Evidence: `diff` exit 0 vs the new file; commit
   `289231a` log. Minimal, risk-reducing, no scope change.
3. **Skipped the "comment out hero `<img>`" step** — handoff Step 3 was
   conditional on `assets/hero.png` not existing. Operator delivered the
   asset mid-session; condition no longer held.
