# UAT-06: Character Sprite Screen — Search Filter Prompt

## 1. What was fixed

The "Modify Character Sprite" screen's search box (type a name to filter the sprite list) used a game API call with the wrong argument order under game version 1.0.5 and would not build/run. It's fixed so the text prompt opens, filters by name, and still enforces its old rules: minimum 3 characters, maximum 30 characters.

## 2. Feature under test

- Screen: **Modify Character Sprite** (`QudUX_CharacterTileScreen`), category **Other (search)**.
- Sub-feature: the "Enter text to filter by object name." text prompt (opened with `,` or `Ctrl+F`), which sets the search filter used to narrow the tile grid to matching game object names.
- Code: `Screens/QudUX_CharacterTileScreen.cs`, method `UpdateSearchString` (~line 384) and `Screen Extenders/CharacterTileScreenExtender.cs` (`CheckBlueprintMatchesQuery`, `InitBlueprintTiles`).

## 3. Preconditions

- **QudUX options:** none required. The option `Modify your sprite from character creation text UI (requires restart)` (`QudUX_OptionCustomSpriteMenu`, Options category "QudUX") does **not** gate this screen — it only controls a character-creation hook, and that hook (`Harmony Patches/Patch_XRL_UI_CreateCharacter.cs`) is entirely commented out in this build. The in-game wish path used below (`Wishes/SpriteMenu.cs`) opens the screen unconditionally, with no option check. Leave this option at its default (checked/Yes) — it has no effect on this test.
- **Game options:** none required to set. "Always map directions to numpad" (`OptionMapDirectionsToKeypad`, Category "Legacy UI", default **Yes**) is the base game default and is assumed below — movement uses Numpad 2/4/6/8. If you've changed this, use your bound direction keys instead.
- **Save state:** existing character `qudux-32-test-uat-character`, loaded and in the game world (not in character creation — the creation-screen route is currently disabled, per above, so this test only covers the wish route).

## 4. Setup

1. Load `qudux-32-test-uat-character` in-game (see `uat/00-setup.md`).
2. Open the wish prompt (see `uat/00-setup.md` for how) and enter exactly:
   ```
   sprite menu
   ```
   This matches the wish regex in `Wishes/SpriteMenu.cs` (`(?:[Qq]ud[Uu][Xx] ?)?[Ss]prite ?[Mm]enu`).
3. **Expected:** the screen immediately switches to a full-screen box titled **"Modify Character Sprite"**, showing a single sprite tile and, near the bottom, "More...".
   - 📸 `UAT-06-00-screen-opened.png` — full screen showing the "Modify Character Sprite" title box.

## 5. Test steps

**Step 1 — Reach the search category**
Action: Press **Numpad-2** (Down) once to highlight "More...", then press **Enter**. In the category list that appears ("Select a category of tiles to browse:"), press **Numpad-2** 8 times to move the cursor down to the last entry, **"Other (search)"**, then press **Enter**.
Expected: The AskString popup appears immediately, with the prompt text **"Enter text to filter by object name."** and an empty input field.
Evidence: 📸 `UAT-06-01-prompt-appears.png` — popup visible with the exact prompt text.

**Step 2 — Below-minimum entry (too short)**
Action: Type `xy` (2 characters) and confirm.
Expected: A popup appears with the exact text **"You must enter at least three characters to perform a search."** After dismissing it, the "Other (search)" screen is shown with the query hint `, or Ctrl+F to change query` at top, but no tile grid (filter was not set) and no crash.
Evidence: 📸 `UAT-06-02-too-short-warning.png` — warning popup with the exact text visible.

**Step 3 — Exactly 3 characters (boundary, accepted)**
Action: Press `,` to reopen the prompt. Type `xyz` (3 characters, a term unlikely to match anything) and confirm.
Expected: No warning this time. Screen shows **"Filtering on: xyz"** at top-left; tile grid area is blank (0 matches) but the screen does not crash or error.
Evidence: 📸 `UAT-06-03-min-length-boundary.png` — header showing "Filtering on: xyz" with an empty grid.

**Step 4 — Valid filter narrows the list**
Action: Press `,` to reopen the prompt. Type `goat` and confirm.
Expected: Header reads **"Filtering on: goat"**; the tile grid now shows a small set of tiles (goat-family objects: e.g. Goat, Farm Goat, Goat Golem, Goat Corpse, goatfolk variants) instead of being empty. Selection info line below the grid shows a display name (e.g. "goat") and a blueprint path containing "Goat".
Evidence: 📸 `UAT-06-04-filtered-goat.png` — grid with goat tiles and the "Filtering on: goat" header both visible.

**Step 5 — Maximum length caps at 30**
Action: Press `,` to reopen the prompt. Type 35 `a` characters in a row (don't confirm yet).
Expected: Input stops accepting characters once 30 `a`s are entered — the 31st–35th keystrokes have no visible effect on the field.
Evidence: 📸 `UAT-06-05a-maxlength-typing.png` — popup showing the capped input before confirming.
Action: Confirm the entry.
Expected: Header reads **"Filtering on: "** followed by exactly 30 `a` characters (count them in the screenshot) — proving the 30-char cap from the fixed call (`MaxLength: 30`) is enforced.
Evidence: 📸 `UAT-06-05b-maxlength-applied.png` — header with the 30-character filter string, countable in the image.

**Step 6 — Escape leaves the filter unchanged**
Action: Press `,` to reopen the prompt, then press **Escape** without typing anything.
Expected: Popup closes, no error, and the header still reads **"Filtering on: "** with the same 30 `a`s as Step 5 (filter unchanged, since an empty/escaped entry is ignored by `UpdateSearchString`).
Evidence: 📸 `UAT-06-06-escape-unchanged.png` — header identical to step 5b.

**Step 7 — Re-entering the category resets and re-prompts**
Action: Press **Escape** (back to the category list, "Other (search)" still highlighted), then press **Enter** again.
Expected: The AskString popup reappears immediately (same prompt text as Step 1) — because re-selecting "Other (search)" clears the stored filter and re-asks. No crash.
Evidence: 📸 `UAT-06-07-reprompt-on-reentry.png` — popup visible again right after re-entering the category.

**Step 8 — Escaping the fresh prompt leaves it cleared**
Action: Press **Escape** at this new prompt.
Expected: Screen shows "Other (search)" with the query hint but a blank grid — filter is empty/cleared (this time there was nothing to preserve, since it was just reset in Step 7). No crash.
Evidence: 📸 `UAT-06-08-cleared-blank.png` — blank grid, no leftover "aaaa..." text.

**Step 9 — Filter doesn't leak into other categories**
Action: Press **Escape** to the category list, move up to a different category (e.g. **Numpad-8** several times to reach "Furniture"), press **Enter**.
Expected: Furniture's full unfiltered tile grid is shown — proving the search filter is scoped to "Other (search)" only and does not affect other categories.
Evidence: 📸 `UAT-06-09-other-category-unaffected.png` — Furniture tile grid, full and unfiltered.

**Step 10 — Apply a filtered tile to the character**
Action: Go back (Escape) to the category list, select "Other (search)" again, at the fresh prompt type `goat` and confirm. On the resulting grid, select the plain "Goat" tile (use Numpad 2/4/6/8 to move the selection box; check the display-name line under the grid reads "goat"). Press **Enter** to confirm.
Expected: Screen closes and returns to the game map; the player character's sprite on the map immediately changes to the goat tile/colors chosen.
Evidence: 🎥 `UAT-06-10-apply-sprite.mp4` — short clip showing the tile selection, pressing Enter, and the character's new sprite visible on the map afterward.

## 6. Edge / negative cases

All the required edge cases for this fix are covered inline above:
- Below-minimum entry → Step 2.
- Exact minimum (3 chars) boundary → Step 3.
- Maximum length cap (30 chars) → Step 5.
- Escape/empty entry behavior → Steps 6 and 8 (unchanged vs. cleared, depending on whether a filter was already set).
- Filter scope/isolation from other categories → Step 9.

There is no player-facing toggle for this specific feature (see Preconditions) — the "toggle off → feature absent" case does not apply here, since the only option that superficially resembles a gate (`QudUX_OptionCustomSpriteMenu`) is unwired for the wish-invoked screen.

## 7. Log check

After completing the steps, open `%USERPROFILE%\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log` (see `uat/00-setup.md` for tailing it live) and search for:

- `QudUX_CharacterTileScreen` — should appear only in benign log lines (e.g. mod load), not inside a stack trace.
- `UpdateSearchString` — should not appear in any exception/stack trace.
- `AskString` — should not appear in any exception/stack trace.
- `Exception`, `Error`, `Failed to retrieve` — check any hits are unrelated to this session's timestamps, or if related, treat as **FAIL** and copy the surrounding lines into the Problems Found section below.
- `Wish invoked: Open Sprite Menu` — should appear once per time you opened the screen via the wish (confirms the wish fired).

**FAIL** if any exception/stack trace references `QudUX_CharacterTileScreen`, `CharacterTileScreenExtender`, `UpdateSearchString`, or `Popup.AskString` at a timestamp matching your test.

## 8. Fail indicators

- Game freezes, crashes to desktop, or the mod menu can't be reopened after any step.
- The prompt text differs from "Enter text to filter by object name." or the warning differs from "You must enter at least three characters to perform a search."
- A 1–2 character entry is accepted as a filter (no warning shown).
- Typing past 30 characters keeps adding characters to the input.
- The tile grid does not change after entering a valid filter (still shows the same tiles as before, or shows all tiles regardless of filter).
- Escaping the prompt changes the filter to something other than "unchanged" (previous value) or "cleared" (when there was nothing to preserve) — e.g. it crashes, or silently applies garbage text.
- Selecting a tile and confirming does not change the character's sprite on the map, or changes it to the wrong tile.
- Any relevant exception in Player.log per Section 7.

## 9. Results

| Step | PASS/FAIL | Evidence file | Notes |
|------|-----------|----------------|-------|
| Setup: screen opens | | | |
| 1. Reach search category / prompt appears | | | |
| 2. Below-minimum entry | | | |
| 3. Exactly 3 chars (boundary) | | | |
| 4. Valid filter narrows list | | | |
| 5. Max length caps at 30 | | | |
| 6. Escape leaves filter unchanged | | | |
| 7. Re-entering category re-prompts | | | |
| 8. Escaping fresh prompt leaves it cleared | | | |
| 9. Filter doesn't leak to other categories | | | |
| 10. Applying tile updates character sprite | | | |
| Log check | | | |

### Problems found

For each problem, copy this block and fill it in:

```
Steps to reproduce:
1.
2.
3.

Expected:

Actual:

Evidence: (file name from qudux-mod-docs/uat/evidence/06/)

Player.log excerpt:
```

## Patch-note correlation (2023–2026)

### 1. Relevant patch entries

Raw UAT source (local notes, lines 67–79) for this topic:
```
# UAT 6: sprite menu OK
note: could use overhaul for modern UI
changing sprites is OK
colors OK
-- issue: if a callout (presumably modern ui issue) renders on top of sprite menu, it gets stuck and you cant exit anymore (esc/5/etc stops working - only fix is to restart the game)
```
This is unlike UAT 2 ("Obsolete: tilemaker for NPC portraits and items") and UAT 4.1 ("qudux menu for recipes is obsolete ... the native menu of modern ui makes it obsolete") — the user did **not** tag UAT 06 obsolete, only "could use overhaul."

Grepped `2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026) for `sprite`, `tile`, `customiz`, `appearance`, `portrait`, `modern ui`, `popup`, `callout`, `scoreboard`:

- **No entry in any year (2023–2026) adds a native player-facing "customize your character's sprite/tile/appearance" menu.** All `tile` hits are art/bugfix items for specific game objects (e.g. `2023.wiki:367`: "Gave portable wall a new tile.", `2023.wiki:1428`: "Replaced the tiles for low-tier relic hats and helmets with different, better-suited existing tiles."). None concern player-avatar sprite customization. `sprite` hits are unrelated (`2024.wiki:948`: "Fixed a bug that caused trails on thrown items to render over their sprite."; `2025.wiki:155/185`: modding-only Textures-folder support). **Unrelated-but-worth-knowing**, not a cause.
- **`2024.wiki` (build `207.40`, released May 18, 2024)**: `"Fixed popups being hidden by other interface elements."` and, same build, `"Fixed a bug that stopped hotkeys from working on an equipment item after interacting with it with a popup until it was reselected."` — **merely related**: both are prior instances of popup/interface-element z-order and input-focus fights during the modern-UI rollout, not a fix of the exact mechanism found below (that mechanism, `UIManager.PassthroughOnTop()`/`AllowPassthroughInput()`, is still present in the current build's decompiled code, so these were narrower fixes for other screens, not this one).
- **`2025.wiki` (build `210.22`, released July 28, 2025)**: `"Fixed a bug that caused the mod manager to close after a popup was dismissed."` and `"Fixed a bug that caused the mod manager to still receive key input while a popup was displayed."` — **closest documented analog to the UAT-06 bug class**: it is the same family of defect (a non-modern-popup UI screen's key-input state getting out of sync with a popup rendered on top of it) but for the built-in mod manager screen, not for QudUX's legacy `TextConsole` screens. Treat as related precedent, not a direct fix — it doesn't touch `Qud.UI.UIManager.PassthroughOnTop`/`GameManager.UpdateInput`, which is what actually gates our screen (see §3).
- **2026 (`index.wiki`, builds 211.33–212.17)**: no `sprite`, `tile`-customization, `popup`, or `callout` entries at all. Nothing relevant, nothing that changes the verdict below.
- Explicit statement: **no patch note in 2023–2026 documents a fix, or even an acknowledgement, of legacy-fullscreen-screen input starvation caused by a modern-UI popup rendering on top.** The bug is real in the current build's shipped code (see §3) and undocumented in patch notes.

### 2. Obsolescence verdict: **STILL VALUABLE**

- The base game has **no native way for the player to change their own character's sprite/tile or colors** in any 2023–2026 patch note found. (Contrast with UAT 4.1's recipe menu and UAT 2's tilemaker, both of which the user found genuinely superseded by native modern-UI menus — sprite customization has no such native counterpart.)
- `XRL.UI.QudUX_CharacterTileScreen` (mod source) is therefore unique functionality, not a mod-vs-native duplicate.
- Recommendation basis: keep the feature; the only real defect is the stuck-modal bug (§3), not obsolescence.

### 3. The stuck-menu bug — root cause

**CONFIRMED** mechanism (via ilspycmd decompile of `Assembly-CSharp.dll` from `D:/SteamLibrary/steamapps/common/Caves of Qud/CoQ_Data/Managed/`):

`Screens/QudUX_CharacterTileScreen.cs` `Show()` is a classic legacy `TextConsole`/`Popup` screen: `[UIView("QudUX:CharacterTile", ForceFullscreen: true, NavCategory: "Menu,Nocancelescape", UICanvas: null)]`, and its body is a blocking `while (true) { ... Keys keys = Keyboard.getvk(Options.MapDirectionsToKeypad); ... if (keys == Keys.Escape || keys == Keys.NumPad5) { ... } }` (lines 77–338, key read at line 199). `Keyboard.getvk` → `GetNextKey` blocks on `ConsoleLib.Console.Keyboard.KeyQueue` (a `CleanQueue<XRLKeyEvent>`), waiting via `KeyEvent.WaitOne(...)` until something calls `Keyboard.PushKey`/`PushMouseEvent`/`PushCommand`.

For a legacy `Menu`-category screen, **nothing pushes to that queue except `GameManager.UpdateInput()`** (decompiled `GameManager` class, no namespace). That method translates Rewired's `player.GetButtonDown(...)` presses into `Keyboard.PushKey(...)` calls, and it does so only when explicit gates pass:

```csharp
// GameManager.UpdateInput(), ~line 2467
if (ControlManager.IsLayerEnabled("Menus") && UIManager.instance.PassthroughOnTop())
{
    if (player.GetButtonDown("Accept")) { Keyboard.PushKey(new Keyboard.XRLKeyEvent(UnityEngine.KeyCode.Space)); }
    if (player.GetButtonDown("Cancel")) { Keyboard.PushKey(new Keyboard.XRLKeyEvent(UnityEngine.KeyCode.Escape)); }
    if (Input.GetMouseButtonDown(1)) { Keyboard.PushKey(UnityEngine.KeyCode.Escape); }
}
...
// ~line 2482
if (currentNavCategory != text) { SetActiveLayersForNavCategory(text); }
else
{
    if (!UIManager.instance.AllowPassthroughInput()) { return; }   // <-- early-out for the WHOLE rest of the method
    ...
    if (ControlManager.isCommandDown("Navigate Up"))    { Keyboard.PushKey(UnityEngine.KeyCode.Keypad8); }
    if (ControlManager.isCommandDown("Navigate Down"))  { Keyboard.PushKey(UnityEngine.KeyCode.Keypad2); }
    if (ControlManager.isCommandDown("Navigate Left"))  { Keyboard.PushKey(UnityEngine.KeyCode.Keypad4); }
    if (ControlManager.isCommandDown("Navigate Right")) { Keyboard.PushKey(UnityEngine.KeyCode.Keypad6); }
    ...
}
```

And the two gates it depends on, `Qud.UI.UIManager.PassthroughOnTop()` and `.AllowPassthroughInput()`:

```csharp
// Qud.UI.UIManager
public bool PassthroughOnTop()
{
    if (GameManager.Instance.CurrentGameView != null
        && GameManager.Instance.CurrentGameView.StartsWith("ModernPopup"))
    {
        return false;
    }
    ...
    if (!(instance.currentWindow == null))
    {
        return instance.currentWindow.AllowPassthroughInput();
    }
    return true;
}

public bool AllowPassthroughInput()
{
    ...
    for (int i = 0; i < windows.Count; i++)
    {
        if (windows[i].Visible && !windows[i].AllowPassthroughInput())
        {
            return false;
        }
    }
    return true;
}
```

Reading these together: the instant a modern-UI overlay is on top — either the active game view's name starts with `"ModernPopup"` (a view class `GameManager` itself registers via `_ViewData["ModernPopup*"] = new ViewInfo(WantsTileOver: false, ForceFullscreen: false, "Menu")`, `GameManager.cs` ~line 215), or any currently-visible `WindowBase` in `UIManager.windows`/`currentWindow` reports `AllowPassthroughInput() == false` — both `PassthroughOnTop()` and `AllowPassthroughInput()` go false. That makes `GameManager.UpdateInput()` either skip the Escape/Accept synthesis block entirely, or hit the `return;` at line ~2488 before it ever reaches the Keypad8/2/4/6 ("Navigate Up/Down/Left/Right") and "Take A Step" (confirm) translations. **No physical key the player presses — Escape, Numpad 5, Numpad 2/4/6/8, Enter — ever reaches `Keyboard.KeyQueue` while that condition holds**, so `QudUX_CharacterTileScreen.Show()`'s `Keyboard.getvk()` call blocks forever: the screen isn't crashed, it's starved of input. This matches the UAT description exactly ("esc/5/etc stops working — only fix is to restart the game"): nothing short of a process restart clears `currentWindow`/`CurrentGameView` state that the legacy screen has no way to touch, because the very keys that would let the player back out are the ones being withheld.

I could not find, in the decompiled code, an explicit case that *recovers* `PassthroughOnTop`/`AllowPassthroughInput` once the offending modern element self-dismisses (e.g. a timed-out toast) — whether recovery ever happens for some callout types and not others is `⚠️ unverified`; the UAT report treats it as permanently stuck for the case tested.

I did not identify the exact concrete class instance of "callout" the user saw (no class literally named `Callout` exists in `Assembly-CSharp.dll`; the closest candidates are `Qud.UI.Notification` — a plain `MonoBehaviour`, not a `WindowBase`, so it is `⚠️ unverified` whether it alone can trigger this — or any `WindowBase`-derived popup/dialog that either pushes a `"ModernPopup*"` game view or sets itself as `UIManager.currentWindow`). The **gating mechanism itself is CONFIRMED from source**; the **specific trigger UI element is HYPOTHESIS**.

**UAT 08 (scoreboard) shares the cause — CONFIRMED.** `Screens/QudUX_GameStatsScreen.cs` uses the identical pattern: `[UIView("QudUX:GameStats", ForceFullscreen: true, NavCategory: "Menu,Nocancelescape")]` (line 13) and a `while (true) { ... Keys keys = Keyboard.getvk(Options.MapDirectionsToKeypad); ... }` loop (lines 44, 78). It depends on exactly the same `GameManager.UpdateInput()` → `Keyboard.KeyQueue` pipeline gated by the same `UIManager.PassthroughOnTop()`/`AllowPassthroughInput()` checks, so the same modern-popup-on-top freeze applies for the same reason. Any other QudUX legacy `[UIView(ForceFullscreen: true, ...)]` screen using `Keyboard.getvk()` in a blocking loop is equally exposed — that includes, per the file list, `QudUX_InventoryScreen.cs`, `QudUX_RecipeSelectionScreen.cs`, `QudUX_QuickPickupSettingsScreen.cs`, `QudUX_IngredientSelectionScreen.cs`, `QudUX_GameDetailsScreen.cs`, `QudUX_BuildLibraryScreen.cs`, `QudUX_AutogetManagementScreen.cs` (not individually verified for this task — flagged for the code review as likely sharing the same class of risk, `⚠️ unverified` per-file).

### 4. Modern-UI interaction

The mod has no native modern-UI screen for sprite customization — this is a pure legacy `TextConsole` screen, always `ForceFullscreen: true`, with no `UICanvas` and no modern-UI equivalent code path. Under the planned feature-flag/UI-mode approach:
- Under **Modern UI**, this screen is exactly the failure mode described above: it can be launched (the wish route has no option gate — see UAT-06 §3 "Preconditions": `QudUX_OptionCustomSpriteMenu` does not gate it), it visually works most of the time, but it carries a live risk of an unrecoverable soft-lock any time a modern popup/notification renders while it's open. That risk is structural (input-starvation by design of `UIManager.PassthroughOnTop`), not a QudUX regression — QudUX did not write `PassthroughOnTop`/`AllowPassthroughInput`, it only inherits their behavior by using the legacy `IScreen`/`Keyboard.getvk` pattern.
- Under **Classic/legacy UI** (`Options.ModernUI == false`), `PassthroughOnTop()`'s and `AllowPassthroughInput()`'s modern-popup-specific branches are moot (no modern popups exist to set `currentWindow`/push `"ModernPopup*"`), so the screen should behave as before with no starvation risk from this cause.
- Net: sprite customization is **not itself a modern-UI feature gap** (nothing native replaces it), but its current implementation is only fully safe under Classic UI.

### 5. Recommendation

**Gate behind a UI-mode flag, keep the legacy screen for Classic UI, and treat "safe under Modern UI" as a follow-up, not a blocker for this release.**

- Short term (this build): leave `Screens/QudUX_CharacterTileScreen.cs` as-is; it is not obsolete and works when no modern popup intervenes. Do not remove it.
- Gate: read `Options.ModernUI` (already used identically elsewhere, e.g. `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs:16`, `&& !(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)`) before/at the wish entry point (`Wishes/SpriteMenu.cs`) to at minimum warn, or to block launching the legacy screen, while a modern-UI notification is on-screen (`UIManager.instance.currentWindow != null` / `GameManager.Instance.CurrentGameView.StartsWith("ModernPopup")` — both readable from mod code without engine changes) — this converts an unrecoverable freeze into a "try again in a moment" message.
- Medium term: consider trapping the specific starvation case defensively inside `QudUX_CharacterTileScreen.Show()`'s loop — e.g., detect `!UIManager.instance.PassthroughOnTop()` each iteration (poll, not block, when starved) and force-close the screen (or suppress/dismiss the offending popup) rather than calling a blocking `Keyboard.getvk()` that can never return. This does not require a native replacement API, just defensive polling against the same `UIManager` state already used for the gate above.
- Long term / "rebuild on a native API": there is no native player-facing sprite/tile picker to rebuild onto (§2) — a genuine modern-UI rebuild would mean writing a new `WindowBase`-derived Unity UGUI screen (paralleling how `Equipment`/`Inventory`/`Status`/etc. are implemented, per the `_ActiveGameView` checks in `UIManager.PassthroughOnTop`/`AllowPassthroughInput`), which is a real engineering investment, not a drop-in fix. Recommend deferring that to a dedicated UI-overhaul milestone; do it only if the gate/defensive-poll mitigation above proves insufficient in practice.
- Do not remove the feature — it is the only way to change a character's sprite/colors in this game (§2).

Files/options involved: `Screens/QudUX_CharacterTileScreen.cs` (`Show`, `UpdateSearchString`), `Screen Extenders/CharacterTileScreenExtender.cs`, `Wishes/SpriteMenu.cs`, `Options.ModernUI` / `QudUX_OptionCustomSpriteMenu`, and (read-only, engine-side, for the gate condition only) `Qud.UI.UIManager.PassthroughOnTop`/`AllowPassthroughInput`/`currentWindow`, `GameManager.CurrentGameView`/`_ViewData["ModernPopup*"]`.

### 6. Confidence + gaps

**Verified directly:**
- Mod source: `Screens/QudUX_CharacterTileScreen.cs` full loop structure, `UIView` attribute, `Keyboard.getvk` blocking read (read in full).
- Mod source: `Screens/QudUX_GameStatsScreen.cs` shares the identical `UIView`/`Keyboard.getvk` pattern (grepped, not fully read).
- Decompiled `GameManager` (no namespace), `ConsoleLib.Console.Keyboard`, `Qud.UI.UIManager`, `ControlManager`, `XRL.UI.Framework.NavigationController` from the shipped `Assembly-CSharp.dll` — quoted the exact gating chain (`UpdateInput` → `PassthroughOnTop`/`AllowPassthroughInput` → `Keyboard.KeyQueue` → `Keyboard.getvk`).
- All patch-note files (`2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki`) grepped for `sprite`, `tile`, `customiz`, `appearance`, `portrait`, `modern ui`, `popup`, `callout`, `scoreboard` — no native sprite/tile customization feature found in any year; two related-but-not-identical popup/input-focus bugfixes found and quoted (builds 207.40 and 210.22).
- Raw UAT source text for UAT 06 and UAT 08 (`qudux_UAT.md` lines 55–99), confirming the bug description and that the user linked the two as the same issue.

**Not verified (gaps, all tagged inline above too):**
- ⚠️ The exact UI element the user encountered as "a callout" — no `Callout` class exists in the assembly; candidates (`Qud.UI.Notification`, or any `WindowBase` popup) not individually confirmed to trigger `PassthroughOnTop()`/`AllowPassthroughInput()` to go false.
- ⚠️ Whether the freeze is ever self-recovering (e.g., after a toast times out) for some modern-UI element types — decompiled code doesn't show an obvious recovery path, but this wasn't exhaustively traced through all `WindowBase` subclasses.
- ⚠️ The other seven legacy `[UIView(ForceFullscreen: true, ...)]` screens listed in §3 were located by grep but not individually confirmed to share the exact same blocking-loop pattern — flagged as likely-affected, not proven per file.
- Did not use the web (no gap required it — the two local patch-note sources and the local decompiled assembly were sufficient for every claim above).

