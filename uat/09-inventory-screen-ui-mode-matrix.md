# UAT 09: Inventory screen UI-mode matrix

**What was fixed:** QudUX's revamped inventory screen was crashing the mod on 1.0.5 because the game option it checked (`OverlayPrereleaseInventory`) no longer exists. The fix swaps in the game's current option (`ModernUI` + `ModernCharacterSheet`) so QudUX's inventory screen shows in exactly the same situations it always did, and steps aside for the game's own modern inventory or a confused player, exactly as before.

> See [`00-setup.md`](00-setup.md) for: installing the build, wish-prompt key (`Ctrl+W`), log location, save discipline, and evidence file conventions. Run this script early — script **12** (inventory filter prompt) needs the QudUX inventory active and assumes you finished this one.

## 1. Feature under test

`Harmony Patches/Patch_XRL_UI_InventoryScreen.cs` — a Harmony prefix on `XRL.UI.InventoryScreen.Show`. It decides, every time you open the inventory (`i` key, `CmdInventory`), which of **three** screens you get:

| Screen | Source |
|---|---|
| **QudUX inventory** | `Screens/QudUX_InventoryScreen.cs` |
| **Game's classic (ASCII) inventory** | vanilla `XRL.UI.InventoryScreen` |
| **Game's modern inventory** | vanilla `Qud.UI.InventoryAndEquipmentStatusScreen` (graphical overlay, not ASCII) |

The current condition (post-fix), from the patch:

```csharp
if (QudUXOptions.UI.UseQudUXInventory
    && !(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)
    && XRLCore.Core.Game.Player.Body.GetConfusion() <= 0)
{
    __result = new XRL.UI.QudUX_InventoryScreen().Show(GO);
    return false; // QudUX screen shown
}
return true; // falls through to whatever the game would normally show
```

Two things this test is specifically trying to catch:
- The game's own dispatcher (`XRL.UI.Screens.Show`) routes to the **modern** inventory only when **both** `ModernUI` **and** `ModernCharacterSheet` are on — not `ModernCharacterSheet` alone. An earlier draft of this fix used `!ModernCharacterSheet` alone, which would have wrongly hidden the QudUX inventory whenever `ModernCharacterSheet` was on even with `ModernUI` off. Row **B** below targets exactly this case.
- Confusion should still force the classic screen regardless of options.

## 2. Preconditions

**Game options** (`Esc` → Options):

| Option | Category | Exact display text | Value needed |
|---|---|---|---|
| `OptionShowAdvancedOptions` | App Settings | **Show advanced options** | `Yes` — must be on first, or the next two options are hidden |
| `OptionModernUI` | UI | **Enable modern UI** | varies by matrix row (see §4) |
| `OptionModernCharacterSheet` | UI | **Enable modern UI character sheet** | varies by matrix row (see §4) — only visible in the menu while `Enable modern UI` is `Yes` |

Neither `OptionModernUI` nor `OptionModernCharacterSheet` is marked `Restart="true"` in the game's `Base/Options.xml` (only `OptionEnableMods` and `OptionEnablePrereleaseContent` are, in the whole file) — **no game restart is needed** after toggling them, unlike some QudUX options.

**QudUX option** (`Esc` → Options → **QudUX** category):

| Option | Exact display text | Value |
|---|---|---|
| `QudUX_OptionUseInventoryMenu` | **Use revamped inventory text UI** | varies by matrix row |

Also on the QudUX category tab (leave at default `Yes` for this test so sprites are visible): `QudUX_OptionShowInventoryTiles` — " In revamped inventory, show item sprites". No restart flag on `QudUX_OptionUseInventoryMenu` either, so this can all be done in one sitting.

**Save state:** loaded `qudux-32-test-uat-character`, baseline save from `00-setup.md` §4.

## 3. Setup

1. `Esc` → Options → **App Settings** → set **Show advanced options** to `Yes`. Back out (`Esc`) to confirm it saved.
2. 📸 `UAT-09-00-advanced-options.png`: App Settings tab showing **Show advanced options = Yes**.
3. Go to Options → **UI** category. 📸 `UAT-09-00-ui-options.png`: showing the exact rows **Enable modern UI** and **Enable modern UI character sheet** (the latter only appears once the former is `Yes` — toggle **Enable modern UI** to `Yes` momentarily if needed to confirm the row exists and its exact text, then continue with step 4).
4. Go to Options → **QudUX** category. 📸 `UAT-09-00-qudux-options.png`: showing **Use revamped inventory text UI** and " In revamped inventory, show item sprites" (leave the sprites option `Yes`).

## 4. Test steps — the matrix

For every row: set the two game options first, back out of Options (`Esc`) to apply, then set the QudUX option, back out again, then press `i` to open the inventory. Take the screenshot, then `Esc`/`5` to close the inventory before moving to the next row.

Row **B** requires an extra sub-step because `Enable modern UI character sheet` is hidden from the menu once `Enable modern UI` is `No` — set it while `Enable modern UI` is still `Yes`, then turn `Enable modern UI` back off without touching the character-sheet checkbox again. ⚠️ unverified — confirm in game: that the underlying `ModernCharacterSheet` value stays `Yes` after `Enable modern UI` is hidden/turned off (rather than the game silently resetting it). Screenshot 4b's result either confirms or refutes this.

| Row | Enable modern UI | Enable modern UI character sheet | Use revamped inventory text UI | Expected screen |
|---|---|---|---|---|
| A | No | No | Yes | QudUX inventory |
| A | No | No | No | Game's classic inventory |
| B | No | Yes *(set while ModernUI was Yes, see above)* | Yes | QudUX inventory |
| B | No | Yes | No | Game's classic inventory |
| C | Yes | No | Yes | QudUX inventory |
| C | Yes | No | No | Game's classic inventory |
| D | Yes | Yes | Yes | Game's modern inventory (QudUX suppressed) |
| D | Yes | Yes | No | Game's modern inventory (QudUX suppressed) |

**How to tell the three screens apart** (all are otherwise similar ASCII layouts, so check these specific tells):

- **QudUX inventory** — distinguishing features from `Screens/QudUX_InventoryScreen.cs`:
  - A **`Main | Other`** tab bar in the top-left (row 1 of the screen), switchable with Left/Right or numpad 4/6, with `Ctrl+M` to move the selected category between tabs. The vanilla classic screen has no tabs at all.
  - Item **sprite tiles** drawn to the left of each item's name (from `TileMaker`) when sprites are on — not just a single ASCII glyph.
  - Pressing `.` or `0` toggles a **value-per-lb display** on the right edge of each item row (via `AltDisplayMode`) — vanilla classic has no such toggle.
  - Category headers still use `[+]`/`[-]` expand markers and `Ctrl+F`/`,` opens the same "Enter text to filter inventory by item name." prompt — these two are shared with the vanilla classic screen, so they do **not** by themselves prove which screen is up; use the tab bar or the value/lb toggle instead.
- **Game's classic inventory** — same ASCII box, header "`[ Inventory ]`", category list with `[+]`/`[-]`, filter prompt — but **no `Main | Other` tab bar** and pressing `.`/`0` does nothing.
- **Game's modern inventory** (`Qud.UI.InventoryAndEquipmentStatusScreen`) — the graphical/Unity panel-based overlay (mouse-usable, no `{{color}}` ASCII markup visible), not the console-style ASCII box at all. Unmistakably different rendering style from either ASCII screen.

**Steps 1–8** (one per matrix row above, in order A/Yes, A/No, B/Yes, B/No, C/Yes, C/No, D/Yes, D/No):

- Action: set options per the row, press `i` to open inventory.
- Expected result: the screen named in the "Expected screen" column for that row appears.
- Evidence: 📸 `UAT-09-0<n>-row-<row><yn>.png` (e.g. `UAT-09-01-row-A-yes.png` … `UAT-09-08-row-D-no.png`), frame must show the full inventory screen including the top-left corner (tab bar or its absence) and at least one item row.

## 5. Confusion negative case (step 9)

Even where the matrix says "QudUX inventory" (e.g. row C, QudUX option `Yes`), a confused player must still get the classic screen — `XRLCore.Core.Game.Player.Body.GetConfusion() <= 0` gates it.

1. Set options to row **C / Yes** (Enable modern UI = No, Enable modern UI character sheet = No, Use revamped inventory text UI = Yes) — confirmed above to show the QudUX inventory.
2. Open the wish prompt (`Ctrl+W`, per `00-setup.md`) and enter: `iamconfused`
   (confirmed in decompiled `XRL.World.Capabilities.Wishing.HandleWish`: `if (Wish == "iamconfused") { XRLCore.player.ApplyEffect(new XRL.World.Effects.Confused(80, 10)); }` — applies 80 turns of confusion, level 10, directly to the player.)
3. Action: press `i` to open the inventory while confused.
   Expected result: the **game's classic inventory** appears, not the QudUX inventory, even though `Use revamped inventory text UI = Yes`.
   Evidence: 📸 `UAT-09-09-confused-classic-inventory.png` — must show the classic ASCII inventory (no `Main | Other` tab bar) plus a visible confusion indicator (status effect icon/message) confirming the player is still confused.
4. `Esc` to close the inventory. Confusion wears off after 80 turns, or wait it out / rest; not required for this script.

## 6. Edge case: escape/cancel

1. From any matrix row with the QudUX inventory showing, press `Esc`.
   Expected: inventory closes cleanly back to the main game view, no error popup, no stuck input.
   Evidence: 📸 `UAT-09-10-escape-clean.png` — main game view after closing, sidebar/HUD intact.

## 7. Log check

After finishing, search `Player.log` (path in `00-setup.md` §2):

```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "Exception|QudUX_InventoryScreen|Patch_XRL_UI_InventoryScreen|GetConfusion|ModernCharacterSheet" | Select-Object -First 80
```

- **FAIL** if any line contains `Exception` with a stack trace mentioning `QudUX_InventoryScreen`, `Patch_XRL_UI_InventoryScreen`, or `InventoryScreen.Show`.
- **FAIL** if Harmony logs a patch-application error for `XRL.UI.InventoryScreen` (search also for `HarmonyLib` + `Prefix` near startup).
- Informational only (not itself a fail): any ordinary `QudUX` log lines from other patches.

Copy `Player.log` to `qudux-mod-docs/uat/evidence/09/Player-09.log` before closing the game (per `00-setup.md` §2).

## 8. Fail indicators

- Wrong screen for a matrix row (e.g., QudUX inventory still shows in row D, or the game's modern inventory shows in rows A/B/C).
- QudUX inventory shows while confused (row 9's expected classic screen doesn't appear).
- Game crashes, hangs, or shows a Harmony/exception popup when pressing `i`.
- `Main | Other` tab bar or value/lb toggle (`.`/`0`) doesn't work inside what's supposedly the QudUX inventory (would mean the wrong screen rendered but was misidentified).
- Any `Exception` in `Player.log` referencing this patch or `QudUX_InventoryScreen` after any step.

## 9. Results

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
| 0 (setup screenshots) | | | |
| 1 (Row A, Yes) | | | |
| 2 (Row A, No) | | | |
| 3 (Row B, Yes) | | | |
| 4 (Row B, No) | | | |
| 5 (Row C, Yes) | | | |
| 6 (Row C, No) | | | |
| 7 (Row D, Yes) | | | |
| 8 (Row D, No) | | | |
| 9 (Confusion → classic) | | | |
| 10 (Escape clean) | | | |
| Log check | | | |

### Problems found

For each problem:

```
Steps to reproduce:
1.
2.

Expected:

Actual:

Evidence: (file name)

Player.log excerpt:
```
```

## Patch-note correlation (2023–2026)

Sources grepped: `patchnotes/2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026 entries, builds 211.33–212.17). No web fetches were needed — everything below is sourced from these local wikitext files plus decompiled game code (`XRL.UI.Screens`, `Qud.UI.InventoryAndEquipmentStatusScreen`, `XRL.UI.InventoryScreen`) and the mod's own `Options.xml` / git history.

### 1. Relevant patch entries

**Build 207.31 "Spring Molting beta" — released May 10, 2024** (`2024.wiki:754-756`):
> `== 207.31: Spring Molting beta ==`
> `[https://freeholdgames.itch.io/cavesofqud/devlog/729335/spring-molting-beta-out-now Released May 10, 2024.]`
> `We added an entire new UI. (ie, we completed work on the modern UI, for folks who've been following our progress). More polish will be coming over the next several weeks, but all the functionality is there.`
> `* There are too many changes to fully document, but here's a partial list of new screens: trade, quests, message log, character sheet, skills, equipment & inventory, tinkering, game summary, ...`

This **confirms** the OBSOLETE.md claim: build 207.31, May 10 2024, is when the modern graphical Equipment & Inventory screen went live as the completed "new UI" (not a prerelease overlay anymore). Prior to this the same feature existed only as an opt-in "prerelease"/"overlay" UI — evidenced by earlier 2023 entries such as `2023.wiki:598` ("Fixed a bug that caused keyboard input not to register on **prerelease inventory** and trade screens...") and `2023.wiki:1777` ("...if you did it through the **overlay UI** inventory view"), both years before the mod's guard condition (`OverlayPrereleaseInventory`, confirmed via `git log -p` on `Patch_XRL_UI_InventoryScreen.cs`, commit history back to `9aec961`) was written against exactly that prerelease flag.

**Build 207.69 — released June 14, 2024** (`2024.wiki:585-592`):
> `== 207.69 ==`
> `[https://store.steampowered.com/news/app/333640/view/6340585850814995083?l=english Released June 14, 2024.]`
> `* Added an option to disable the modern character sheet while keeping other elements of the new UI (Modern UI > Modern character sheet).`

This **confirms** the second OBSOLETE.md claim exactly: build 207.69, June 14 2024, added the `Modern UI > Modern character sheet` toggle — the same `ModernCharacterSheet` option `Patch_XRL_UI_InventoryScreen.cs` now checks. Both dates/builds in OBSOLETE.md are correct; no corrections needed.

**Later native inventory/equipment feature growth QudUX used to be unique for** — all from **Release 1.0, build 209.29, December 5, 2024** (`2024.wiki:107-135`), well after the mod's last known update:
> `* Renamed the Skills & Powers screen to Skills, the Attributes screen to Attributes & Powers, and the Inventory & Equipment screen to Equipment.`
> `** Added a new option that allows pagination binds to move between inventory and equipment panes and arrow keys to collapse / expand categories...`
> `** Added a selection highlight to the inventory and equipment lists.`
> `** Added hotkeys to the equipment pane in list mode.`
> `** The inventory search filter now prefers exact substring matches instead of fuzzy searching by default. The search mode can be changed in the toggle options on the Equipment screen.`
> `** Added tooltips for the filter categories.`

Earlier, `207.72` (`2024.wiki:580`, released June 21, 2024) also fixed: "Fixed a layout issue that prevented some inventory categories from being clicked when there were a lot of them" — confirming category-based UI was already present in the modern screen by mid-2024.

**Gap found:** searching `2025.wiki` and `index.wiki` (2026, builds 211.33–212.17) for `inventory|equipment screen|filter|sort|category|value.per` returns **zero matches**. The native inventory/equipment screen appears to have had no further functional changes logged in 2025–2026 beyond the 209.29 (Dec 2024) feature set. ⚠️ unverified — this is an absence-of-evidence result from the available wiki exports, not a confirmed feature freeze; it's possible relevant changes were folded into unlabeled "misc UI" bugfix lines that didn't match these keywords.

**No native value-per-weight feature was ever found.** Grepping all four files for `value per|value/weight|per pound|per lb|drachm` returns nothing. Decompiling `Qud.UI.InventoryAndEquipmentStatusScreen` confirms its `SortMode` enum is only `{ AZ, Category }` — there is no value or value-per-weight sort/display mode, and no `GetValue`/price-per-item field on the inventory line data (only `priceText`, which is the free drams/water counter, not item value). This is a real, still-unique QudUX feature (`AltDisplayMode`, toggled by `.`/`0`, calling `InventoryScreenExtender.GetItemValueString`).

### 2. Obsolescence verdict: **PARTLY OBSOLETE**

Feature-by-feature, QudUX inventory (`Screens/QudUX_InventoryScreen.cs`) vs. the two native screens:

| QudUX feature | vs. native **modern** inventory (`InventoryAndEquipmentStatusScreen`, ModernUI+ModernCharacterSheet) | vs. native **legacy/classic** inventory (`XRL.UI.InventoryScreen`) |
|---|---|---|
| Category grouping, expand/collapse `[+]`/`[-]` | Present natively (`categoryCollapsed` dict, `SetCategoryExpanded`) — **obsolete** | Present natively (`CategoryList`/`Expanded`, same mechanism QudUX was cloned from) — **obsolete** |
| Name filter/search (`Ctrl+F`/`,`) | Present natively, and improved past QudUX (exact-substring default since 209.29, plus a fuzzy mode) — **obsolete, native is better** | Present natively (`filterString`, decompiled confirms) — **obsolete** |
| Category filter bar (filter to one category at a time) | Present natively (`FilterBar`, `filterBarCategories`, `"*All"`) — **obsolete** | **Not present** in classic screen (only expand/collapse, no isolate-one-category filter) — **still valuable for legacy-UI players** |
| Sprite/tile icons per item (`TileMaker`) | Native modern screen is fully graphical — has real icons everywhere — **obsolete, native is strictly better** | Classic screen decompiled: no `TileMaker`/tile rendering at all — **still valuable for legacy-UI players** |
| `Main \| Other` tab split | No equivalent concept in modern screen (it uses inventory+equipment panes, not a Main/Other split) — **different paradigm, not directly replaced** | Not present in classic screen — **still valuable for legacy-UI players** |
| Value-per-lb toggle (`.`/`0`, `AltDisplayMode`) | **Not present natively in either UI mode** (`SortMode` is AZ/Category only, no value field) — **still valuable in both modes** | Not present — **still valuable for legacy-UI players** |
| Sort by category vs. name | Native modern has explicit `SortMode.AZ`/`SortMode.Category` — **obsolete** | Classic screen has the same category/name sort QudUX cloned (`SortVs`) — **obsolete** |

Net: against the **modern** screen, QudUX's inventory is almost entirely obsolete — the only genuinely unique surviving feature is the value-per-lb toggle, which no native screen offers. Against the **legacy/classic ASCII** screen (still reachable whenever `ModernUI=No`, or `ModernUI=Yes` with `ModernCharacterSheet=No`), QudUX still adds real value: sprite tiles, the category filter bar, the Main/Other tab split, and the value-per-lb toggle, none of which the classic screen has. Since the takeover guard only ever suppresses QudUX when **both** `ModernUI` and `ModernCharacterSheet` are on (i.e., exactly when the player is already getting the strictly-better modern screen), the feature is not "obsolete" so much as **correctly scoped** by the existing guard — OBSOLETE.md's blanket "remove it" verdict conflates "obsolete when compared to modern UI" with "obsolete everywhere," which the legacy-UI comparison above does not support.

### 3. Modern-UI interaction (from decompiled `XRL.UI.Screens.Show`)

Decompiling `XRL.UI.Screens.Show(GameObject GO)` (`Assembly-CSharp.dll`) gives the authoritative dispatch:

```csharp
public static void Show(GameObject GO)
{
    if (Options.ModernUI && Options.ModernCharacterSheet)
    {
        ...
        _ = StatusScreensScreen.show(startingScreen, GO).Result;
        return;
    }
    GameManager.Instance.PushGameView("StatusScreens");
    ...
    while ((screenReturn = ScreenList[CurrentScreen].Show(GO)) != ScreenReturn.Exit) { ... }
    ...
}
```

`ScreenList` is built in the constructor as `{ SkillsAndPowersScreen, StatusScreen, InventoryScreen, EquipmentScreen, FactionsScreen, QuestLog, JournalScreen, TinkeringScreen }` — i.e. the classic ASCII `XRL.UI.InventoryScreen` (the type QudUX's Harmony patch targets) is only ever reached when the `if` branch is **not** taken.

Resulting matrix, combining the game's own dispatch with QudUX's guard `UseQudUXInventory && !(ModernUI && ModernCharacterSheet) && confusion <= 0`:

| ModernUI | ModernCharacterSheet | QudUX "Use revamped inventory text UI" | Screen shown |
|---|---|---|---|
| No | No (hidden) | Yes | QudUX inventory |
| No | No (hidden) | No | Native classic ASCII inventory |
| Yes | No | Yes | QudUX inventory |
| Yes | No | No | Native classic ASCII inventory |
| Yes | Yes | Yes or No (irrelevant) | Native modern graphical inventory (`StatusScreensScreen`/`InventoryAndEquipmentStatusScreen`) — QudUX's Harmony prefix never even runs `XRL.UI.InventoryScreen.Show`, since `XRL.UI.Screens.Show` never calls into `ScreenList` at all in this case |
| any | any | any | Confused player (`GetConfusion() > 0`): always native classic ASCII inventory, overriding QudUX regardless of the option |

This exactly matches the fix already on this branch (`!(ModernUI && ModernCharacterSheet)`), and matches the UAT-09 test matrix rows A–D in `09-inventory-screen-ui-mode-matrix.md` §4. `ModernCharacterSheet` alone (with `ModernUI` off) has **no effect** on this dispatch — the game only reads it when `ModernUI` is also true — so row B's expectation (QudUX inventory shows even with `ModernCharacterSheet=Yes` as long as `ModernUI=No`) is correct per the decompiled logic.

**What a feature-flag implementation should do:** mirror the game's own two-flag AND gate, not a single flag — this is already what the current guard does (`!(ModernUI && ModernCharacterSheet)`), so no change is needed to the gating logic itself. The flag should continue to be read live (both game options are confirmed un-flagged for `Restart="true"` in `Base/Options.xml`, so no restart-caching concerns), and should not be collapsed to `!ModernCharacterSheet` alone (that was explicitly called out in this doc as the wrong earlier draft, row B being the counterexample) or to `!ModernUI` alone (would wrongly suppress QudUX for the very common case of ModernUI=Yes/ModernCharacterSheet=No, which is one of the "still valuable" configurations above).

### 4. Recommendation

**Keep the feature, gated exactly as-is (`!(ModernUI && ModernCharacterSheet)`), with a documentation/option-text update — do not remove.**

- Files involved: `Screens/QudUX_InventoryScreen.cs` (screen itself — keep), `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs` (guard — keep current fixed condition, already correct), `Options.xml` (`QudUX_OptionUseInventoryMenu`, DisplayText "Use revamped inventory text UI" — keep, consider clarifying display text to note it only applies outside full modern UI), `Utilities/TileMaker.cs` (inventory sprite caller — keep; it's the source of QudUX's only remaining edge over the legacy screen alongside the value-per-lb toggle).
- "Remove" would only be correct if the feature were dead in **both** UI modes. It isn't: for any player with `ModernUI=No`, or `ModernUI=Yes`+`ModernCharacterSheet=No` (a real, officially supported combination per 207.69's own patch note, which explicitly added `ModernCharacterSheet` as a way to keep "other elements of the new UI" while opting out of just the character sheet/inventory-style modern screen), QudUX's screen is the only one offering sprite tiles, the category filter bar, and the value-per-lb toggle.
- Do not "simplify by leaning on a native API" for the legacy-UI case — there is no native API providing tile-based icons or value-per-lb display inside the classic ASCII screen; the closest native equivalent (`InventoryAndEquipmentStatusScreen`) is a different, non-ASCII, non-reachable-in-legacy-mode screen, not an API QudUX's legacy-mode users could opt into without also accepting the full modern UI switch.
- This matches the user's stated overall conclusion: obsolete-with-modern-UI features should keep working for legacy-UI players via a feature-flag approach driven by the player's UI setting — that is precisely the shape of the existing `!(ModernUI && ModernCharacterSheet)` guard, so the recommendation is to leave the gating logic as committed on this branch.

### 5. Confidence + gaps

**Verified directly:**
- Both OBSOLETE.md build/date claims (207.31 / May 10 2024; 207.69 / June 14 2024), via verbatim wikitext quotes.
- The exact dispatch logic in `XRL.UI.Screens.Show` via decompilation — confirms the AND-gate and confirms the classic `InventoryScreen` is unreachable when `ModernUI && ModernCharacterSheet`.
- Feature comparison across all three screens via decompiling `Qud.UI.InventoryAndEquipmentStatusScreen` and `XRL.UI.InventoryScreen`, and reading `QudUX_InventoryScreen.cs` in full.
- The mod's prior (`OverlayPrereleaseInventory`) vs. current (`ModernUI && ModernCharacterSheet`) guard condition, via `git log -p` on `Patch_XRL_UI_InventoryScreen.cs`.
- `Options.xml` confirms `QudUX_OptionUseInventoryMenu` DisplayText and default.

**Gaps / not verified:**
- ⚠️ unverified — whether 2025–2026 native builds changed the inventory/equipment screen in ways not captured by the keyword grep (the wiki entries for those years may summarize UI changes without using the exact searched terms). No 2025/2026 entries were found at all touching this area, which is itself notable but not independently cross-checked against another source (e.g., Steam patch notes directly) since the task caps web fetches and none were spent here as the local wiki search was conclusive enough for the two claims required.
- ⚠️ unverified — in-game confirmation that `ModernCharacterSheet` value persists correctly when `ModernUI` is toggled off (the existing UAT script itself already flags this as unverified in §4/row B; this correlation pass does not add new evidence either way).
- Did not exhaustively decompile `StatusScreensScreen` or `EquipmentScreen` (native classic) beyond what was needed to confirm the dispatch and feature set; deeper paperdoll/equipment-specific comparisons were out of scope for the inventory-focused UAT-09 topic.
