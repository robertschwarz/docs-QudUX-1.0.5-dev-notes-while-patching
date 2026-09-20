# Step 14 — Remove the dead debug string in the pickup action

**Lane D · Depends on: 12, 13 · Owns: `AutoAct/PickupSelection.cs` (the `pickupDebug` string)**

Read `00-README.md` first. This is the smallest step here; do it last in its lane so it cannot conflict with step 13's work in the same file.

## Goal

Delete a debug string that is built but never used.

## Why

`PickupSelection.Continue()` builds `pickupDebug` (around lines 54 and 67-69) and never logs, shows or returns it. It has no runtime effect beyond wasted allocation, but it actively misleads anyone debugging this file — its "And failed?" and "And succeeded." branches read as though they are inverted, which costs reading time during exactly the investigation step 13 performs.

## Required change

Remove the declaration and every append to it. Nothing else.

If step 13 found that string useful while diagnosing, do not just delete it — turn it into a real log line through the mod's `Utilities/Logger`, or delete it outright. What must not survive is a string that is assembled and discarded.

## Non-goals

- Do not remove other unused members you notice. Report them instead; this branch is not a cleanup pass.
- Do not change any pickup logic.
- Do not reformat the method.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors, and no new unused-variable warnings.
- `grep -n "pickupDebug" "AutoAct/PickupSelection.cs"` returns nothing, or only a deliberate logger call.
- The diff touches only the lines that built the string.

## Re-validates

Nothing user-visible. Confirm `../uat/03` still passes after steps 12 and 13, so this cleanup is not blamed for a later regression.
