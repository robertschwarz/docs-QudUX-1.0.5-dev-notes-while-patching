# Step 05 — Guard the stats screen against an empty games list

**Lane A · Depends on: 02 · Owns: `Screens/QudUX_GameStatsScreen.cs` (the Enter/Space handler)**

Read `00-README.md` first.

## Goal

Stop the enhanced scoreboard from crashing when the player presses Enter or Space with no games recorded.

## Why

`QudUX_GameStatsScreen.cs:86-91` indexes the list with no count check:

```csharp
if (keys == Keys.Enter || keys == Keys.Space && (currentPage == StatsPage.GamesList))
{
    EnhancedScoreEntry esh = ScoreList[scoreTable.Offset + scoreTable.SelectedIndex];
    QudUX_GameDetailsScreen detailsScreen = new QudUX_GameDetailsScreen();
    detailsScreen.GameDetails = esh.Details;
    detailsScreen.Show(GO);
}
```

`Table.Display` in `Utilities/ConsoleUtilities.cs:199-222` only resets `SelectedIndex` when it exceeds `CurrentVisibleRows`. On an empty table `CurrentVisibleRows` is 0 and `SelectedIndex` stays 0, so `0 > 0` is false and the index is never reset. `ScoreList[0]` on an empty list throws. A fresh install with no finished games hits this on the first keypress.

## Required change

Guard the index before it is used: do nothing when `ScoreList` is null or empty, or when the computed index falls outside the list.

Follow the pattern the mod already uses at `Screens/QudUX_RecipeSelectionScreen.cs:86-89` and `Screens/QudUX_IngredientSelectionScreen.cs:254-257`, which bail out early on an empty collection.

While you are in this expression, note that `keys == Keys.Enter || keys == Keys.Space && (currentPage == StatsPage.GamesList)` binds as `Enter || (Space && onGamesList)` because `&&` outranks `||` — so **Enter opens the details screen from any page**, not just the games list. Report it in your hand-back. Fix it only if the guard you add does not already make it harmless; if you do fix it, say so explicitly, since it changes behavior beyond the crash.

## Non-goals

- Do not redesign the table or the selection model.
- Do not add an empty-state message or any other UI.
- Do not touch `Screen Extenders/EnhancedScoreBoard.cs` — it only builds entries and is not implicated.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- With an empty games list, Enter and Space do nothing and throw nothing.
- With a non-empty list, opening the details screen still works exactly as before.

## Re-validates

`../uat/08`, which currently tells the tester to avoid this keypress. Once fixed, pressing Enter on an empty list is a valid test rather than a hazard.
