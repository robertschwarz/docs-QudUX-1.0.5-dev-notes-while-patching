# Error 9 — `Options.OverlayPrereleaseInventory` removed

**File:** `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs:16`

**Error:**
```
error CS0117: 'Options' does not contain a definition for 'OverlayPrereleaseInventory'
```

## Status: CORRECTED ✅ (guard tightened by orchestrator — see bottom)

The baseline fix (dropping the condition entirely) compiled but was a **behavior regression**: it made QudUX unconditionally replace the inventory screen even for players who opted into the game's modern/overlay inventory UI. Restored the guard using the 1.0.5 successor flag.

```diff
  if (QudUXOptions.UI.UseQudUXInventory
+     && !GameOptions.ModernCharacterSheet
      && XRLCore.Core.Game.Player.Body.GetConfusion() <= 0)
```

## Agent review (1.0.5)

**Verdict: CORRECTED**

The baseline fix (`git apply baseline.patch`) simply deleted the `!GameOptions.OverlayPrereleaseInventory` guard, meaning `QudUXOptions.UI.UseQudUXInventory && Confusion <= 0` was the only condition left. That compiles, but it silently drops the original intent of the guard: "don't take over the inventory screen if the player is using the game's own modern/overlay inventory UI." With the guard removed, QudUX would always intercept `InventoryScreen.Show`, even for players who explicitly opted into the vanilla modern inventory — a real behavior regression, not just a rename cleanup.

### What I verified against the decompiled 1.0.5 `Assembly-CSharp.dll`

1. **`Options.OverlayPrereleaseInventory` is gone** — confirmed via `ilspycmd -t XRL.UI.Options`. No field/property by that name or anything containing "Prerelease...Inventory" remains.
2. **Found the successor flag: `Options.ModernCharacterSheet`.** Despite the name, it's not just for the character sheet. Decompiling `GameManager` (the class that owns the `ModernUI`/legacy-input dispatch logic) shows this block gating passthrough input for a whole family of screens:
   ```csharp
   if (ModernUI && !Options.ModernCharacterSheet)
   {
       bool flag = false;
       if (Instance._ActiveGameView == "Equipment") flag = true;
       if (Instance._ActiveGameView == "Inventory") flag = true;
       if (Instance._ActiveGameView == "Status") flag = true;
       if (Instance._ActiveGameView == "Factions") flag = true;
       if (Instance._ActiveGameView == "Quests") flag = true;
       if (Instance._ActiveGameView == "Journal") flag = true;
       if (Instance._ActiveGameView == "Tinkering") flag = true;
       if (Instance._ActiveGameView == "SkillsAndPowers") flag = true;
       if (flag) { pushKeyEvents(); return; }
   }
   ```
   i.e. even with the general `ModernUI` flag on, Inventory (and Equipment/Status/Factions/Quests/Journal/Tinkering/SkillsAndPowers) still run through the classic legacy-input/ASCII screen path **unless** `Options.ModernCharacterSheet` is also true. When `ModernCharacterSheet` is true, those screens route instead to the new Unity-UI `Framework`-driven screens (see next point). This is a direct semantic match for what `OverlayPrereleaseInventory` used to gate — it's the shipped, generalized version of that prerelease-only flag.
3. **Confirmed a real modern/overlay inventory screen exists and is structurally distinct**: `Qud.UI.InventoryAndEquipmentStatusScreen : BaseStatusScreen<InventoryAndEquipmentStatusScreen>`, tagged `[UIView("InventoryAndEquipmentStatusScreen", ..., NavCategory = "Menus", UICanvas = "StatusScreens", ...)]`. This is the game's modern overlay inventory/equipment UI, driven through the `XRL.UI.Framework` UIView/NavigationController system — a completely separate code path from `XRL.UI.InventoryScreen.Show()`.
4. **Verified the classic `XRL.UI.InventoryScreen.Show(GameObject GO) : ScreenReturn`** contains zero `ModernUI`/`Overlay` checks internally (grepped the full decompiled method body, ~430 lines) — it's purely the legacy ASCII screen, unconditionally. The choice of classic-vs-modern happens upstream in `GameManager`, per point 2.
5. **Harmony patch target validity**: `[HarmonyPatch(typeof(XRL.UI.InventoryScreen))]` / `[HarmonyPatch("Show")]` prefixing `static bool Prefix(GameObject GO, ref ScreenReturn __result)`. Decompiled signature: `public ScreenReturn Show(GameObject GO)` on `XRL.UI.InventoryScreen` — name, single `GameObject` parameter, and `ScreenReturn` return type all still match exactly. The patch target still exists and will bind correctly; it will not silently no-op or throw at Harmony patch time.

### Fix applied

```diff
  if (QudUXOptions.UI.UseQudUXInventory
+     && !GameOptions.ModernCharacterSheet
      && XRLCore.Core.Game.Player.Body.GetConfusion() <= 0)
```

(`GameOptions` is the existing `using GameOptions = XRL.UI.Options;` alias already in the file — no new using needed.)

### Build result

`dotnet build Mods.csproj` — zero errors touching this file. Only remaining error in the tree is the pre-existing, out-of-scope `Brain.Factions` issue (#11), owned by another agent.

### Runtime risks / open questions

- I did not find (and did not exhaustively trace) the exact call site that decides, per-keypress, whether `InventoryScreen.Show()` or `InventoryAndEquipmentStatusScreen` gets invoked for the "Inventory" command — that logic is presumably in a `NavigationController`/command-binding layer I did not fully chase down (whole-assembly decompile was too slow to complete in this session). The `GameManager` evidence above is strong indirect confirmation (it explicitly branches legacy-vs-modern input passthrough for the `"Inventory"` view keyed on exactly this flag), but I have not directly observed the line that calls `new InventoryScreen().Show(...)` vs `NavigationController...GotoView("InventoryAndEquipmentStatusScreen")`.
- Low risk either way: if `ModernCharacterSheet` turns out not to gate `InventoryScreen.Show()` invocation directly, the guard is still harmless — it only makes QudUX's own inventory screen opt out in the same situations the original pre-1.0.5 code did, which was already the intended, shipped behavior for QudUX users.

## Orchestrator follow-up (1.0.5) — guard tightened

Traced the open question above. Dispatch lives in `XRL.UI.Screens` (decompiled 1.0.5):

```csharp
public static void Show(GameObject GO)
{
    if (Options.ModernUI && Options.ModernCharacterSheet)
    {
        ... _ = StatusScreensScreen.show(startingScreen, GO).Result;   // modern screen
        return;
    }
    ...
    while ((screenReturn = ScreenList[CurrentScreen].Show(GO)) != ScreenReturn.Exit)  // classic, includes InventoryScreen
```

(`Screens.Show(GO, string Screen)` has the same `ModernCharacterSheet && ModernUI` branch, else `PopupScreens[Screen].Show(GO)`.)

So classic `InventoryScreen.Show` is used whenever `!(ModernUI && ModernCharacterSheet)`. The agent's `!ModernCharacterSheet` alone would wrongly disable QudUX's inventory for `ModernCharacterSheet=on, ModernUI=off` (classic screen still shown there) — contradicting the agent's "harmless" note. Guard changed to mirror the game's own condition exactly:

```diff
  if (QudUXOptions.UI.UseQudUXInventory
-     && !GameOptions.ModernCharacterSheet
+     && !(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)
      && XRLCore.Core.Game.Player.Body.GetConfusion() <= 0)
```

Build: 0 errors (full tree, all 12 issues merged). Status: **FIXED ✅** (not runtime-tested in-game).
