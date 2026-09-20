# UAT 00: Shared setup (read first)

> Mod: QudUX v2 on Caves of Qud 1.0.5 · Branch `fix/add-support-for-1.0.5` (staged, not committed)
> Test character: **`qudux-32-test-uat-character`**
> Scope: **functional** testing of the 12 fixes. If something fails, write it up in that script's "Problems found" section; it gets reviewed afterwards. If everything passes, we move on to code review.

## 1. Install the build under test

1. From the repo root, run `./move-mod.ps1`. This copies the mod to `%USERPROFILE%/AppData/LocalLow/Freehold Games/CavesOfQud/Mods/QudUX_v2`.
2. Launch the game, open **Mods**, and confirm QudUX is enabled with no red or error state.
3. 📸 `UAT-00-01-mod-list.png`: mod list showing QudUX enabled.

## 2. Clean log baseline

- The game log is at `%USERPROFILE%/AppData/LocalLow/Freehold Games/CavesOfQud/Player.log`. The previous session's log is `Player-prev.log`.
- The log is overwritten on every launch. **Before closing the game after a test session, copy `Player.log` to `uat/evidence/<NN>/Player-<NN>.log`.**
- Quick check (PowerShell):
  ```powershell
  Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "Exception|QudUX" | Select-Object -First 50
  ```
  Any `Exception` whose stack trace mentions `QudUX` counts as a **FAIL** for the script you were running.

## 3. Keys used across scripts (verified in the game's `Base/Commands.xml`)

| Action | Key |
|---|---|
| Wish prompt | `Ctrl+W` |
| Wish menu | `Shift+W` |
| Save game | `F5` |
| Load game | `F9` |
| System menu | `Esc` |

Wish syntax for spawning an object: open the wish prompt and type the blueprint ID (each script lists the exact IDs it needs).

## 4. Save discipline

- Load `qudux-32-test-uat-character`, then press `F5` to save a **clean baseline** before starting.
- Order: run **08 (scoreboard) last**, because its death step ends the character.
- Several scripts change options (09 changes the UI mode; 10 changes a global "shown once" flag). Put the options back as each script says before starting the next one.

## 5. Evidence

- Folder layout: `qudux-mod-docs/uat/evidence/<NN>/`
- File names: `UAT-<NN>-<step>-<short>.png|.mp4`, exactly as each script specifies.
- Screenshots: any tool works (Win+Shift+S, Steam F12, etc.) as long as the relevant UI is readable.
- Video: record with Win+Alt+R (Xbox Game Bar) or OBS. **Include game audio** where a script asks for it (#10).

## 6. Suggested run order

| Order | Script | Notes |
|---|---|---|
| 1 | [01](01-conversation-helper-loads.md) | conversation smoke test |
| 2 | [05](05-merchant-restock-timer.md) | merchant restock |
| 3 | [07](07-quest-giver-locator.md) | quest giver locator |
| 4 | [09](09-inventory-screen-ui-mode-matrix.md) | UI-mode matrix; decides which inventory shows |
| 5 | [12](12-inventory-filter-prompt.md) | needs the QudUX inventory active (see 09) |
| 6 | [02](02-tilemaker-render.md) | tile rendering |
| 7 | [03](03-quick-pickup-multiselect.md) | multi-pickup |
| 8 | [10](10-autoget-exclusion-confirm.md) | auto-pickup exclusion; changes a global flag |
| 9 | [04](04-recipe-ingredient-quantities.md) | cooking |
| 10 | [06](06-character-tile-search-filter.md) | sprite / tile screen |
| 11 | [11](11-legendary-creature-factions.md) | legendary creatures |
| 12 | [08](08-enhanced-scoreboard.md) | **last**, needs a death |

## 7. Overall results

| # | Script | PASS / FAIL | Problems filed? |
|---|---|---|---|
| 01 | Conversation helper loads | | |
| 02 | TileMaker render | | |
| 03 | Quick pickup multi-select | | |
| 04 | Recipe ingredient quantities | | |
| 05 | Merchant restock timer | | |
| 06 | Character tile search filter | | |
| 07 | Quest giver locator | | |
| 08 | Enhanced scoreboard | | |
| 09 | Inventory UI-mode matrix | | |
| 10 | Auto-pickup exclusion confirm | | |
| 11 | Legendary creature factions | | |
| 12 | Inventory filter prompt | | |

## 8. Items to confirm in game (`⚠️ unverified` in the scripts)

The script agents couldn't confirm these from source or decompiled code. They are not failures in themselves. Just note in the results what you actually see.

| Script | Item |
|---|---|
| 02 | Optional "flying" stretch step. It uses a Gyrocopter Backpack stand-in, because no stock path was found for the case the code comment mentions. |
| 05 | Exact dialog wording of "Wait a number of turns" (Ctrl+`) |
| 07 | Whether Joppa already has a quest-giving villager; the script has a fallback |
| 08 | The "Killed by" text for a wished death, and how many death/epilogue screens appear before the main menu |
| 09 | Row B (Modern character sheet ON + Modern UI OFF): whether the hidden character-sheet setting keeps its value after Modern UI is turned off |
| 11 | Step 7: whether wish-spawned heroes count as zone "points of interest" for the batch journal command |
| 12 | Whether the mouse filter command (CmdFilter) has a clickable element in the ASCII inventory (optional step) |

## 9. Gotchas found while writing the scripts

- **`move-mod.ps1` wipes the auto-pickup exclusions.** QudUX stores them, plus the "shown once" popup flag, in `<mod folder>/QudUX_AutogetSettings.json`, and the script deletes and recreates the mod folder. Don't reinstall in the middle of script 10.
- **Tile graphics must be on** (Options → Display → "Enable tile graphics") for 02. Otherwise TileMaker does nothing.
- Some of these were found incidentally and are **not in scope for UAT**. They are parked for code review, so don't file them as UAT failures:
  - The QudUX sprite menu in character creation is dead code: its Harmony patch is fully commented out. Script 06 tests it through the `sprite menu` wish instead.
  - Quick pickup silently drops shields and tools from its list. This predates the 1.0.5 branch, and script 03 avoids those items.
  - The cooking screen's multi-serving branch can't be reached through normal play (every recipe uses 1 of each ingredient). Script 04 covers only the reachable branch.
  - The enhanced scoreboard may crash on Enter when the Games list is empty. Script 08 avoids doing that.

## 10. Patch-note research roll-up (2026-09-20)

One agent per topic cross-checked the UAT findings against Caves of Qud patch notes for 2023–2026 (wiki `Version_history/2023`, `/2024`, `/2025`, plus the 2026 builds 211.33–212.17 on the index page) and the decompiled 1.0.5 assembly. Full evidence is in each script's `## Patch-note correlation (2023–2026)` section.

| # | Feature | Verdict | Action |
|---|---|---|---|
| 01 | Conversation helper | STILL VALUABLE | Keep, no UI gate. The "only after trading" behavior is the mod's own gate, not a regression. |
| 02 | TileMaker | PARTLY OBSOLETE | Split by caller: inventory sprites already gated correctly; conversation portrait still fills a real gap in the legacy UI. |
| 03 | Quick pickup | STILL VALUABLE | No native multi-select exists. Broken cases are not UI-related — don't gate. |
| 04 | Cooking menu | PARTLY OBSOLETE | Vanilla already shows the same ingredient counts. Gate the campfire patch behind `!ModernUI`. |
| 05 | Restock timer | STILL VALUABLE | Keep as is. No native equivalent; UI-independent. |
| 06 | Sprite menu | STILL VALUABLE | Keep. No native sprite picker. Needs freeze mitigation. |
| 07 | Quest giver locator | STILL VALUABLE | Fix — it's a mod regression, not a game removal. |
| 08 | Enhanced scoreboard | STILL VALUABLE | Keep active. No native aggregate stats; the freeze is a vanilla defect. |
| 09 | Inventory screen | PARTLY OBSOLETE | Keep for legacy UI. Value-per-weight exists in neither native screen. |
| 10 | Auto-pickup exclusions | STILL VALUABLE | Keep all of it. The native system has no per-item exclusion. |
| 11 | Legendary journal | STILL VALUABLE | Repair. No native legendary tracking. |
| 12 | Inventory filter | PARTLY OBSOLETE | Keep for legacy UI; it's a near-verbatim fork of the vanilla legacy filter. |

### Corrections to earlier assumptions

- **"Native autoget makes the mod's version redundant" — no.** The base game only has global category checkboxes and an author-only blueprint tag. Per-item exclusion, the interaction-menu action, and the management screen have no native counterpart. Nothing to drop for redundancy.
- **`evidence/02/OBSOLETE.md` "TileMaker has no remaining callers" — too broad.** The legacy conversation path computes a portrait icon and discards it, so the gap the mod filled is still real there. Both build claims in `evidence/09/OBSOLETE.md` (207.31, 207.69) are correct.
- **"Quick pickup broke because of the UI overhaul" — probably not.** Every API it uses is legacy engine code untouched by the modern-UI rewrite. Gating it behind a UI flag would not fix it and would cost legacy users the feature.

### Cross-cutting findings for code review

1. **The stuck-screen freeze (06 and 08) is one shared vanilla defect, confirmed.** Legacy fullscreen screens block on a key queue that the game stops feeding while any modern-UI window or popup is visible, so the wait never ends. Vanilla's own high-score screen has it too. Mitigation is defensive polling in the mod's wait loops, not UI gating. Around 7 other legacy screens are likely exposed.
2. **`Popup.ShowYesNo` is fire-and-forget under the modern UI.** It returns the default result immediately, before the popup renders. This explains the missing confirmation modal in UAT 10, and it means the 1.0.5 fix for error #10 does not make the prompt appear.
3. **Silent-confirm risk in the exclusions management screen.** Its two `ShowYesNo` calls omit the default, which makes it `Yes`, so under the modern UI a keypress could confirm a deletion with no popup shown. Fix alongside finding 2.
4. **The quest-giver choice hangs off a mod-manufactured event that never fires** (`PlayerBeginConversation`). Its delivery patch was commented out as collateral damage when an unrelated broken transpiler in the same file was disabled. The restock choice survives only because it has a second, independent trade-screen trigger. Splitting that patch file prevents a repeat.
5. **Quest-giver action-key bug**, latent until the choice shows again: both branches use `ApplyNewQuestGiverEffect`, so the already-started-quest branch would highlight the wrong list.
6. **Legendary batch marking excludes hostile creatures by design** — it filters "points of interest," which skip anything hostile to you. Most legendary monsters are hostile, hence the empty result.
7. **The legendary journal write (11.2) is still unexplained.** The journal APIs look unchanged and compatible. A Player.log from a marking attempt is the next step; all journal categories being empty hints at something broader.
8. **Hardcoded `?` key in the game-stats screen** bypasses the rebindable bind system, so the game's own 1.0.4 non-English-layout fix can't reach it. That's the German-keyboard problem.
9. **The cooking patch isn't UI-gated** the way the inventory patch is, so the mod's ASCII recipe screen currently loads in both UI modes.
