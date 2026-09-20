# UAT 08: Enhanced Scoreboard / Detailed Stats — high-score load no longer hangs

> Read `uat/00-setup.md` first (install, log location, wish key `Ctrl+W`, save key `F5`). This script does not repeat those steps.
> **Run this script LAST.** Step 6 kills `qudux-32-test-uat-character` permanently (permadeath). Finish every other UAT script first.

## 1. What was fixed

The game's save/load API changed to be asynchronous. QudUX's high-score loader (`EnhancedScoreboard.Init()`) didn't compile against it and had to be updated to block on the new async call. In plain terms: opening the **High Scores** screen and QudUX's **Detailed Stats** add-on must load promptly, without hanging or crashing the game, whether or not you have any saved runs yet.

## 2. Feature under test + where it lives

- Vanilla screen: **Main Menu → Records / High Scores** (internal UIView `HighScores`, class `XRL.Core.Scores`).
- QudUX's addition (Harmony-patched into that screen, `Harmony Patches/Patch_XRL_Core_Scores.cs`): when the score list is non-empty, a new hint `[M - Detailed Stats]` appears near the bottom of the screen. Pressing **M** opens QudUX's own **"Games statistics"** screen (`Screens/QudUX_GameStatsScreen.cs`, internal view `QudUX:GameStats`) — this is the code path that exercises the fix (`Screen Extenders/EnhancedScoreBoard.cs:36`).
- That screen has 3 pages, cycled with **4 / 6**: **Games List**, **Level Stats**, **Death Stats**. On the Games List page, **Enter/Space** on a row opens a **Game Detail** sub-screen (`Screens/QudUX_GameDetailsScreen.cs`) showing the full end-of-game summary text for that run. **Ctrl+A** toggles whether abandoned games count. **?** shows a quick-key popup.

## 3. Preconditions

**QudUX options:** none. There is no on/off toggle for this feature in `Options.xml` — the Harmony patch is always active, so there's no "toggled off" variant to test.

**Game options** (Options menu, exact display text):
| Option | Category | Required value |
|---|---|---|
| `Enable modern UI` | UI | Either. Default **Yes**. Changes only the label you see in step 1, not the underlying screen — both routes call the same `XRL.Core.Scores.Show()` that QudUX patches. |
| `Get prompted to confirm deaths` | Debug (only visible with "Show advanced options" on) | Leave at default **No**, so the death wish in step 6 doesn't need an extra Yes/No confirmation. If it's **Yes**, just answer **Yes** to the popup that appears. |

**Save state:** `qudux-32-test-uat-character`, freshly loaded per 00-setup (no prior death this session — this is the last script, so the local scoreboard should still be empty going into step 3).

## 4. Setup

No special setup beyond 00-setup's baseline load. The wishes used below (`gamestats menu`, `die`) are typed into the wish prompt (`Ctrl+W` per 00-setup).

## 5. Test steps

**Step 1 — Confirm the menu label**
Action: At the main menu, look at the entry that opens the scoreboard (do not select it yet).
Expected: With `Enable modern UI = Yes` (default) it reads **Records**. With it set to **No**, it reads **High Scores**. Either way its shortcut key is **H** (confirmed in `Qud.UI.MainMenu.LeftOptions` and the classic main-menu loop in `XRL.Core.XRLCore`).
Evidence: 📸 `UAT-08-01-main-menu-label.png` — main menu with that entry and its shortcut legible.

**Step 2 — Open it and confirm no freeze**
Action: Press **H** (or click the entry).
Expected: The screen renders within about a second — no hang, no beachball, no crash. It shows either "Local Scores" listing or the empty-state message (step 3).
Evidence: 🎥 `UAT-08-02-open-no-freeze.mp4` — start recording just before pressing H, stop once the screen is fully drawn. Timestamp of key-press to fully-rendered screen must be visible or inferable (e.g. include a visible on-screen clock or narrate the count).

**Step 3 — Empty-scoreboard state (vanilla screen)**
Action: With no prior scores, observe the screen from step 2.
Expected: Text reads exactly **"No high scores!"**. No `[M - Detailed Stats]` hint is shown at this point — that hint only renders when there's at least one score (it's inside the same code branch as the score list). Press **Esc** or **NumPad 5**: returns cleanly to the main menu.
Evidence: 📸 `UAT-08-03-empty-vanilla.png` — the "No high scores!" message visible.
⚠️ If this profile already has old Caves of Qud scores from unrelated play, this screen won't be empty — skip to Step 5 and just note it in the results table.

**Step 4 — Exercise the fix directly against an empty scoreboard**
Since the `[M - Detailed Stats]` hint doesn't appear on an empty list, reach QudUX's screen directly to confirm `EnhancedScoreboard.Init()` handles zero scores without crashing.
Action: Open the wish prompt, type `gamestats menu`, Enter. (Matches `Wishes/GameStatsMenu.cs`'s regex `(?:qudux ?)?gamestats ?menu`, case-insensitive.)
Expected: A bordered **"Games statistics"** screen opens immediately (no freeze/crash). Top-right page indicator reads **1/3**. Table headers **Name / Date / Score / Lvl / Killed by** are visible with zero rows. Bottom-left reads **"0 games"**.
Evidence: 📸 `UAT-08-04-empty-detailed-stats.png`.
Press **Esc**/**NumPad 5** to close.

**Step 5 — (Optional but recommended) Note the quick-key popup**
Action: While still on the Games List page (reopen via the same wish if you closed it), press **?**.
Expected: A popup titled "Statistics quick keys" lists 8/2/9/3/Ctrl+A/Enter bindings. Esc or any key closes it without error.
Evidence: 📸 `UAT-08-05-quick-keys-popup.png`.

**Step 6 — Kill the character to create a score entry**
⚠️ Irreversible — do this only after every other UAT script is done.
Action: Open the wish prompt, type `die`, Enter. If a "DEBUG: Do you really want to die?" popup appears (only happens if the Debug option above was left at Yes), choose **Yes**.
Expected: The character dies immediately (`The.Player.Die(The.Player, "wished dead")`); the game plays its normal death/epilogue sequence.
Evidence: 🎥 `UAT-08-06-death-wish.mp4` — from typing the wish to the death sequence starting.

**Step 7 — Return to the main menu**
Action: Click/press through any epilogue or tombstone screens.
Expected: You land back on the main menu (run has ended). ⚠️ unverified — confirm in game: exact number of screens/prompts before you reach the main menu.
Evidence: 📸 `UAT-08-07-main-menu-after-death.png`.

**Step 8 — Reopen Records/High Scores with real data, confirm no freeze**
Action: Press **H** again.
Expected: Loads within about a second (same no-freeze check as step 2, now with a populated scoreboard — this is the more realistic exercise of `Scoreboard2.Load().GetAwaiter().GetResult()`). One row is listed for `qudux-32-test-uat-character`. The `[M - Detailed Stats]` hint is now visible near the bottom.
Evidence: 🎥 `UAT-08-08-scores-with-entry.mp4`.

**Step 9 — Open Detailed Stats with the new entry**
Action: Press **M**.
Expected: Games List page shows one row: Name `qudux-32-test-uat-character`, today's date (`yyyy-MM-dd`), a Score value, your character's Level, and a "Killed by" value derived from the death text (expect it to reference "wished dead" — ⚠️ unverified exact wording, QudUX derives it by text-parsing the game's death summary). Footer reads "1 games".
Evidence: 📸 `UAT-08-09-games-list-entry.png`.

**Step 10 — Page through Level Stats and Death Stats**
Action: Press **6** twice (Games List → Level Stats → Death Stats); page indicator should read 2/3 then 3/3.
Expected: Level Stats shows one row for your death level at 100.0%. Death Stats shows one row for the death cause at 100.0%.
Evidence: 📸 `UAT-08-10-level-and-death-stats.png` (both pages, one screenshot each is fine — take two if needed, same filename with `-a`/`-b` suffix).

**Step 11 — Open Game Detail for the new entry**
Action: Press **4** twice to return to Games List, then **Enter** (or **Space**) on the single row.
Expected: A bordered "Game Detail" screen opens showing the full multi-line end-of-game summary text for that run (character line, date/time of death, cause, level, turns). No crash.
Evidence: 📸 `UAT-08-11-game-detail.png`.

**Step 12 — Unwind via Escape at every level**
Action: Press **Esc** three times: Game Detail → Detailed Stats (Games List) → vanilla High Scores → main menu.
Expected: Each Escape returns one level up cleanly, no errors, ends back at the main menu.
Evidence: 📸 `UAT-08-12-back-to-main-menu.png`.

## 6. Edge / negative cases

- **Do not** press Enter/Space on the Games List page while it has zero rows (i.e., during Step 4). `QudUX_GameStatsScreen.Show()` indexes `ScoreList[0]` unconditionally on Enter, which throws on an empty list. This is a separate, out-of-scope defect from issue #08 (which is only about the async `Load()` call) — if you hit it by accident, log it in "Problems found" below but don't mark this whole script FAIL over it.
- **Ctrl+A** on the Games List page toggles counting abandoned runs. With only one non-abandoned entry (`wished dead` isn't an "abandoned" death), the count should stay at 1 either way — confirms the toggle doesn't crash, not that it changes anything visible here.
- No QudUX option gates this feature, so there's no "feature absent when option is off" case to test.

## 7. Log check

After finishing (and before closing the game — copy `Player.log` per 00-setup step 2), search for:

```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "QudUX.*Scores|QudUX.*Failed to load HighScores|QudUX.*Game Stats|EnhancedScoreboard|Scoreboard2|QudUX_GameStatsScreen|Exception"
```

FAIL if any of these appear:
- `QudUX: (Error) Failed to load HighScores data [...]` — the patched `Load()` call threw.
- `QudUX: (Error) Encountered an exception while showing the Game Stats menu [...]` — the wish/M-key path crashed.
- Any `Exception` whose stack trace mentions `EnhancedScoreboard`, `QudUX_GameStatsScreen`, `QudUX_GameDetailsScreen`, or `Scoreboard2`.
- The startup patch line for this patch reads **"Failed."** instead of **"Patched successfully."** — search for `Scores...` in the log (from `PatchHelpers.LogPatchResult("Scores", ...)`); "Failed" means the M-key hook never installed and the whole Detailed Stats feature is unreachable regardless of what you saw on screen.

Not a fail by itself: `QudUX: (Error) Unexpected issue parsing High Score entry [...]` is a soft, per-field parse failure in `EnhancedScoreEntry`'s text parsing — only treat it as a FAIL if it corresponds to the entry you just created in Step 6 (e.g. Level/Killed-by ends up blank in Step 9/10 for that row).

## 8. Fail indicators

- Game freezes/hangs (no input response) opening Records/High Scores or Detailed Stats.
- Crash to desktop or an unhandled-exception dialog.
- `[M - Detailed Stats]` hint never appears once a score exists, or **M** does nothing.
- After death, no new row appears in either the vanilla list or Detailed Stats.
- Blank/garbled columns in Games List, Level Stats, or Death Stats.
- Page indicator doesn't update, or 4/6 don't cycle pages.
- Any of the log-check FAIL conditions in section 7.

## 9. Results

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
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
| 12 | | | |
| Log check | | | |

### Problems found

For each problem:
```
Steps to reproduce:
Expected:
Actual:
Evidence: (file in qudux-mod-docs/uat/evidence/08/)
Player.log excerpt:
```

## Patch-note correlation (2023–2026)

Sources grepped: `patchnotes/2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026, builds 211.33–212.17). Decompiled: `XRL.Core.Scores`, `ConsoleLib.Console.TextConsole`, `ConsoleLib.Console.Keyboard`, `GameManager` (top-level namespace), `Qud.UI.UIManager`, `Qud.UI.WindowBase`. Mod source read: `Screen Extenders/EnhancedScoreBoard.cs`, `Screens/QudUX_GameStatsScreen.cs`, `Harmony Patches/Patch_XRL_Core_Scores.cs`, `Wishes/GameStatsMenu.cs`. No web fetches were needed — everything was answerable from local sources.

### 1. Relevant patch entries

**The modern-UI rewrite never touched this screen.** `2024.wiki`, build `207.31: Spring Molting beta` (Released May 10, 2024):
> "We added an entire new UI. (ie, we completed work on the modern UI, for folks who've been following our progress)... There are too many changes to fully document, but here's a partial list of new screens: trade, quests, message log, character sheet, skills, equipment & inventory, tinkering, game summary, every journal tab, status effects, reputation, world generation, interact nearby, books, in-game terminals."

High Scores/Records is conspicuously **absent** from that "partial list of new screens." **Cause of what the UAT saw** (the "showing old ui" part): I confirmed by decompiling `XRL.Core.Scores.Show()` that it is still pure `TextConsole`/`ScreenBuffer` code today — it has one `if (GameManager.Instance.ModernUI)` branch that only tweaks internal list building, never swaps in a modern-UI canvas. So "shows the legacy screen" isn't a QudUX regression; the vanilla game itself never migrated this screen off the console renderer, in any 2023–2026 patch I could find.

**Native "Records" name, still the legacy screen underneath.** `2024.wiki`, `1.0.1 (build 209.41)` (Released December 18, 2024):
> "Fixed a bug that caused loss of input focus when viewing the Records screen without any records."
> "Fixed a bug that caused the wrong endgame state to be logged in the Records screen."
> "Fixed a bug that caused endgame Records to be saved twice."

This is the earliest patch-note evidence I found of the "Records" name being used for this screen (matches the UAT's own finding, from `Qud.UI.MainMenu.LeftOptions`, that the label is "Records" when modern UI is on, "High Scores" when off). **Related, not causal**: these three are separate, already-fixed bugs in the same screen family (input focus loss, endgame logging, double-save) — not the same defect as the UAT's stuck-screen freeze, but they show Freehold has hit input-focus bugs on this exact screen before. I found **no patch note in 2023–2026 documenting an actual UI-framework migration of Records/High Scores to a modern-UI canvas** — the Dec 2024 fixes are bugfixes to the same legacy screen, not a rewrite.

**Unrelated-but-adjacent**: `2023.wiki`, build in the `206.20 beta` bugfix block:
> "Fixed a bug that caused leaderboard scores to be truncated on low DPI displays."

This is about the Steam online-leaderboard display, not the local `Scores.Show()` screen QudUX patches — unrelated to the UAT finding, noted for completeness since it matched the "scoreboard" keyword.

**Keyboard-layout entries** (for the German "?" problem): two hits, both **related but not causal**:
- `2023.wiki`, build `204.56` (Released February 3, 2023): "Fixed several issues with MacOS and non-English keyboard layouts." — too vague/early to be about this specific hotkey, predates the modern UI entirely.
- `2025.wiki`, `1.0.4 (build 210.10)` (Released May 22, 2025): "Fixed a bug that caused some control binds on non-English keyboard layouts to render incorrectly."

Neither explains the German-keyboard bug directly, because — CONFIRMED by reading `Screens/QudUX_GameStatsScreen.cs:94-97` — the `?` quick-keys popup is gated by a **hardcoded raw key check**, not a rebindable control:
```csharp
if (keys == Keys.OemQuestion)
{
    ShowQuickKeys();
}
```
This never goes through `ControlManager`'s bind-resolution system at all, so Freehold's control-bind/layout fixes (which patch the rebindable-command path) can't reach it regardless of game version. The actual mechanism (CONFIRMED via decompile of `ConsoleLib.Console.Keyboard.InitKeymap()`):
```csharp
Keymap.Add(UnityEngine.KeyCode.Question, Keys.OemQuestion);
Keymap.Add(UnityEngine.KeyCode.Slash, Keys.OemQuestion);
```
Unity's legacy `KeyCode.Slash`/`KeyCode.Question` correspond to a fixed physical key position (the US-QWERTY `/` key), not to the character actually produced by the OS layout. On a German QWERTZ layout, `?` is produced by Shift+`ß`, a different physical key — so it's very unlikely to ever raise `KeyCode.Slash`/`Question` in Unity, meaning `Keys.OemQuestion` never fires. ⚠️ unverified: I did not test on an actual German keyboard or find a decompiled table of what KeyCode `ß`/Shift+`ß` raises — this is the standard, well-documented behavior of Unity's legacy Input class for symbol keys, but I could not confirm the exact KeyCode German `ß` maps to from this codebase alone.

### 2. Obsolescence verdict: **STILL VALUABLE**

I found no patch note anywhere in 2023–2026, and no code in decompiled `XRL.Core.Scores.Show()`, indicating the native game ever added: a sortable/tabulated games list, per-level or per-death-cause aggregate breakdowns, an abandoned-games filter, or a per-game full detail view. The native screen (per the decompiled `Show()` method) presents a flat "Local Scores" / "Daily" / "Daily (friends)" tabbed list built from raw `Scoreboard.Scores` — there's no evidence of columns, sorting, or statistics.

QudUX's `EnhancedScoreboard`/`QudUX_GameStatsScreen` adds, none of which exist natively:
- A tabulated Games List (Name/Date/Score/Lvl/Killed by), sorted by score.
- Level Stats page: death-level distribution with % of total.
- Death Stats page: death-cause distribution with % of total.
- Ctrl+A toggle to include/exclude abandoned runs from the stats.
- A "Game Detail" drill-down screen showing the full end-of-game summary text per entry.

Removing this feature would lose all of the above — the base game has never shipped an equivalent aggregate-stats view in any patch through build 212.17 (March 2026).

### 3. Stuck-screen bug — root cause: CONFIRMED

Both the vanilla `XRL.Core.Scores.Show()` and QudUX's `QudUX_GameStatsScreen.Show()` (`Screens/QudUX_GameStatsScreen.cs:44-152`) run the same pattern: an unbounded `while (true)` loop that blocks on `Keyboard.getvk(...)` (default `wait: true`) every iteration, with no cooperative check for "is a modern-UI overlay currently intercepting input."

Traced the actual stall, end to end, via decompile:

1. `Keyboard.getvk()` → `GetNextKey()` (`ConsoleLib.Console.Keyboard`) spins in `while(true) { ...; KeyEvent.WaitOne(20); ... PopKey(); }` — it never returns until `PopKey()` yields a queued key. No timeout, no escape valve.
2. The only producer for that queue is `GameManager.pushKeyEvents()`, called from `GameManager.OnGUI()`. But `OnGUI()` gates it:
   ```csharp
   if (UIManager.instance == null || !UIManager.instance.AllowPassthroughInput())
   {
       return;
   }
   ```
   i.e. if this returns false, `pushKeyEvents()` (and thus `Keyboard.PushKey`) is never called for that frame — nothing gets queued, at all, for any screen.
3. `Qud.UI.UIManager.AllowPassthroughInput()`:
   ```csharp
   for (int i = 0; i < windows.Count; i++)
   {
       if (windows[i].Visible && !windows[i].AllowPassthroughInput())
       {
           return false;
       }
   }
   return true;
   ```
   Any visible modern-UI window that reports `false` blocks input passthrough for **everything** — not just its own screen.
4. `Qud.UI.WindowBase.AllowPassthroughInput()` (the base class every modern-UI window inherits, unless it explicitly overrides):
   ```csharp
   public virtual bool AllowPassthroughInput()
   {
       return false;
   }
   ```
   Default is `false`. Most modern-UI windows never override this to `true` (only a short allowlist of `_ActiveGameView` names like Equipment/Inventory/Status/Factions/Quests/Journal/Tinkering/SkillsAndPowers get a `true` from the earlier static overload).

Net effect: whenever any modern-UI window/modal becomes `Visible` on top of a legacy `TextConsole` screen — and doesn't happen to be one of that small allowlisted set — `GameManager.OnGUI()` stops feeding `Keyboard.KeyQueue` entirely. The blocking `while(true) + Keyboard.getvk()` loop in both `QudUX_GameStatsScreen.Show()` and vanilla `Scores.Show()` then spins forever with zero input, including Esc/NumPad5, matching exactly "gets stuck... only fix is restart." This is a **generic vanilla-engine behavior**, not something QudUX introduced — the vanilla High Scores screen has the identical exposure, since it uses the identical blocking-loop pattern.

I could not identify, from the UAT screenshots or code, exactly *which* modern-UI element (achievement toast, mod-alert popup, tooltip, etc.) was visible in the user's repro — the general mechanism above is CONFIRMED, but the specific trigger in this session is ⚠️ unverified / HYPOTHESIS.

**Does UAT 06 (sprite menu) share this cause?** I did not read the sprite-menu's own source in this pass (another agent is covering it), so I state this as my own inference, not a copy of theirs: the symptom is described identically ("gets stuck if a callout renders on top," "only fix is restart"). Given the mechanism above is a property of the **shared vanilla input-dispatch code** (`GameManager.OnGUI` → `UIManager.AllowPassthroughInput` → `WindowBase.AllowPassthroughInput`) rather than anything specific to the Scores screen, it would affect *any* screen using the same blocking `Keyboard.getvk()` pattern — which very likely includes the sprite menu if it's also a `TextConsole`-based legacy screen. Marked HYPOTHESIS pending confirmation that the sprite menu uses the same blocking-loop pattern.

### 4. Modern-UI interaction

The planned feature-flag approach (load different mod behavior depending on the player's UI setting) **does not apply cleanly here**, because the vanilla screen itself never branches on modern UI in a meaningful way. Confirmed: `XRL.Core.Scores.Show()` is TextConsole-based **unconditionally** — `Enable modern UI = Yes/No` only changes the main-menu button's label (via `Qud.UI.MainMenu.LeftOptions`, per the UAT's own step 1), not which code path renders the screen. So:
- Gating `Patch_XRL_Core_Scores.cs` off when modern UI is enabled would **not** protect against the freeze — the vanilla screen is exactly as exposed to the same `AllowPassthroughInput`/`WindowBase` stall whether or not QudUX's patch is active, since the freeze lives in vanilla `Scores.Show()`'s own blocking loop, not in QudUX's transpiler additions.
- Gating it off would only cost the player the `[M - Detailed Stats]` hint and `QudUX_GameStatsScreen` under modern UI, for zero freeze-prevention benefit.
- The German "?" bug is independent of modern UI entirely — it's a raw `KeyCode` mapping problem in QudUX's own popup trigger, not something the modern-UI flag would touch.

### 5. Recommendation: **keep as is, with two independent, small QudUX-side fixes**

- **Keep the feature active regardless of UI mode.** A modern-UI feature-flag gate doesn't fix or reduce the freeze (see §4), and the feature is still valuable (see §2), so gating it off under modern UI is a net loss with no upside.
- **Fix the German-keyboard "?" bug independently**: in `Screens/QudUX_GameStatsScreen.cs`'s `ShowQuickKeys()` trigger, stop keying off the raw, layout-dependent `Keys.OemQuestion` and instead check the resolved character (`Keyboard` already builds `uccharmap`/`lccharmap` tables, e.g. `uccharmap.Add(UnityEngine.KeyCode.Slash, '?')`) or route the binding through the game's control-bind system so it resolves per-layout the way rebindable commands do. This is a self-contained, low-risk fix scoped to QudUX's own code.
- **Optional, higher-effort mitigation for the freeze**: since the root cause lives in vanilla's blocking-loop pattern (shared by `Scores.Show()` itself), QudUX cannot fix vanilla's copy of the bug. QudUX *could* harden its own `QudUX_GameStatsScreen.Show()` loop by polling `Qud.UI.UIManager.instance?.AllowPassthroughInput()` (if a public/reflectable path exists) and forcing an exit/redraw if a modern-UI window is blocking passthrough for too long — but this only protects QudUX's own screen, not the vanilla one the player reaches first, so it would not fully resolve the UAT's reported symptom. Given that scope limit, I'd treat this as optional rather than required.
- No case here for rebuilding on a native API — there isn't one; the native Records screen has no equivalent aggregate-stats surface to build on top of (see §2).
- **Files/options involved**: `Screens/QudUX_GameStatsScreen.cs` (both the input loop and `ShowQuickKeys()`), `Harmony Patches/Patch_XRL_Core_Scores.cs` (the transpiler that adds the `[M - Detailed Stats]` hint — leave active, do not gate), `Screen Extenders/EnhancedScoreBoard.cs` (already fixed for the async `Scoreboard2.Load()` API per the UAT's step 0 — no further action needed there), `Wishes/GameStatsMenu.cs` (unaffected).

### 6. Confidence + gaps

**Verified directly (code-traced, high confidence):**
- `XRL.Core.Scores.Show()` is still pure `TextConsole`/`ScreenBuffer` code today (decompiled), confirming "shows old UI" is inherited from vanilla, not a QudUX defect.
- The full input-stall chain: `QudUX_GameStatsScreen.Show()`/`Scores.Show()` blocking loop → `Keyboard.getvk` → `GetNextKey` → `Keyboard.KeyQueue` fed only by `GameManager.pushKeyEvents()` → gated by `UIManager.AllowPassthroughInput()` → `WindowBase.AllowPassthroughInput()` defaults to `false`. All five links read directly from decompiled `Assembly-CSharp.dll`.
- QudUX's `?` quick-keys check is a hardcoded `Keys.OemQuestion` comparison bypassing the control-bind system entirely (read from mod source).
- `Scoreboard2.Load()` is genuinely an async (`Task<Scoreboard2>`)-returning API today, matching the UAT's description of the fix.
- No patch note 2023–2026 documents a modern-UI rewrite of the Records/High Scores screen, nor a native aggregate-stats feature, across all four wiki files.

**Gaps / could not verify:**
- Which specific modern-UI element triggers the freeze in the user's actual repro session — not identifiable from code alone.
- Whether the sprite menu (UAT 06) uses the same blocking `Keyboard.getvk` loop pattern — I did not read its source in this pass; flagged as HYPOTHESIS.
- The exact Unity `KeyCode` produced by a German `ß`/Shift+`ß` press — no decompiled table or live test available; inferred from Unity's well-known legacy-Input behavior only.
- No patch note explicitly documents the underlying game's save/load API becoming asynchronous (the UAT's stated root cause for the original hang) — this is only visible in the decompiled `Task<Scoreboard2>` signature, not called out in any Steam/itch.io patch note text I could find; the exact build/date this landed in is unknown from these sources.
- Did not confirm column/sort behavior of the native populated "Local Scores" list beyond what `Scores.Show()`'s early setup code shows (didn't decompile the full populated-state rendering branch), though the complete absence of any "stats"/"aggregate" patch-note keyword across 2023–2026 makes it very unlikely the native screen gained tabulation or filtering since.
