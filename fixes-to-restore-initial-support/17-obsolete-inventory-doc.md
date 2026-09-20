# Step 17 — Write the obsolete-feature inventory

**Lane E · Depends on: nothing (finish after 15 so deletions can be recorded) · Owns: `../obsolete-inventory.md`**

Read `00-README.md` first. This step writes documentation only. No code.

## Goal

Produce one document that tells branch 2 what to remove, what to keep, and what it would cost either way — so that branch starts from a decision list instead of a research project.

Write it to `../obsolete-inventory.md` (repo root).

## Sources — read, do not redo

The research is done. Pulling from these is the job; re-deriving verdicts is not.

- `../uat/00-setup.md` §10 — the roll-up table, the corrections to earlier assumptions, and nine cross-cutting findings
- `../uat/NN-*.md`, each file's `## Patch-note correlation (2023–2026)` section — per-feature evidence with build numbers and dates
- `../uat/evidence/02/OBSOLETE.md` and `../uat/evidence/09/OBSOLETE.md` — the tester's own calls. **Both overreach**, and the document must say so: 02 claims TileMaker has no remaining callers (the legacy conversation portrait is still a real gap), and 09 recommends removing the inventory screen outright (no native screen offers value-per-weight).
- `../1.0.5-agent-results.md` — what the compile-fix pass changed
- Step 15's hand-back — paths and SHAs of deleted files

## Required contents

**One row per feature** with these columns:

| Feature | Files | Status | Superseded by (build + date) | Value under legacy UI | Branch 2 recommendation |

`Status` is one of `dead` (gone already, with the recovery SHA), `obsolete under modern UI`, or `keep`.

Cover every feature in the §10 roll-up: conversation helper, TileMaker split by caller, quick pickup, cooking menu, restock timer, sprite menu, quest-giver locator, scoreboard, inventory screen, auto-pickup exclusions, legendary journal, inventory filter.

**Then these sections:**

1. **Deleted in branch 1** — what step 15 removed, with the SHA each can be recovered from. The conversation-portrait transpiler matters most: branch 3 may want it back.
2. **Known gaps, not bugs** — features that never existed rather than broke. Quick pickup cannot see inside containers or chests (it only reads cell objects). The sprite menu has no modern-UI equivalent. The `MakeSecretId` collision for two same-named legendaries in one zone. Legendary marking is one-way, not a toggle.
3. **Engine-level defects the mod cannot fix** — the input-starvation freeze (vanilla legacy screens have it too) and `Popup.ShowYesNo` returning its default unrendered under the modern UI. Say what branch 1 did to blunt each, and what remains.
4. **Corrections to earlier assumptions** — carry over the three from §10 (native autoget is *not* a replacement for per-item exclusions; TileMaker is not caller-less; quick pickup did not break because of the UI overhaul), plus the fourth this planning pass produced: **the quest-giver locator's Harmony patch does bind**, so the "wrong signature" explanation was wrong, and the real cause is whatever step 07 finds.
5. **Open questions** — anything steps 07, 11 or 13 ended without proving.

## Style

Written for someone deciding what to delete, not for someone who already knows the codebase. Name files by path. Every obsolescence claim carries a build number and date, or is marked `⚠️ unverified`. Blunt about uncertainty: "not reproduced" and "cause unknown" are acceptable entries and better than a confident guess.

## Non-goals

- No new research. If the sources disagree, report the disagreement.
- No code changes.
- Do not design branch 2's removals beyond a recommendation per row.
- Do not speculate about branch 3.

## Acceptance

- The file exists at the path above and covers every feature in the §10 roll-up.
- Every `dead` row has a recovery SHA.
- The two overreaching `OBSOLETE.md` claims are explicitly corrected.
- Someone who has not followed this project can read it and know what branch 2 should do.
