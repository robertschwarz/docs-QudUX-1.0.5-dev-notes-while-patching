# UAT 12: Inventory filter prompt — OBSOLETE, SKIPPED

**Status:** SKIPPED — depends entirely on `QudUX_InventoryScreen`, which is obsolete. See `evidence/09/OBSOLETE.md`.

**What was fixed:** `Popup.AskString` signature changed in game 1.0.5, breaking the mod build. The fix updates the call signature.

**Why skipped:** The filter prompt lives inside `Screens/QudUX_InventoryScreen.cs`. That screen is superseded by the native graphical inventory (build 207.31, May 2024). No QudUX inventory screen → no filter prompt → nothing to test.

---

## Original test content (archived)

The game updated a popup function's signature (`Popup.AskString`), which broke QudUX's build entirely (the mod wouldn't compile/load at all). The fix updates the call so the mod builds again. In-game, this test proves the **inventory item-name filter prompt** (opened with `Ctrl+F` or `,`) still works exactly like it did before: it opens, shows your current filter as a starting point, filters the list as you'd expect, and doesn't crash on Escape.

## Feature under test

- **Where:** QudUX's classic (ASCII) inventory screen, `Screens/QudUX_InventoryScreen.cs`.
- **What:** the "Enter text to filter inventory by item name." text-input popup, opened three ways inside that screen: `Ctrl+F`, the `,` key, or the mouse/UI command `CmdFilter` (also bound to `Ctrl+F` at the base-game level, category "Menus", `DisplayText="Filter/Search"`).
- **Code path (for reference, not something you need to read to run this):** `Screens/QudUX_InventoryScreen.cs:632-636`. Filtering itself matches against `Obj.GetCachedDisplayNameForSort()` using `CompareOptions.IgnoreCase` (case-insensitive substring match, color codes stripped). `Popup.AskString` is called with `MaxLength: 80, MinLength: 0`. `FilterString` is a **static field reset to `""` at the very top of `Show()`** (line 305) — so, per the code, it does **not** persist across closing and reopening the inventory screen.

## Preconditions

1. QudUX option (Options menu, category **QudUX**): **"Use revamped inventory text UI"** = `Yes` (this is the default).
2. Game options must NOT have both of the following on at once, or QudUX's classic inventory screen is bypassed entirely in favor of the game's own modern inventory UI (see `uat/09-*.md` for the full UI-mode matrix; this is the minimum combo needed here):
   - Options → **App Settings** → **"Show advanced options"** = `Yes` (needed just to reveal the next option)
   - Options → **UI** → **"Enable modern UI character sheet"** = `No`
   - ("Enable modern UI" itself can be left on or off — only "Enable modern UI character sheet" needs to be off.)
3. Save state: fresh/new game is fine. Character: `qudux-32-test-uat-character` (per setup doc).

## Setup

Use the wish prompt (see `uat/00-setup.md` for how to open it). These are exact, verified blueprint IDs from the base game's `ObjectBlueprints/Items.xml` (exact case matters — wishing resolves via an exact dictionary key lookup first):

1. Wish `Dagger` → spawns a **Dagger** blueprint next to you, displayed in-game as "**bronze dagger**".
2. Wish `Waterskin` → spawns a **Waterskin** blueprint, displayed as "**waterskin**".
3. Wish `Club` → spawns a **Club** blueprint, displayed as "**club**".
4. Walk onto each spawned item's tile to pick it up (autoget, or the game's default "get" command, per setup doc).
5. Open your inventory. Confirm you see the classic QudUX ASCII inventory (title bar reads `[ Inventory ]` centered at the top, with `ESC or 5 to exit` on the same line) — **not** a modern/graphical inventory panel. If you see anything else, fix your options per the Preconditions section before continuing.

## Test steps

Evidence goes in `qudux-mod-docs/uat/evidence/12/`.

1. **Baseline inventory view.**
   Action: With the QudUX inventory open and no filter active, look at the item list.
   Expected: All 3 wished items (bronze dagger, waterskin, club) are visible in their categories; no "Filtering on" line at the top; no "DEL to remove filter" hint.
   Evidence: 📸 `UAT-12-01-baseline.png` — full screen showing all 3 items and the plain top bar (no filter banner).

2. **Open the filter prompt with Ctrl+F.**
   Action: Press `Ctrl+F`.
   Expected: A text-input popup appears with the message **"Enter text to filter inventory by item name."** and an empty input field (no prior filter set).
   Evidence: 📸 `UAT-12-02-ctrlf-prompt.png` — the popup with its exact message text visible.

3. **Escape with no input (negative case).**
   Action: Press `Escape` to cancel the prompt without typing anything.
   Expected: No crash, no freeze. The prompt closes and the inventory redraws. Per the decompiled behavior, this call returns `""` (not `null`), so the filter is/remains empty — no "Filtering on" banner appears, and the full item list is shown.
   Evidence: 📸 `UAT-12-03-escape-empty.png` — inventory screen after Escape, no filter banner, all items visible.

4. **Open the filter prompt with the `,` key.**
   Action: Press `,` (comma).
   Expected: Same popup as step 2 appears — message text **"Enter text to filter inventory by item name."**, empty field (filter is still unset from step 3).
   Evidence: 📸 `UAT-12-04-comma-prompt.png`.

5. **Filter by partial name — "dagger".**
   Action: In the open prompt, type `dagger` and press Enter.
   Expected: List now shows only the **bronze dagger**. Top-left shows `Filtering on "dagger"` (yellow); bottom-left shows `N items hidden by filter`; top-right area shows `DEL to remove filter`.
   Evidence: 📸 `UAT-12-05-filter-dagger.png` — must show the "Filtering on "dagger"" line, the "items hidden by filter" count, and only the dagger in the list.

6. **Filter persists as the default when reopening the prompt.**
   Action: Press `Ctrl+F` again to reopen the filter prompt.
   Expected: The input field is **pre-filled with `dagger`** (the current `FilterString` is passed as `Popup.AskString`'s `Default` argument) — not empty.
   Evidence: 📸 `UAT-12-06-prefilled-default.png` — the popup showing `dagger` already in the field.

7. **Case-insensitive matching.**
   Action: Clear the pre-filled text and type `DAGGER` (all caps), press Enter.
   Expected: Same result as step 5 — only the bronze dagger shown, "Filtering on "DAGGER"" banner (note: the banner echoes back exactly what you typed, case included, even though the match itself is case-insensitive).
   Evidence: 📸 `UAT-12-07-case-insensitive.png`.

8. **Filter by a different partial name — "water".**
   Action: Open the prompt (`Ctrl+F` or `,`), clear the field, type `water`, press Enter.
   Expected: List now shows only the **waterskin**. Dagger and club are hidden; "items hidden by filter" count reflects that.
   Evidence: 📸 `UAT-12-08-filter-water.png`.

9. **Max length (80 chars) does not crash.**
   Action: Open the prompt, clear the field, and type this 90-character string (paste if possible, otherwise type it out — exact length doesn't need to be visually verified, this step is checking for a crash/freeze, not counting characters on screen):
   `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA`
   Press Enter.
   Expected: No crash, no freeze, no error popup. The screen redraws normally (with the "Filtering on" banner showing whatever text was actually accepted — it's fine if it's visibly truncated versus what you typed).
   Evidence: 📸 `UAT-12-09-maxlength.png` — screen after submitting, showing no crash/error dialog.

10. **Empty submit clears the filter.**
    Action: Open the prompt, select-all/delete the field so it's empty, press Enter.
    Expected: Filter clears — "Filtering on" banner and "DEL to remove filter" hint disappear, all 3 items are visible again, same as step 1.
    Evidence: 📸 `UAT-12-10-empty-submit-clears.png`.

11. **Filter does NOT persist across closing/reopening the inventory.**
    Action: Filter by `club` (Ctrl+F → type `club` → Enter) and confirm only the club is shown. Then exit the inventory screen entirely (`Escape` or `5`), then reopen the inventory.
    Expected: Per the code (`FilterString = ""` at the top of `Show()`), the filter resets — you should see **all 3 items again with no filter banner**, even though you didn't manually clear it. This is expected, correct behavior for this version, not a bug.
    Evidence: 🎥 `UAT-12-11-filter-resets-on-reopen.mp4` — record from applying the `club` filter, through exiting, to the reopened inventory showing all items unfiltered.

## Edge / negative cases

- **Delete key clears an active filter (adjacent code path, not the prompt itself, but worth confirming still works after the fix).**
  Action: With a filter active (e.g. redo step 5's `dagger` filter), press `Delete`.
  Expected: Filter clears immediately, no prompt appears, all items shown again.
  Evidence: 📸 `UAT-12-12-delete-clears.png`.

- ⚠️ unverified — confirm in game: whether the mouse/`CmdFilter` command has any actual clickable on-screen element in this classic ASCII inventory screen (the code checks `keys.IsMouseEvent("Command:CmdFilter")`, but no visible clickable "Filter" button/label was found in the screen-drawing code). If you have a mouse bound and want to try clicking anywhere near the filter-related text, note the result, but don't block the test on this — it's optional.

- **Option toggled off → feature absent.** Not applicable to this specific popup (it's part of the inventory screen itself); if "Use revamped inventory text UI" is set to `No`, the whole QudUX inventory screen (and thus this filter prompt) is replaced by the base game's inventory — this is expected and out of scope for this issue.

## Log check

After finishing the steps above, open `%USERPROFILE%\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log` (see `uat/00-setup.md`) and search for:

- `Exception` or `Error` anywhere in the vicinity of timestamps matching your test session — especially anything mentioning `AskString`, `QudUX_InventoryScreen`, or `Popup`.
- `NullReferenceException` — would indicate the "Escape returns null instead of empty string" risk called out in the issue doc actually manifested.
- `QudUX` — general sanity check that no QudUX-sourced error fired during the session.

**FAIL** if any exception/stack trace appears that is timestamped during or immediately after steps 2–11, or if the log shows the inventory screen was force-closed/reset unexpectedly.

## Fail indicators

Any of the following means FAIL, even if not explicitly caught in Player.log:

- Game freezes, crashes, or force-closes the inventory screen when opening/submitting/escaping the filter prompt.
- The prompt's message text does not read exactly "Enter text to filter inventory by item name."
- Typing text does not filter the list, or filters incorrectly (wrong items shown/hidden).
- Filter matching is case-sensitive (typing `DAGGER` fails to match "bronze dagger").
- Reopening the prompt does NOT show the current filter pre-filled (shows empty instead).
- Escape crashes the game or visibly leaves a stale/incorrect filter state.
- Empty submit does not clear the filter.
- Typing a >80-character string crashes, freezes, or corrupts the screen.

## Results

| Step | PASS/FAIL | Evidence file | Notes |
|------|-----------|----------------|-------|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |
| 9 | | | |
| 10 | | | |
| 11 | | | |
| 12 (edge) | | | |

### Problems found

If any step fails, copy this block per issue:

```
Steps to reproduce:
1.
2.
3.

Expected:

Actual:

Evidence: (filename(s) in qudux-mod-docs/uat/evidence/12/)

Player.log excerpt:
```

## Patch-note correlation (2023–2026)

### 1. Relevant patch entries

Searched `2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026, builds 211.33–212.17) for `filter|search|sort|category`. Every entry that is about item-name filtering/searching (not just any use of the word "category"/"sort"):

- **Build 207.31 "Spring Molting beta", released May 10, 2024** (`2024.wiki`, `=== UI ===` section):
  > "Added a search bar to relevant screens."
  This is the same build `evidence/09/OBSOLETE.md` cites for the native graphical inventory's debut — confirmed correct, and this is also where the modern UI's inventory/equipment *search bar* itself first appears.

- **Build 208.21, released November 11, 2024** (`2024.wiki`):
  > "The inventory search filter now prefers exact substring matches instead of fuzzy searching by default, search mode can be changed in the toggle options on the inventory screen"

- **Release 1.0 (build 209.29), released December 5, 2024** (`2024.wiki`, `=== UI ===` section, restated verbatim as part of the 1.0 Equipment-screen changelog):
  > "The inventory search filter now prefers exact substring matches instead of fuzzy searching by default. The search mode can be changed in the toggle options on the Equipment screen."
  > "Added tooltips for the filter categories."

No 2025 or 2026 entry (`2025.wiki`, `index.wiki`) mentions inventory/equipment name filtering or search again — the feature has been stable since the 1.0 release. (2025/2026 `search` hits are unrelated: options-menu search-text-clearing and Skills-screen search.)

**No patch note anywhere mentions the *legacy* ASCII inventory screen's filter** — because, per the decompiled code below, that filter already existed before 2023 and was never changed in the 2023–2026 window covered by these wikis.

### 2. Native versus mod comparison (decompiled, `Assembly-CSharp.dll` from the installed 1.0.5 game)

**Legacy native `XRL.UI.InventoryScreen`** (decompiled directly, not through the mod) already has this exact feature, independent of QudUX:

```csharp
public static string filterString = "";
...
else if (keys == (Keys.F | Keys.Control) || c == ',')
{
    filterString = Popup.AskString("Enter text to filter inventory by item name.", filterString);
    ClearLists();
}
...
if (filterString != "" && !gameObject.GetDisplayName(int.MaxValue, null, null, AsIfKnown: false, Single: false, NoConfusion: false, NoColor: false, Stripped: true)
        .Contains(filterString, CompareOptions.IgnoreCase))
{
    itemsSkippedByFilter++;
    continue;
}
```
- Same trigger keys (`Ctrl+F` or `,`), same prompt string, same `Popup.AskString(prompt, currentValue)` pre-fill pattern, same case-insensitive substring match (`CompareOptions.IgnoreCase`) against the item's display name with color codes stripped.
- `filterString` is reset to `""` at the top of `Show()` (same non-persistence behavior QudUX has).
- The vanilla legacy screen has **no** `Keys.Delete`-to-clear shortcut and **no** `CmdFilter` mouse/UI-command binding — I grepped the decompiled type for `Keys.Delete`, `CmdFilter`, `CmdDelete` and got zero hits.

**QudUX's `Screens/QudUX_InventoryScreen.cs:632-636`** is essentially a copy of that native code, with three additions:
```csharp
FilterString = Popup.AskString("Enter text to filter inventory by item name.", FilterString, MaxLength: 80, MinLength: 0);
```
- Explicit `MaxLength: 80, MinLength: 0` (native call passes no length args, i.e. relies on `Popup.AskString`'s own defaults).
- Also fires on `keys.IsMouseEvent("Command:CmdFilter")` — `CmdFilter` is a real base-game command (`Commands.xml:375`, `DisplayText="Filter/Search"`, `Category="Menus"`, bound to `Ctrl+F` at the engine level), so this is just recognizing the same command by its mouse/gamepad path, not new engine behavior.
- Adds `Keys.Delete` / `Command:CmdDelete` to instantly clear the filter — the one behavior genuinely absent from the vanilla legacy screen.

So, verified: **the legacy native inventory has always had this exact filter feature**, with the identical prompt text and matching semantics. QudUX's version is a near-verbatim fork of it, not a new capability, plus a minor UX nicety (Delete-to-clear).

**Modern `Qud.UI.InventoryAndEquipmentStatusScreen`** (decompiled) has its own, functionally superior implementation, present since build 207.31 and refined in 208.21/209.29:
```csharp
using FuzzySharp;
using FuzzySharp.Extractor;
...
public static string SEARCH_MODE_FUZZY = "Fuzzy";
public static string SEARCH_MODE_STRICT = "Strict";
public string SearchMode = SEARCH_MODE_STRICT;
...
enumerable4 = (!(SearchMode == SEARCH_MODE_FUZZY))
    ? enumerable4.Where(item => item.sortString.Contains(searcher.sortString, CompareOptions.IgnoreCase))
    : ... Process.ExtractTop(searcher, enumerable4, i => i.sortString, null, enumerable4.Count(), 50) ...
```
- It's a **live incremental text box** (`OnSearchTextChange`/`FilterUpdated`, no modal popup), not a popup prompt like the legacy/QudUX version.
- Two selectable modes, toggled via a UI button and persisted across sessions with `PlayerPrefs.SetString("InventoryScreenSearchMode", ...)`: **Strict** (case-insensitive substring match, `CompareOptions.IgnoreCase`, same primitive as legacy/QudUX) and **Fuzzy** (ranked via the `FuzzySharp` library, top 50 results) — confirms the December 5, 2024 patch note that Strict is now the default.
- Also has a category filter bar (`filterBar`/`filterBarCategories`, "*All" plus per-category chips) that neither the legacy screen nor QudUX has.
- The actual filter **text** resets on screen entry (`OnSearchTextChange(null)` is called during setup), matching the legacy/QudUX non-persistence behavior, but the **search-mode preference** (Strict vs Fuzzy) persists across sessions — legacy/QudUX has no equivalent mode toggle at all.

### 3. Obsolescence verdict: **PARTLY OBSOLETE**

- Relative to the **legacy native inventory**: FULLY redundant. Vanilla `XRL.UI.InventoryScreen` already ships the identical filter (same keys, same prompt text, same case-insensitive substring match) with or without QudUX installed. QudUX's version adds only the Delete-to-clear shortcut as genuinely new behavior.
- Relative to the **modern native inventory**: not reachable there at all (see below) — QudUX's classic screen is bypassed whenever the modern character sheet is active — but even judged on its own merits the modern screen's live incremental box with a persisted Strict/Fuzzy toggle and category chips is a strictly more capable implementation of the same idea.
- Net effect: this specific popup is not "broken" or "wrong," it is a preserved-but-unnecessary reimplementation of something the base game already does on its own, for the one UI mode (legacy) where it's still shown.

### 4. Modern-UI interaction

Per `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs`, QudUX's screen (and thus this filter prompt) only intercepts `XRL.UI.InventoryScreen.Show` when:
```csharp
QudUXOptions.UI.UseQudUXInventory
    && !(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)
    && XRLCore.Core.Game.Player.Body.GetConfusion() <= 0
```
- So this feature is reachable **only** when "Use revamped inventory text UI" = Yes **and** the modern character sheet is not simultaneously active (i.e., Legacy UI, or Modern UI with "Enable modern UI character sheet" = No — matching the UAT-09 matrix already documented in `evidence/09/OBSOLETE.md`).
- Whenever the modern character sheet is on, the native `Qud.UI.InventoryAndEquipmentStatusScreen` handles inventory entirely and QudUX's `Show` prefix never fires — the mod's filter popup is simply never shown, superseded by the native live search box described in §2.
- A feature-flag implementation should therefore gate this file's relevance the same way `Patch_XRL_UI_InventoryScreen` already gates its *activation*: no code change is needed to "turn it off" for modern UI, because the base game already turns it off by construction. The remaining decision is purely about maintenance burden for the legacy-only code path.

### 5. Recommendation

**Keep as-is for legacy UI, do not remove.** The user's stated position is that legacy-UI players should keep working features, and for that audience this prompt is the *only* filter available (the native legacy screen has the same capability, but QudUX's version is what legacy-UI QudUX users are actually shown and have muscle memory for, plus it has Delete-to-clear which vanilla legacy lacks).

- Do **not** "simplify by leaning on a native API" here in the sense of pointing legacy-UI users at `Qud.UI.InventoryAndEquipmentStatusScreen` — that screen is only reachable via the modern character sheet, a different code path entirely, and forcing it on legacy players would violate the user's own instruction-adherence stance (don't change scope/behavior beyond what's asked).
- The only low-risk simplification available, if maintenance burden ever matters: since `XRL.UI.InventoryScreen.filterString`/`Popup.AskString` prompt/match logic is already byte-for-byte equivalent to QudUX's, `Screens/QudUX_InventoryScreen.cs` could in principle delegate the filter-storage/match logic to the vanilla static fields instead of duplicating it — but this is a refactor, not a behavior change, and out of scope for a UAT pass. Flagging it for awareness only.
- Files involved: `Screens/QudUX_InventoryScreen.cs` (632-636, 305), `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs` (existing gate is already correct and needs no change).
- No feature-flag work is required beyond what already exists (`QudUXOptions.UI.UseQudUXInventory` + the `ModernUI && ModernCharacterSheet` check already fully separates the two code paths).

### 6. Confidence and gaps

- **High confidence**: legacy native filter behavior, modern native search behavior, and the QudUX gating logic — all read directly from decompiled `Assembly-CSharp.dll` (game install, presumed 1.0.5 per the mod's current update target) and from the mod's own source, not inferred.
- **High confidence**: the three patch-note quotes above are verbatim from the local wiki dumps with build numbers and dates attached.
- **Gap**: I did not independently re-verify that the installed `Assembly-CSharp.dll` used for decompiling is exactly build 1.0.5 (the task brief states the mod is being updated for 1.0.5 and points at this install; I did not check a build-number string in the binary or game files). ⚠️ unverified — assumed from task context, not confirmed against an in-game version string.
- **Gap**: did not test in-game; all matching-rule claims (case sensitivity, substring vs fuzzy, `sortString` composition) are from static decompiled code, not runtime observation. `sortString`'s exact contents (whether it includes tags/categories beyond display name) were not traced further than its use site.
- No web fetches were needed — the two local wiki dumps plus decompilation fully answered the topic, so the 2-fetch web allowance was unused.
