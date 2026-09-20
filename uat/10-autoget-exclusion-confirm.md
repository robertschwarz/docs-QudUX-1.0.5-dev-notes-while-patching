# UAT 10: Auto-pickup exclusion confirmation popup

> Read `uat/00-setup.md` first (install, keys, log, evidence conventions). Not repeated here.

## What was fixed

The "disable auto-pickup for this item" confirmation had a broken popup call (compile error on 1.0.5) that a naive fix would have left permanently silent. The corrected version restores the original **UI notification sound** on the popup and keeps it un-escapable, matching pre-1.0.5 behavior.

## Feature under test

- **File:** `Parts and Effects/QudUX_AutogetHelper.cs` (~line 91), reached through `XRL.World.OwnerGetInventoryActionsEvent` / `InventoryActionEvent`.
- **UI path:** Inventory screen → highlight an auto-gettable item already in your inventory → press `Space` (twiddle) to open its action list → choose **"Disable auto-pickup for this item"**. That fires the `QudUX_DisableItemAutoget` command, which (the first time, ever, across all characters) shows a Yes/No popup before applying the exclusion.
- Re-enabling a single item from inventory uses the mirrored action **"Re-enable auto-pickup for this item"** (no popup). Bulk re-enabling uses the QudUX wish menu (see Step 8).

## Preconditions

| Setting | Location | Required value |
|---|---|---|
| `Use item interaction menu to disable auto-pickup for specific items` | Options → **QudUX** category | **Yes** (this is also the default) |
| `Show advanced options` | Options → **App Settings** category | **Yes** (only needed so the next row is visible/editable — it does not gate the feature itself) |
| `Autoget copper, silver, and gold nuggets` | Options → **Autoget** category (hidden unless Advanced Options is Yes) | **Yes** (this is the game's own auto-pickup toggle for the test item; default Yes) |

Save state: fresh baseline save of `qudux-32-test-uat-character` per `00-setup.md` §4.

## Setup

1. **Reset the "shown once" flag** (it's stored globally, not per-save, so a prior test run or prior play session can suppress the popup). With the game closed, open:
   `%USERPROFILE%\AppData\LocalLow\Freehold Games\CavesOfQud\Mods\QudUX_v2\QudUX_AutogetSettings.json`
   Delete the file entirely (or, if you want to preserve other entries, delete just the `"Metadata:InfoboxWasShown":"Yes"` line and any `"ShouldAutoget:..."` lines for items you'll use below). If the file doesn't exist yet, there's nothing to do — the flag is already unset.
2. Launch the game, load `qudux-32-test-uat-character`.
3. Confirm the three options in the Preconditions table above, adjusting as needed.
4. 📸 `UAT-10-00-options.png`: Options screen showing `Use item interaction menu to disable auto-pickup for specific items` = Yes and `Autoget copper, silver, and gold nuggets` = Yes in the same or consecutive shots.
5. Open the wish prompt (`Ctrl+W`) and wish `Copper Nugget` to spawn one. Step onto its tile if it doesn't auto-pickup immediately (it should, since nugget autoget is on and not yet excluded).
6. `F5` to save this as your working checkpoint for this script.

## Test steps

1. **Confirm baseline autoget works.**
   Action: Check your inventory (`I`) for the copper nugget you just picked up.
   Expected: It's there — proves the item is currently auto-getting normally, before any exclusion exists.
   Evidence: 📸 `UAT-10-01-nugget-in-inventory.png` — inventory list showing the copper nugget.

2. **Trigger the confirmation popup.**
   Action: In the inventory (`I`), highlight the copper nugget entry, press `Space` to open its action list, and choose **"Disable auto-pickup for this item"**.
   Expected: A Yes/No popup appears reading exactly:
   `Disabling auto-pickup for copper nuggets.` / `Changes to auto-pickup preferences will apply to ALL of your characters. If you proceed, this message will not be shown again.` / `Proceed?`
   — and the game's standard UI notification sound plays at the moment it appears (this is the regression the fix restores).
   Evidence: 🎥 `UAT-10-02-popup-appears.mp4` — **with audio enabled**, capturing the moment the popup opens and the chime with it. Full popup text must be readable in frame.

3. **Escape must not close it.**
   Action: With the popup still open, press `Esc` (and try `Space`/`5` too, since those normally cancel other menus).
   Expected: Nothing happens — the popup stays open, no dialog result is registered. This is the `AllowEscape: false` argument in the fix.
   Evidence: 📸 `UAT-10-03-escape-blocked.png` — popup still visible after pressing Esc.

4. **Answer No.**
   Action: Select **No**.
   Expected: Popup closes. The exclusion is NOT applied. Drop the nugget (`Ctrl+D` or the inventory drop action), step off its tile and back onto it.
   Expected result: it auto-picks up again with no popup and no interruption.
   Evidence: 📸 `UAT-10-04-no-still-autogets.png` — nugget back in inventory / pickup message in the log after answering No.

5. **Popup reappears after No.**
   Action: Repeat step 2 (inventory → `Space` on the nugget → "Disable auto-pickup for this item").
   Expected: The popup appears **again** — answering No does not set the "shown once" flag, only Yes does (see source: the flag is only written inside the `Yes` branch).
   Evidence: 📸 `UAT-10-05-popup-again-after-no.png`.

6. **Answer Yes.**
   Action: This time select **Yes**.
   Expected: Popup closes. Drop the nugget, step off and back onto its tile.
   Expected result: it is **not** auto-picked up this time (it stays on the ground; you must pick it up manually, e.g. with `g`/get). Re-open inventory-action list on it (once picked up manually) and confirm the action now reads **"Re-enable auto-pickup for this item"**.
   Evidence: 🎥 `UAT-10-06-yes-blocks-autoget.mp4` — walking onto the nugget's tile and it staying on the ground, then the inventory action list showing "Re-enable auto-pickup for this item".

7. **Popup does not reappear for a different item.**
   Action: Wish (`Ctrl+W`) `Silver Nugget`, let it auto-pickup, open inventory, `Space` on the silver nugget, choose "Disable auto-pickup for this item".
   Expected: **No popup this time** — the exclusion is applied to the silver nugget immediately and silently, because the global "shown once" flag was set in Step 6.
   Evidence: 📸 `UAT-10-07-no-popup-second-item.png` — inventory action list immediately showing "Re-enable auto-pickup for this item" for the silver nugget with no popup having appeared.

8. **Re-enable via the QudUX auto-pickup menu wish.**
   Action: Open the wish prompt (`Ctrl+W`) and wish `autopickup menu` (also accepts `QudUX autopickup menu`). On the **"Auto-pickup Exclusions"** screen, confirm both `copper nugget` and `silver nugget` are listed. Select one and press `Space`/`Enter` to remove it, then press `R` to remove all remaining.
   Expected: List empties (shows "You haven't disabled auto-pickup for any items"). Drop the copper nugget, step off and back onto it.
   Expected result: it auto-picks up again, confirming the exclusion was cleared.
   Evidence: 📸 `UAT-10-08a-management-screen.png` (list with both nuggets before removal) + 📸 `UAT-10-08b-reenabled-autoget.png` (nugget auto-picked up again after clearing).

## Edge / negative cases

- **QudUX option off:** Set `Use item interaction menu to disable auto-pickup for specific items` = No in Options. Confirm the "Disable auto-pickup for this item" action no longer appears in the inventory action list at all for any item (`QudUX_AutogetHelper.HandleEvent(OwnerGetInventoryActionsEvent)` bails out early when the option is off). Turn the option back on afterward. Evidence: 📸 `UAT-10-09-option-off-no-action.png`.
- **Wishing the menu with the option off:** With the option off, wish `autopickup menu` again — expect a popup stating you've disabled the option and must re-enable it to use the menu, instead of the management screen opening.

## Log check

After finishing, per `00-setup.md` §2, search `Player.log`:

```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "Exception|QudUX_AutogetHelper|QudUX_AutogetManagementScreen|ShowYesNo|AutopickupMenu"
```

FAIL if you see:
- Any `Exception` with `QudUX_AutogetHelper`, `QudUX_AutogetManagementScreen`, or `AutopickupMenu` in the stack.
- A `Failed to parse ... QudUX_AutogetSettings.json` error from `NameValueBag.Load` (means the JSON file got corrupted, e.g. by a manual edit in Setup step 1 — fix and re-copy the log).

## Fail indicators

- Popup never appears in Step 2, or appears with no sound / wrong or truncated text.
- `Esc` (or any other key) closes the popup in Step 3 without registering Yes or No.
- Item stops auto-getting after answering **No** (Step 4), or keeps auto-getting after answering **Yes** (Step 6).
- Popup does **not** reappear in Step 5, or **does** reappear in Step 7 (global flag not behaving as global/one-shot).
- "Auto-pickup Exclusions" screen (Step 8) doesn't list the excluded items, or removing them doesn't restore autoget.
- Any exception in `Player.log` per above.

## Results

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
| 1 – baseline autoget | | | |
| 2 – popup + sound | | | |
| 3 – escape blocked | | | |
| 4 – No keeps autoget | | | |
| 5 – popup reappears after No | | | |
| 6 – Yes blocks autoget | | | |
| 7 – no popup for 2nd item | | | |
| 8 – re-enable via wish menu | | | |
| Edge – option off hides action | | | |

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

## Patch-note correlation (2023–2026)

### 1. Native autoget timeline

Every 2023–2026 patch-note entry matching `autoget|auto-pickup|pick.?up|nugget|trade good`, in chronological order, quoted verbatim from the local wikitext dumps (`2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` for 2026).

| Build | Date | Entry |
|---|---|---|
| 204.53 | Jan 27, 2023 | "Any time the player is forced to drop an item, that item is now considered a 'special item' until the next time the player or a companion of the player picks it up, meaning autoget will pick it up if 'Auto-pickup special items' is on." |
| 204.53 | Jan 27, 2023 | "The 'Auto-pickup ammo' option no longer strips energy cells out of devices." |
| 204.56 | Feb 3, 2023 | "Items unintentionally dropped by your companions are now auto-picked-up when the 'Auto-pickup special items' option is enabled." |
| 206.26 | Nov 15, 2023 | "Added an autoget option 'Exclude liquid in containers you've dropped from auto-collect'." (→ ships as `OptionAutogetDroppedLiquid`, inverted framing: "Autoget liquids from containers you've dropped") |
| 206.69 | Mar 15, 2024 | "Fixed a bug that caused the 'Auto-pickup of zero weight items' but skip picking up items with fractional weight." [sic, wiki typo for "option"] |
| **207.31** "Spring Molting beta" | **May 10, 2024** | "We added an entire new UI. (ie, we completed work on the modern UI...) More polish will be coming... but all the functionality is there." — **this is the Modern UI general-availability build**, not an autoget note per se, but it is the root dependency for the bug in §4 below. |
| 207.89 | Aug 9, 2024 | "Disarmed mines originally created by the player are now autogettable." |
| 207.97 | Aug 23, 2024 | "Objects are now re-flagged for autoget after being thrown." |
| *(none found in 2025.wiki)* | — | Grep of `2025.wiki` for autoget/pick-up/nugget/trade-good returned no hits. Only unrelated "modern UI" font-fallback entries exist that year. |
| 211.36 (beta) | Mar 1, 2026 | "Autoget now ignores nuggets if the 'Autoget nuggets' option is disabled, even if 'Autoget trade goods' is enabled." |

No entry in any year mentions a per-item / per-blueprint exclusion, an interaction-menu toggle, or an exclusions-management screen. Native changes are exclusively: new global category options, and precedence/bugfixes between existing category options.

### Native autoget APIs and options in 1.0.5 (decompiled)

Decompiled `XRL.World.GameObject` (Assembly-CSharp.dll, via ilspycmd):

```csharp
public bool CanAutoget(bool takeableOnly = true)
{
    if (Physics == null) return false;
    if (Physics.Owner != null) return false;
    if (!Physics.IsReal) return false;
    if (IsTemporary) return false;
    if (FungalVisionary.VisionLevel <= 0 && HasPart<FungalVision>()) return false;
    if (!PhaseMatches(XRL.The.Player)) return false;
    if (IsHidden) return false;
    if (takeableOnly && !IsTakeable()) return false;
    if (HasTagOrProperty("NoAutoget")) return false;      // <- static per-blueprint escape hatch (see below)
    if (HasIntProperty("DroppedByPlayer")) return false;  // <- why QudUX_AutogetHelper toggles this property off/on
    return true;
}

public bool ShouldAutoget()
{
    if (!CanAutoget()) return false;
    if (Options.AutogetSpecialItems && IsSpecialItem()) return true;
    if (HasTagOrProperty("Nugget")) return Options.AutogetNuggets;   // <- nugget check now short-circuits BEFORE the Trade Goods category check
    if (GetInventoryCategory() switch {
        "Trade Goods" => Options.AutogetTradeGoods,
        "Food" => Options.AutogetFood,
        "Books" => Options.AutogetBooks,
        _ => false,
    }) return true;
    // ... AutogetFreshWater (weight<=1), AutogetZeroWeight, AutogetScrap
}
```

This confirms the March 1 2026 patch note is not just documentation — the shipped 1.0.5 code puts the `HasTagOrProperty("Nugget") => Options.AutogetNuggets` branch **ahead of** the `"Trade Goods" => Options.AutogetTradeGoods` branch, so a nugget's category ("Trade Goods") is never consulted; `AutogetNuggets` alone decides it, exactly as the patch note states.

`Options.xml` (StreamingAssets/Base) — every native autoget option, all `Category="Autoget"`, all gated on `Requires="OptionShowAdvancedOptions==Yes"`:

| Option ID | Display text | Default |
|---|---|---|
| `OptionAutogetAmmo` | Autoget ammo | Yes |
| `OptionAutogetPrimitiveAmmo` | Autoget primitive ammo | No |
| `OptionAutogetNuggets` | Autoget copper, silver, and gold nuggets | Yes |
| `OptionAutogetTradeGoods` | Autoget trade goods | Yes |
| `OptionAutogetFood` | Autoget food | Yes |
| `OptionAutogetFreshWater` | Autoget fresh water | Yes |
| `OptionAutogetDroppedLiquid` | Autoget liquids from containers you've dropped | Yes |
| `OptionAutogetSpecialItems` | Autoget special items | Yes |
| `OptionAutogetArtifacts` | Autoget artifacts | Yes |
| `OptionAutogetScrap` | Autoget scrap | Yes |
| `OptionAutogetBooks` | Autoget books | No |
| `OptionAutogetZeroWeight` | Autoget weightless items | Yes |
| `OptionAutogetIfHostiles` | Autoget if hostiles are nearby | No |
| `OptionAutogetFromNearby` | Autoget from adjacent cells | No |

There is exactly one native "exclude a specific object" mechanism: the `NoAutoget` blueprint tag (`<tag Name="NoAutoget" Value="1" />`), found on 6 blueprints total across `Foods.xml`, `Furniture.xml`, and `Items.xml` in the base game data. It is **author-time only** — set by Freehold in the object's XML definition, never exposed through any options menu, wish, or in-game UI for players to set on arbitrary items. Grep of the full decompiled assembly for any player-facing "ignore this item" or per-object autoget blacklist API returned nothing beyond this static tag and `HasIntProperty("DroppedByPlayer")` (which is transient, cleared on next pickup, not a persistent player choice).

### 2. Feature-by-feature comparison

| Mod capability | Native equivalent (1.0.5) | Verdict |
|---|---|---|
| Per-item-blueprint exclusion, player-set at will, persists globally (`QudUX_AutogetSettings.json`, `ShouldAutoget:<Blueprint>`) | None. `NoAutoget` tag exists but is static/author-only, not player-settable. | **Still unique** |
| Category options (nuggets, trade goods, food, ammo, scrap, books, artifacts, freshwater, dropped-liquid, special items, zero-weight, if-hostile, from-nearby) | Fully native — 14 `Category="Autoget"` options in `Options.xml`, actively maintained through 2026 (see nugget/trade-goods precedence fix). QudUX does **not** reimplement any of these; `QudUX_AutogetManagementScreen` explicitly states "Your default game settings for auto-pickup will be applied first, and then any additional exclusions you have set here will also be applied." | **Fully redundant with native (mod correctly defers to it, doesn't duplicate it)** |
| Interaction-menu action "Disable/Re-enable auto-pickup for this item" | None — no per-item toggle exists anywhere in the base game's inventory action list. | **Still unique**, but its confirmation dialog is implemented with a legacy blocking-popup pattern that is broken under Modern UI (§4). |
| Management screen (`QudUX_AutogetManagementScreen`, wish `autopickup menu`) | None — nearest native analog is the flat Options > Autoget checkbox list, which has no concept of "list of individually-excluded items." | **Still unique**, but implemented as a legacy full-screen `IScreen` (`ConsoleLib.Console` buffer draw), not a Modern UI screen, and its own two Yes/No confirmations (`Remove selected` / `Remove all`) use `Popup.ShowYesNo` calls that **omit `defaultResult`**, so they default to `DialogResult.Yes` — see §4, this is a worse variant of the same bug. |
| Persistence | Native options persist through the same global (not per-save) options store; QudUX persists through its own JSON `NameValueBag` at the mod's install directory. Conceptually parallel (both global, not per-character), just a separate file. | **Parity — not obsolete, just a separate store** |

### 3. Obsolescence verdict

**STILL VALUABLE.**

- **Must stay:** the per-item exclusion data model (`QudUX_AutogetHelper.IsAutogetDisabledByQudUX`, `Patch_XRL_World_GameObject_CanAutoget`), the interaction-menu action, and the management screen/wish. None of this exists natively in any 2023–2026 patch, and the March 2026 nugget/trade-goods fix only refines a *category*-level toggle, not per-item control.
- **Could be dropped:** nothing needs dropping for redundancy — the mod does not duplicate any native category logic. The only things worth removing are the **broken legacy popup calls themselves** (not the feature they gate), replaced with a Modern-UI-safe confirmation (§5/§6).

### 4. Why the confirmation popup doesn't show

Two candidate root causes, both grounded in code, presented in order of likelihood for what the user's specific UAT run hit:

**HYPOTHESIS A — "shown once" flag already set (most likely actual cause of the recorded UAT run): CONFIRMED as a valid code path, unverified as the specific trigger without inspecting the user's actual `QudUX_AutogetSettings.json` at test time.**
`QudUX_AutogetHelper.HandleEvent(InventoryActionEvent)` only calls `Popup.ShowYesNo` at all when `Metadata:InfoboxWasShown` is not `"Yes"`:
```csharp
bool bInfoboxShown = AutogetSettings.GetValue("Metadata:InfoboxWasShown", "").EqualsNoCase("Yes");
if (!bInfoboxShown) { /* ...while loop with Popup.ShowYesNo... */ }
else { AutogetSettings.SetValue($"ShouldAutoget:{E.Item.Blueprint}", "No"); }  // <- silent, no popup, matches observed behavior exactly
```
This flag is global (stored in the mod's JSON, not per-save) and is set the first time any player answers Yes on any character. A prior test pass, or any earlier play session, silently disables the popup for all future items on all characters — which is exactly what was observed: exclusion works, no modal, no error. This is precisely why the UAT script's own Setup step 1 calls for deleting the JSON / clearing the flag before testing.

**HYPOTHESIS B — `Popup.ShowYesNo` is non-blocking under Modern UI, so even with the flag unset, the confirmation would malfunction: CONFIRMED as a genuine defect by decompiled code, not directly proven to be what the user's log shows (would manifest as a hang, which was not reported).**
Decompiled `XRL.UI.Popup.ShowYesNo` (Assembly-CSharp.dll):
```csharp
public static DialogResult ShowYesNo(string Message, string Sound = "...", bool AllowEscape = true,
    DialogResult defaultResult = DialogResult.Yes, Action<DialogResult> callback = null)
{
    SoundManager.PlayUISound(Sound, 1f, false, true);
    ...
    DialogResult result = defaultResult;
    if (UIManager.UseNewPopups)
    {
        WaitNewPopupMessage(Message, PopupMessage.YesNoButton, delegate(QudMenuItem i) {
            if (i.command == "No") result = DialogResult.No;
            if (i.command == "Yes") result = DialogResult.Yes;
            ...
        });
        return result;   // <- returns IMMEDIATELY with defaultResult; the callback fires later, asynchronously
    }
    // ...legacy synchronous Keyboard.getvk() blocking loop, only reached when UseNewPopups is false
}
```
and `Qud.UI.UIManager`:
```csharp
public static bool UseNewPopups => Options.ModernUI;
```
`OptionModernUI` defaults to `Yes` in `Options.xml`. `WaitNewPopupMessage` is `async void`; on the UI thread it `await`s `NewPopupMessageAsync`, which itself doesn't actually show the popup window until `await The.UiContext` resumes on a later frame via `GameManager.Instance.uiSynchronizationContext` — a normal `SynchronizationContext` that requires a return to Unity's frame loop to pump. Because `ShowYesNo` does not await any of this, it returns `defaultResult` to its caller on the same call stack, before the popup has rendered.

`QudUX_AutogetHelper.HandleEvent` wraps this in:
```csharp
DialogResult choice = DialogResult.Cancel;
while (choice != DialogResult.Yes && choice != DialogResult.No)
{
    choice = Popup.ShowYesNo(msg, AllowEscape: false, defaultResult: DialogResult.Cancel);
}
```
Under Modern UI, every call returns `Cancel` (the passed `defaultResult`) instantly — `Cancel` never satisfies the loop's exit condition, so this is a synchronous busy-loop that never returns control to Unity's message pump, meaning the pending popup's own `await The.UiContext` continuation can never run either. In principle this should hang the game, not merely skip the popup silently — which is *not* what the UAT observed, so Hypothesis A (flag already set, loop never entered) is the better fit for the specific run recorded in `qudux_UAT.md`. Hypothesis B remains a real, separate defect that would surface as a **freeze**, not a silent skip, for any player who has genuinely never answered the popup before (e.g. a brand-new install with Modern UI on, which is the 1.0.5 default). ⚠️ unverified whether the user's specific session actually froze briefly or not — `qudux_UAT.md` doesn't mention a hang either way.

**A distinct, worse instance of Hypothesis B, found while reading `QudUX_AutogetManagementScreen.cs`:** its two confirmations —
```csharp
if (Popup.ShowYesNo($"Remove auto-pickup exclusion for {optionStrings[selectedIndex]}?") == DialogResult.Yes)
if (Popup.ShowYesNo("Remove ALL of your auto-pickup exclusions?") == DialogResult.Yes)
```
— call the overload with **no `defaultResult` argument**, so `defaultResult` defaults to `DialogResult.Yes`. Under Modern UI this means these calls return `Yes` **immediately**, before the player has seen or answered anything: pressing Space/Enter or `R` in the management screen will silently delete the selected exclusion (or all of them) right away, with a popup that appears moments later and has no effect on what already happened. This is CONFIRMED by the same decompiled `ShowYesNo` default-parameter behavior above; it was not part of the original UAT-10 script but is directly relevant to "why the modal doesn't behave as expected" in the same feature area and should be fixed alongside the main confirmation.

The sibling fix already on this branch (`git diff` of `Parts and Effects/QudUX_AutogetHelper.cs`) only changed the call from positional args `(false, DialogResult.Cancel)` to named args `(AllowEscape: false, defaultResult: DialogResult.Cancel)` — a compile-fix for 1.0.5's parameter changes, not a behavior change, and it does **not** address either hypothesis above. It correctly keeps `Cancel` (a safe, non-destructive default) rather than `Yes`, which at least avoids the management-screen's data-loss failure mode for this specific call site.

### 5. Modern-UI interaction and what a feature-flag implementation should do

`Popup.ShowYesNo` is not "old UI vs new UI rendering conflict" in the sense of two things drawing on top of each other and clashing visually (as with the sprite-menu/callout issue noted in UAT 6) — it is a **synchronous-call-site vs asynchronous-implementation mismatch**. The method still exists and still works, but under `Options.ModernUI == true` it is fundamentally fire-and-forget: it schedules a popup and returns a default value immediately, expecting callers to either use the `callback` parameter or call `Popup.ShowYesNoAsync` / `NewPopupMessageAsync` and `await` it. Any mod code written for the pre-207.31 (pre–May 2024) fully-synchronous console UI that calls `ShowYesNo` in a blocking `while` loop and inspects the return value will misbehave the moment the player has Modern UI enabled (the 1.0.5 default).

A feature-flag / compatibility implementation should:
- Detect `Options.ModernUI` (or just always use the async-safe path, since it also works when Modern UI is off) and call `Popup.ShowYesNoAsync(msg)` with `await`, inside an `async` handler, rather than polling `ShowYesNo` in a `while` loop.
- Never rely on `ShowYesNo`'s return value alone under Modern UI without either awaiting `ShowYesNoAsync` or using the `callback` parameter and deferring the state-changing side effect (`AutogetSettings.SetValue(...)`) into that callback.
- Always pass an explicit `defaultResult` matching the *safe* (non-destructive) choice for that dialog, since the default-parameter return is what's actually delivered under Modern UI's fire-and-forget path.

### 6. Recommendation

- **Keep**: per-item exclusion data model, `Patch_XRL_World_GameObject_CanAutoget` Harmony prefix, `QudUX_AutogetHelper.IsAutogetDisabledByQudUX`, the interaction-menu action pair, the management screen concept, and the `autopickup menu` wish. None of this is covered by any native API as of build 212.17 (2026) — there is no native per-item/per-blueprint exclusion or management UI, only category checkboxes.
- **Gate/rewrite**: replace the blocking `while (choice != Yes && choice != No) { choice = Popup.ShowYesNo(...) }` pattern in `Parts and Effects/QudUX_AutogetHelper.cs` (~line 84–96) with an `async`-aware confirmation using `Popup.ShowYesNoAsync` (or `ShowYesNo`'s `callback` parameter) so the state change (`AutogetSettings.SetValue`) happens only once the player actually answers. Apply the same fix to both `Popup.ShowYesNo(...)` calls in `Screens/QudUX_AutogetManagementScreen.cs` (remove-one and remove-all), additionally passing `defaultResult: DialogResult.No` (or restructure to not read the return value synchronously at all) so a Modern-UI player can't have exclusions silently deleted before the popup even renders.
- **Simplify onto a native API**: none available — `Options.AutogetNuggets` / `Options.AutogetTradeGoods` and friends are category-wide only; there is no native per-item toggle to delegate to. No simplification opportunity exists here beyond what's already correctly deferred (category checks already flow through to native `ShouldAutoget()`/`Options.Autoget*`).
- **Remove**: nothing. The feature is not obsolete; only its confirmation-dialog plumbing needs a Modern-UI-safe rewrite.
- Files/options involved: `Parts and Effects/QudUX_AutogetHelper.cs`, `Screens/QudUX_AutogetManagementScreen.cs`, `Wishes/AutopickupMenu.cs`, `Harmony Patches/Patch_XRL_World_GameObject_CanAutoget.cs`, QudUX option `QudUX_OptionAutogetExclusions` (Options.xml, Category "QudUX"), native options `OptionAutogetNuggets` / `OptionAutogetTradeGoods` / etc. (Category "Autoget"), and the JSON store `QudUX_AutogetSettings.json` (`Metadata:InfoboxWasShown`, `ShouldAutoget:<Blueprint>` keys).

### 7. Confidence and gaps

- **High confidence, code-grounded:** native autoget has no per-item exclusion (verified by decompiling `GameObject.CanAutoget`/`ShouldAutoget` and grepping all base-game XML for `NoAutoget`); `Popup.ShowYesNo` is fire-and-forget under Modern UI (verified by decompiling `Popup.cs` and `Qud.UI/UIManager.cs`); the nugget-vs-trade-goods precedence in shipped 1.0.5 code matches the March 2026 patch note verbatim; `OptionModernUI` defaults to Yes (verified in `Options.xml`); the management screen's two `ShowYesNo` calls omit `defaultResult` and thus default to `Yes` under Modern UI (verified by reading `QudUX_AutogetManagementScreen.cs` against the decompiled `ShowYesNo` signature).
- **Gap — which hypothesis actually fired in the recorded UAT run:** I could not inspect the tester's actual `QudUX_AutogetSettings.json` from the session that produced the "no modal" observation in `qudux_UAT.md`, so I cannot confirm whether Hypothesis A (flag already set) or a partial/non-hanging variant of Hypothesis B actually occurred. Marked ⚠️ unverified above. Recommend checking `Player.log` timestamps or the JSON file from that specific run if it's still on disk.
- **Gap — 2025 patch notes are thin.** `2025.wiki` is only 19KB (vs. 177KB/92KB for 2023/2024) and contains no autoget entries at all; this is consistent with 1.0's Dec 5, 2024 release shifting the changelog cadence, not with a gap in the local archive, but I did not cross-check against the live wiki to confirm no 2025 autoget entries were missed by the mirror. No web fetch was used for this — the task's 2-fetch allowance wasn't needed since the local `.wiki` files were sufficient for every claim above.
- **Not independently tested in-engine:** all of the above is static analysis (decompiled IL/C# and XML) plus the existing UAT transcript; I did not launch the game to reproduce the hang/no-hang question in Hypothesis A vs B.
