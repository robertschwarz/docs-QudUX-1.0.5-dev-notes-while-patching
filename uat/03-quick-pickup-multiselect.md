# UAT 03: Quick Pickup multi-select popup

**What was fixed:** the "Quick Pickup" multi-item popup could not be confirmed — picking any items and hitting Accept always said "You cannot select more than 0 options!", so the feature was completely unusable. It's fixed now: pick one or several items, confirm, and you actually get them.

## 1. Feature under test

`QudUX_QuickPickupPart` (`Parts and Effects/QudUX_QuickPickupPart.cs`). Command **"Display Quick PickUp QuickMenu"**, bound to **Ctrl+Numpad0**. When pressed, it scans every *explored* cell in the current zone for Armor, Melee Weapons, Missile Weapons, Shields and Tools, then opens a popup titled **"Which item do you want to get ?"** listing every matching item found (not just items on your own tile). Confirming a selection queues an auto-walk-and-pickup action (`AutoAct`) that paths to each chosen item and picks it up, even if it isn't adjacent.

A companion command, **"Display Quick PickUp Settings"** (Ctrl+Numpad5), opens a filter screen (Weapons / Equipment / Tiers tabs) that lets you disable specific item types, armor slots, or tiers so they're skipped by the scan.

## 2. Preconditions

- QudUX options: none — there is no global on/off toggle for this feature in `Options.xml`/`Concepts/Options.cs`; it's always active once the mod loads.
- Quick Pickup filters (`Ctrl+Numpad5`, per-type toggles): all default to **Enabled** on a fresh character (`Tier 0`–`Tier 8`, all weapon/tool types, all armor slots). Don't change anything before Step 1.
- Game options: none required.
- Save state: fresh `qudux-32-test-uat-character`, baseline save from `00-setup.md` step 4.

## 3. Setup — drop 4 distinct items on one tile

Use the wish prompt (`Ctrl+W`, see `00-setup.md`). The `item:<Blueprint>:<count>:here` wish places `<count>` copies of `<Blueprint>` directly on **your current cell** (confirmed by decompiling `Wishing.HandleWish`'s `item:` branch — without `:here` it instead places in an adjacent empty cell). Enter each of these as its own wish:

```
item:Dagger:1:here
item:Club:1:here
item:Short Bow:1:here
item:Leather Armor:1:here
```

These 4 blueprints were picked deliberately: `Dagger` (→`BaseDagger`→`MeleeWeapon`), `Club` (→`BaseCudgel`→`MeleeWeapon`), `Short Bow` (→`BaseBow`→`BaseMissileWeapon`→`MissileWeapon`), `Leather Armor` (→`BaseArmor`→`Armor`, worn on `Body`) — all confirmed in `ObjectBlueprints/Items.xml`. They're chosen from the three categories (`MeleeWeapon`/`MissileWeapon`/`Armor`) that `BuildPopup()`'s internal re-ordering step (`SelectAndOrderObjects`) actually keeps; a Shield or Tool dropped the same way is a valid candidate during the scan but gets silently dropped by that re-ordering step before the popup is built. That's a separate, pre-existing behavior — **not** part of this fix — so avoid Shields/Tools in this test to prevent a false FAIL.

## 4. Test steps

1. **Confirm default filters.** Press `Ctrl+Numpad5`. On the Weapons tab, confirm every entry reads `Enabled` (green). Press `Esc` (or `Numpad5`) to close.
   Expected: settings screen opens, all types show `Enabled`, closes cleanly.
   Evidence: 📸 `UAT-03-01-settings-default.png` — Weapons tab fully visible with all `Enabled` labels.

2. **Drop the 4 items.** Run the 4 wishes from section 3, on the same tile you're standing on.
   Expected: no errors from the wishes; your tile now holds 4 items.
   Evidence: 📸 `UAT-03-02-items-dropped.png` — game view showing your character's tile (item stack indicator or `look`/examine output listing the 4 items).

3. **Open the popup and inspect layout.** Press `Ctrl+Numpad0`.
   Expected: popup opens with the exact title **"Which item do you want to get ?"**; all 4 items listed, each with an unchecked `[ ]` box and a distinct item icon to its side; popup width is a normal, readable column (not collapsed/near-zero width, not text overlapping the icons); footer shows two extra lines: `[Delete/Backspace] Accept` and `[Tab] Select All`.
   Evidence: 📸 `UAT-03-03-popup-open.png` — full popup visible: title, all 4 rows with icons, both footer buttons.

4. **Select one item and confirm.** Highlight `Dagger` (arrow keys) and press Enter (or click it) to check its box — it should change to `[_]`. Then press **Delete or Backspace** (Accept).
   Expected: popup closes immediately, no "cannot select more than" error; because the item is on your own tile, pickup happens without a walk (may take 1 turn). Open inventory (`i`) to confirm the dagger is there.
   Evidence: 📸 `UAT-03-04-dagger-checked.png` — Dagger's box checked, others unchecked, just before pressing Accept.
   Evidence: 📸 `UAT-03-04-dagger-in-inventory.png` — inventory screen showing the dagger now carried.

5. **Select several items and confirm (the regression this fix targets).** Press `Ctrl+Numpad0` again — the popup now lists the 3 remaining items (Club, Short Bow, Leather Armor). Press **Tab** (Select All) to check all 3, then press **Delete or Backspace** (Accept).
   Expected: all 3 boxes check when you press Tab; pressing Accept closes the popup with **no** "You cannot select more than 0 options!" popup (that message was the exact old bug) and no exception; all 3 items end up in your inventory.
   Evidence: 🎥 `UAT-03-05-selectall-confirm.mp4` — record from pressing Tab (all boxes check) through pressing Accept (popup closes cleanly) to opening inventory (all 3 new items visible).

6. **Escape cancels, nothing picked up.** Wish one more item on your tile: `item:Dagger:1:here`. Press `Ctrl+Numpad0`, then press `Esc` without checking anything.
   Expected: popup closes, no items picked up, the dagger is still on the ground.
   Evidence: 📸 `UAT-03-06-escape-cancel.png` — tile still showing the dagger, popup gone, inventory unchanged.

7. **Disabled filter hides that item type.** Press `Ctrl+Numpad5`, go to the Weapons tab, highlight `Daggers`, press `Space` or `Enter` to set it to `Disabled`, then `Esc` to close. Press `Ctrl+Numpad0` (the dagger from step 6 is still on the ground).
   Expected: the popup either shows no items ("There is nothing of interest to pick up here...") or lists other nearby matching items but **not** the dagger.
   Evidence: 📸 `UAT-03-07-filter-disabled-dagger-absent.png` — popup (or "nothing of interest" message) with the dagger absent.
   Cleanup: reopen `Ctrl+Numpad5`, re-enable `Daggers` before continuing (restores default for later scripts).

8. **Turn/AutoAct behavior for an off-tile item.** Wish an item without `:here` so it lands adjacent instead of underfoot: `item:Dagger:1`. Walk 4–5 tiles away from it (arrow keys). Press `Ctrl+Numpad0`, check the dagger, press Accept.
   Expected: turn advances; your character auto-walks back toward the dagger's cell over multiple turns (movement is visible/steppy, not instant); once adjacent/on its cell it's picked up and you see the message "You picked all the items you were interested in."; check inventory afterward.
   Evidence: 🎥 `UAT-03-08-autoact-walk.mp4` — from pressing Accept through the walk to the pickup message.

## 5. Log check

After finishing, search `Player.log` (per `00-setup.md` §2) for:

```
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "QudUX_QuickPickupPart|PickSeveral|BuildPopup|Exception"
```

FAIL if any hit's stack trace mentions `QudUX_QuickPickupPart`, `PickSeveral`, or `BuildPopup`, or if any unrelated `Exception` appears at the moments you pressed `Ctrl+Numpad0` or Accept.

## 6. Fail indicators

- Popup title is missing, blank, or not exactly "Which item do you want to get ?".
- Any option row has no icon, or icons overlap/are cut off; popup column is collapsed to near-zero width or text overlaps the icon column.
- Checking more than 0 items and pressing Accept shows **"You cannot select more than 0 options!"** and the popup won't close — this is the exact original bug; treat as FAIL.
- Pressing Accept with items checked closes the popup but nothing appears in inventory.
- Escape picks up items anyway.
- Game freezes, crashes, or shows an unhandled-exception popup at any step.
- Player.log shows an exception referencing `QudUX_QuickPickupPart`, `PickSeveral`, or `BuildPopup`.

## 7. Results

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

### Problems found

For each problem:

```
Steps to reproduce:
Expected:
Actual:
Evidence:
Player.log excerpt:
```

## Patch-note correlation (2023–2026)

Correlates UAT 3.1/3.2/3.3 (`qudux_UAT.md`) against `wiki.cavesofqud.com/Version_history` 2023–2026 (local `.wiki` dumps) and decompiled `Assembly-CSharp.dll`. Code referenced: `Parts and Effects/QudUX_QuickPickupPart.cs` (`BuildPopup`), `AutoAct/PickupSelection.cs` (`Continue`, `GetPath`), `Parts and Effects/QudUX_AutogetHelper.cs`, `Harmony Patches/Patch_XRL_World_GameObject_CanAutoget.cs`, `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs`.

### 1. Relevant patch entries

**UI overhaul (context for the user's 3.2 hypothesis):**

- Build **207.31 "Spring Molting beta"**, released May 10, 2024: *"We added an entire new UI. (ie, we completed work on the modern UI, for folks who've been following our progress)... here's a partial list of new screens: trade, quests, message log, character sheet, skills, equipment & inventory, tinkering, game summary, every journal tab, status effects, reputation, world generation, interact nearby, books, in-game terminals."* — **RELATED**, not a direct cause. This is the entry that created the legacy-vs-modern UI split the user's overall QudUX finding is built on. It rewrote the `Equipment`/inventory screen and `interact nearby`, but `QudUX_QuickPickupPart.BuildPopup()` never touches either of those screens — it calls `XRL.UI.Popup.PickSeveral` directly, which is a screen-agnostic legacy popup type still present unchanged in the current `Assembly-CSharp.dll` (verified by decompile, signature matches the mod's call exactly: `Title, Options, Icons, AllowEscape` all still valid named params). This is why UAT 3.1 (the popup itself) still works.
- Build **1.0 (209.29)**, Dec 5, 2024, `=== UI ===`: *"Renamed the Skills & Powers screen to Skills, the Attributes screen to Attributes & Powers, and the Inventory & Equipment screen to Equipment."* — **RELATED / worth knowing**. The old "InventoryScreen" the mod's `Patch_XRL_UI_InventoryScreen.cs` targets is a legacy-UI-only concept now; the class name `XRL.UI.InventoryScreen` still exists (the Harmony patch targets it by type) but modern UI no longer routes through it, which is exactly why that patch already gates on `!(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)`. This is precedent for the recommended fix pattern below, but it patches a different screen than Quick Pickup uses.

**Pathfinding/movement engine changes (most likely cause of 3.2):**

- Build **211.33 (beta)**, released January 29, 2026: *"Greatly improved pathfinding performance."* / *"Greatly improved performance when auto-moving to the edge of a zone."* / *"Improved pathfinding logic of creatures, causing them to take more sensible paths around obstacles."* / *"Fixed a bug that caused you to bump into obstacles on the destination when moving to zone edges."* — **CAUSE (hypothesis)**. This is the closest-to-1.0.5 change touching the exact subsystem `PickupSelection.Continue()`/`GetPath()` depends on (`XRL.World.AI.Pathfinding.FindPath`, `AutoAct.TryToMove`). The public constructor signatures the mod calls are unchanged (verified by decompile — `FindPath(Cell,Cell,bool,bool,GameObject,int,bool,bool,bool,bool,bool,CleanQueue<XRLCore.SortPoint>)` and `AutoAct.TryToMove(GameObject,Cell,Cell,string,...)` both still match the mod's call sites argument-for-argument), so this isn't a compile break — it's a change to internal pathing behavior/heuristics that could plausibly make `FindPath.Usable` come back false, or produce a path the mod's step-by-step walker mishandles, for cases that worked before. No decompiled diff of the internal algorithm was done (out of scope/time), so this stays a hypothesis, not confirmed.

**Native autoget filtering changes (candidate for 3.3, none conclusive):**

- Build **211.36 (beta)**, released March 1, 2026: *"Autoget now ignores nuggets if the 'Autoget nuggets' option is disabled, even if 'Autoget trade goods' is enabled."* — **RELATED**. Shows the native `GameObject.ShouldAutoget()` category logic (decompiled: `GetInventoryCategory()` switch over Trade Goods/Food/Books/Nugget) is still being actively changed close to the 1.0.5 window. QudUX's autoget layer (`QudUX_AutogetHelper` + `Patch_XRL_World_GameObject_CanAutoget`) sits on top of this and doesn't touch the category logic itself, so this alone doesn't explain a break, but it confirms "how items are interpreted" (the user's own words) for autoget purposes has changed recently.
- Build **207.97**, released August 23, 2024: *"Objects are now re-flagged for autoget after being thrown."* — **RELATED / worth knowing**, touches the same `ShouldAutoget`/`CanAutoget` pipeline the mod's Harmony prefix wraps, but not obviously the cause.
- Build **206.69**, released March 15, 2024: *"Fixed a bug that caused the 'Auto-pickup of zero weight items' but skip picking up items with fractional weight."* (quoted verbatim, including the wiki's own garbled wording) — **RELATED / worth knowing**. Confirms `Options.AutogetZeroWeight` weight-threshold logic (present in the decompiled `ShouldAutoget()` body) has a history of edge-case bugs.
- Build **207.89**, released August 9, 2024: *"Disarmed mines originally created by the player are now autogettable."* — **unrelated but worth knowing**, same subsystem.
- Build **204.56**, released February 3, 2023: *"Items unintentionally dropped by your companions are now auto-picked-up when the 'Auto-pickup special items' option is enabled."* — **RELATED / worth knowing**.
- Build **204.53**, released January 27, 2023: *"Any time the player is forced to drop an item, that item is now considered a 'special item' until the next time the player or a companion of the player picks it up, meaning autoget will pick it up if 'Auto-pickup special items' is on."* — **RELATED / worth knowing**. This is exactly the `DroppedByPlayer`/`IsSpecialItem` interaction that `QudUX_AutogetHelper.HandleEvent(OwnerGetInventoryActionsEvent E)` explicitly works around (it strips `DroppedByPlayer` before calling `E.Object.ShouldAutoget()` so the disable-list check isn't skewed by a recent player-drop). Confirms the mod's workaround targets a real, still-current native mechanic; not evidence of breakage by itself.
- Build **206.26**, released November 15, 2023: *"Added an autoget option 'Exclude liquid in containers you've dropped from auto-collect'."* — **unrelated**, liquids not weapons/armor/tools.

**Native multi-select / "take all" search — nothing found.** Grepped all four `.wiki` files (2023–2026 inclusive) for `get all`, `take everything`, `loot all`, `grab all`, `multiselect`/`multi-select`, `select multiple`, `bulk pick` — **zero matches**. The base game has not added a native equivalent of "scan the zone for gear matching filters, show a checklist, walk-and-collect the selection" at any point 2023–2026.

### 2. Obsolescence verdict: **STILL VALUABLE**

- No patch note in the 2023–2026 window adds a native batch/multi-select pickup feature (see search above). The closest native things are: (a) walking over an item to trigger `ShouldAutoget()`, an all-or-nothing automatic behavior with no manual multi-select, and (b) the per-item `CanAutoget`/interaction-menu "disable auto-pickup" toggle, which the mod itself already layers onto (and which UAT confirms still works).
- The mod's actual value-add — cross-zone scan by item category/tier/armor-slot with a checklist popup and auto-walk-and-fetch — has no native counterpart, so it isn't redundant. It is currently **broken** for the ground/chest case (3.2) and the interaction with native autoget appears degraded (3.3), but "broken" and "obsolete" are different verdicts here: the feature still does something the base game can't.

### 3. Modern-UI interaction

- `Popup.PickSeveral` (the popup UAT 3.1 confirms works) is a legacy `XRL.UI` popup type, not part of the modern-UI screen rewrite from build 207.31/1.0 — its signature is unchanged and it renders regardless of `GameOptions.ModernUI`. **The popup layer of this feature is not modern-UI-gated and, per the evidence gathered, has no reason to behave differently between UI modes.**
- The follow-up mechanics — `AutoAct`, `FindPath`, `GameObject.TakeObject`, `GameObject.CanAutoget` — are all engine/world-simulation code, not UI-screen code, and are not part of the modern-UI/legacy-UI split at all; they run identically regardless of which UI the player has selected.
- **This weakens the user's own hypothesis for 3.2** ("most likely not compatible with UI overhaul"). The code path Quick Pickup exercises after the popup closes doesn't intersect any of the classes the modern-UI rewrite replaced. The UAT 3.3 note ("even with old UI on") independently supports this: if the bug were modern-UI compatibility, legacy UI should have been immune, and it wasn't.
- Practical consequence for the planned feature-flag/UI-mode approach: **gating this feature behind UI mode would not fix 3.2 or 3.3**, because the failure isn't in UI-mode-specific code. It would be a wasted gate for this particular feature — unlike e.g. `Patch_XRL_UI_InventoryScreen.cs`, which legitimately needs the gate because it patches a screen class modern UI bypasses.

### 4. Root-cause hypotheses for 3.2 and 3.3

**3.2 — ground pickup and chest pickup fail:**

- **HYPOTHESIS**: Engine pathfinding behavior changed in build 211.33 (Jan 29, 2026 — "improved pathfinding logic... more sensible paths around obstacles," "bump into obstacles on destination" fix). `PickupSelection.GetPath()` (`AutoAct/PickupSelection.cs`) builds a `FindPath` per target and picks the first `Usable` one; `Continue()` then calls `AutoAct.TryToMove` step by step and finally `_Player.TakeObject(target, false, false, null, "QuickPickup")` once adjacent/on-cell. All four call signatures still match the current decompiled API exactly (verified), so this is not a compile-time break — if it's broken, it's a behavioral/heuristic change inside pathfinding that the mod's manual walk-and-fetch loop doesn't handle the same way anymore (e.g. `FindPath.Usable` now returning false, or the returned `Steps`/`Directions` no longer lining up with what `TryToMove` expects near obstacles). Not confirmed — no decompiled diff of `FindPath`'s internal body across versions was performed.
- **HYPOTHESIS, secondary**: "Also doesn't work in a chest" may not be a regression at all. `BuildPopup()` populates candidates from `Cell.GetObjectsThatInheritFrom`, which iterates only `Cell.Objects` — the flat list of things lying directly on that cell (verified by decompile). It does not, and structurally cannot, reach into a container object's own `Inventory` part. So Quick Pickup was likely never able to see or fetch items *inside* a chest; it only ever found loose floor items. If the user expected chest contents to be scannable, that's a scope gap rather than something the game changed. This needs a screenshot/description check against `image-21.png` to confirm what was actually attempted — flagged as a gap below.

**3.3 — autoloot wrong even with legacy UI:**

- **HYPOTHESIS, weak**: native `ShouldAutoget()`/`CanAutoget()` category and threshold logic has had multiple small changes across the window (211.36 nugget/trade-goods interaction change, 206.69 zero/fractional-weight bug fix, 207.97 re-flag-after-throw, 204.53/204.56 dropped-item "special item" flagging) showing the "item interpretation" the user suspects does keep moving, but no single quoted entry describes the specific symptom (UAT screenshots 22/23 weren't legible to this pass — see gaps). `QudUX_AutogetHelper`'s own logic (`IsAutogetDisabledByQudUX`, the Harmony prefix on `CanAutoget`) is a thin, unconditional layer that defers to native `ShouldAutoget()`/`CanAutoget()` whenever QudUX hasn't explicitly disabled an item, so a native-side regression there would surface exactly as "autoget behaves oddly, but the disable-list toggle itself still works" — which matches UAT 3.3's own note that the "disable auto-pickup" toggle passes. This is consistent with, but not proof of, a native-side cause.
- **Not confirmed**: nothing in the patch notes explicitly says "autoget miscounts/misfires" or similar in a way that maps cleanly onto whatever 3.3's screenshots show. This root cause needs the actual 3.3 repro steps (what was expected vs what happened) before a code reviewer can target a fix.

### 5. Recommendation

**Keep the feature, but split the fix from the UI-mode question:**

- **Do not gate Quick Pickup behind the UI-mode flag.** Section 3 shows this code path (`QudUX_QuickPickupPart`, `PickupSelection`, `Popup.PickSeveral`, `FindPath`, `AutoAct`, `GameObject.TakeObject`/`CanAutoget`) is UI-mode-agnostic engine code, not modern-vs-legacy screen code. A UI-mode gate would leave 3.2/3.3 broken for legacy-UI users too, which conflicts with the user's stated goal of keeping legacy-UI users on working features.
- **Investigate 3.2 as a pathfinding/engine-version issue**, not a UI issue: instrument or log `FindPath.Usable` and `.Steps`/`.Directions` output in `PickupSelection.GetPath()`/`Continue()` against a known-good ground item and a known-good adjacent-to-chest item, on current 1.0.5, to see whether `Usable` is false or the walk loop diverges. If `FindPath` truly changed shape, the fix is local to `AutoAct/PickupSelection.cs` — no native API to lean on instead, since there's no native "walk to and fetch a set of items" call (`XRL.World.Capabilities.AutoAct` doesn't expose anything comparable; confirmed no such native helper method in the decompiled `AutoAct` class beyond generic `TryToMove`).
- **Chest/container scanning**: if the user actually wants Quick Pickup to reach into containers (not just cell floors), that's a **new feature**, not a regression fix — `Cell.GetObjectsThatInheritFrom` structurally can't do it today. Recommend confirming intent with the 3.2 screenshots before scoping any container-reach work.
- **3.3**: no native API found that would let QudUX "simplify by leaning on it" — the native autoget pipeline (`ShouldAutoget`/`CanAutoget`) is exactly what QudUX already hooks into via `Patch_XRL_World_GameObject_CanAutoget`; there's nothing higher-level to delegate to instead. Recommend reproducing 3.3 with a controlled item drop + Player.log capture (per this file's own §5 Log check) before attempting a code fix, since no patch note pinpoints the exact symptom.
- Files/options involved: `Parts and Effects/QudUX_QuickPickupPart.cs`, `AutoAct/PickupSelection.cs`, `Parts and Effects/QudUX_AutogetHelper.cs`, `Harmony Patches/Patch_XRL_World_GameObject_CanAutoget.cs`. No `Options.xml`/UI-mode option needs to change for this feature specifically.

### 6. Confidence + gaps

**Verified directly:**
- Full text of `QudUX_QuickPickupPart.cs`, `PickupSelection.cs`, `QudUX_AutogetHelper.cs`, both relevant Harmony patches.
- Decompiled current 1.0.5 `Assembly-CSharp.dll` signatures for `Popup.PickSeveral`, `GameObject.TakeObject` (all overloads), `GameObject.CanAutoget`/`ShouldAutoget` (full body), `XRL.OngoingAction`, `XRL.World.Capabilities.AutoAct` (`TryToMove`, `Action` property), `XRLCore.PlayerAvoid`/`SortPoint`, `XRL.World.AI.Pathfinding.FindPath` (all constructor overloads), `Cell.GetObjectsThatInheritFrom`, `Zone.GetExploredCells` — every one matches what the mod's source calls, so nothing here is a plain compile/signature break.
- Grepped all of `2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026) for pickup/autoget/container/UI-overhaul/multiselect keywords; entries quoted above are the complete relevant set found, not a sample.

**Gaps / not verified:**
- Did not decompile the *internal implementation* of `FindPath`'s pathing algorithm to diff it against a pre-211.33 build — no older assembly was available locally to diff against, so the pathfinding-change hypothesis for 3.2 is circumstantial (timing + description match), not a confirmed code diff. Tag: ⚠️ unverified beyond the quoted patch note.
- Did not view the UAT screenshots (`image-17.png` through `image-23.png`, `image-41.png`/`image-42.png`) — this pass is text-only. The chest-vs-floor distinction for 3.2 and the exact symptom for 3.3 are inferred from the surrounding UAT prose, not the images. Recommend a follow-up pass that reads those images before a code fix is attempted.
- No explicit "1.0.5" version header exists in the wiki dumps — the newest entries are build **212.17 (experimental)**, March 27, 2026, and build **211.36 (beta)**, March 1, 2026. Treating these as the closest available proxy for "current 1.0.5"; if 1.0.5 shipped with additional unlisted changes between 212.17 and release, those aren't covered here. ⚠️ unverified: exact build number that corresponds to "1.0.5."
- Did not use any of the 2 permitted web fetches — everything needed was in the local wiki dumps and the decompiled assembly.
