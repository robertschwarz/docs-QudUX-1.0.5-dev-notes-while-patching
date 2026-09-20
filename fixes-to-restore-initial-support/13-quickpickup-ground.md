# Step 13 — Find and fix "quick pickup does nothing" on the ground

**Lane D · Depends on: nothing · Owns: `AutoAct/PickupSelection.cs`**

Read `00-README.md` first.

## Goal

Make selecting items in the quick-pickup menu actually pick them up.

Confirm the cause before changing anything. The leading hypothesis below is a hypothesis, not a finding.

## Why

UAT 3.2: the menu opens and items can be selected, but nothing is picked up. The tester guessed the modern UI was to blame. **That is unlikely** — `Popup.PickSeveral`, `FindPath`, `AutoAct` and `TakeObject` are all legacy engine code that the modern-UI rewrite did not touch. Do not gate this feature on a UI mode.

## Leading hypothesis

In `PickupSelection.Continue()`:

- `GetPath()` builds a `FindPath` per remaining target and picks the first usable one.
- `_TargetCell = path.Steps[path.Steps.Count - 1];`
- The gather branch runs when `_TargetCell.IsAdjacentTo(_Player.CurrentCell) || _TargetCell == _Player.CurrentCell`.
- If targets still remain afterwards, execution falls through to the movement block and calls `AutoAct.TryToMove(_Player, _Player.CurrentCell, path.Steps[1], path.Directions[0])`.

When the player is **already standing on** `_TargetCell`, that path has a single step, so `path.Steps[1]` is out of range. The exception is thrown inside the action loop and, if swallowed there, the visible result is exactly "the command does nothing". Standing on the item is the normal case when picking up from your own tile.

## Method

1. Confirm the throw. Add temporary logging around `GetPath()`, `path.Steps.Count`, `_TargetCell`, the branch taken, and whether `TakeObject` returned true. Wrap the movement block so the exception is visible rather than swallowed.
2. Reproduce per `../uat/03` and read `Player.log` — specifically: standing on the item, and standing next to it, each with one target and several.
3. Fix only what the log shows. If it is the index, guard it: when the chosen path has no step to move to, do not attempt the move; continue with the remaining targets or end the action cleanly.
4. If the log shows something else — `TakeObject` failing, `GetPath` returning nothing usable, the action never re-arming — fix that instead and say so plainly.

Watch for a second failure mode while you are in there: after a successful gather, does the action correctly continue to the *next* target, or end early?

## Out of scope — record, do not build

**Containers and chests.** UAT also found pickup does not work on a chest. `BuildPopup` only reads objects sitting in a `Cell` and never recurses into a container's inventory, so chest contents are never candidates in the first place. That is a missing feature, not a regression — adding it is new functionality and belongs to a later branch. Note it in your hand-back for the obsolete/gap inventory in step 17.

## Non-goals

- Do not rewrite the pathing strategy.
- Do not change the menu or candidate collection — step 12 owns `QudUX_QuickPickupPart.cs`.
- Do not add container support.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- Picking up one item while standing on it works.
- Picking up several items across several cells works, and the action ends cleanly when the list is exhausted.
- Temporary logging removed.
- Hand-back names the confirmed cause with its log evidence.

## Re-validates

`../uat/03`, steps 4-7.
