# Step 02 — Route every screen through the key helper

**Lane A · Depends on: 01 · Owns: `Screens/*.cs`**

Read `00-README.md` first.

## Goal

Replace every direct `Keyboard.getvk(...)` call in `Screens/` with the helper from step 01, so no QudUX screen can hang when a modern-UI window takes input.

## Call sites

Exactly eight live sites. Line numbers are from the current branch — confirm before editing.

| File | Line | Current call |
|---|---|---|
| `Screens/QudUX_AutogetManagementScreen.cs` | 112 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |
| `Screens/QudUX_CharacterTileScreen.cs` | 199 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |
| `Screens/QudUX_GameDetailsScreen.cs` | 44 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |
| `Screens/QudUX_GameStatsScreen.cs` | 78 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |
| `Screens/QudUX_IngredientSelectionScreen.cs` | 376 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |
| `Screens/QudUX_InventoryScreen.cs` | 589 | `ConsoleLib.Console.Keyboard.getvk(Options.MapDirectionsToKeypad, true)` |
| `Screens/QudUX_QuickPickupSettingsScreen.cs` | 212 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |
| `Screens/QudUX_RecipeSelectionScreen.cs` | 173 | `Keyboard.getvk(Options.MapDirectionsToKeypad)` |

**`QudUX_InventoryScreen.cs:589` passes `pumpActions: true`** — preserve that. The other seven use the default.

**Skip `Screens/QudUX_BuildLibraryScreen.cs`.** Its `getvk` on line 156 sits inside a fully commented-out file. Step 15 deletes it.

## Required change

Swap each call for the helper. Nothing else in these loops changes — same variable names, same key comparisons, same exit conditions.

Each screen's loop already exits on `Keys.Escape` (usually `Escape || NumPad5`), which is why the helper returns `Escape` on bail-out. **Verify that per screen**, because one differs: `QudUX_CharacterTileScreen.cs:202` only exits on Escape when `screenMode == CoreTiles`; in other modes Escape changes mode and loops again. Report in your hand-back whether a starved bail-out there escapes fully or only steps back one level. Do not restructure that screen — just report it.

## Non-goals

- No changes to key handling, rendering, or screen flow.
- Do not tidy the loops you are editing.
- Do not touch popup calls in these files. Step 03 owns those.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- `grep -rn "Keyboard.getvk" Screens/` returns only the commented-out line in `QudUX_BuildLibraryScreen.cs`.
- The diff is one changed line per screen, plus a `using` if needed. A diff showing whole reformatted files means the line endings were rewritten — redo it.

## Re-validates

`../uat/06` (sprite menu) and `../uat/08` (scoreboard) are the two screens where the freeze was actually observed. The real test: open one, cause a modern-UI popup to appear over it, and confirm the screen exits on its own instead of locking the game.
