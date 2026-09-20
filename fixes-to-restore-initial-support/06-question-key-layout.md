# Step 06 — Make the help key work on non-English keyboards

**Lane A · Depends on: 02 · Owns: `Screens/QudUX_GameStatsScreen.cs` (the `?` handler)**

Read `00-README.md` first.

## Goal

Let the quick-keys help open on a German keyboard, where it currently cannot be triggered at all.

## Why

`QudUX_GameStatsScreen.cs:94` tests one physical key:

```csharp
if (keys == Keys.OemQuestion)
{
    ShowQuickKeys();
}
```

`Keys.OemQuestion` is a physical key position, not the character `?`. On a German layout `?` is Shift+ß, which is a different position, so the help is unreachable. UAT confirmed it works on an English keyboard and not on a German one. The game's own fix for non-English layouts (build 210.10 / 1.0.4) cannot help here, because this comparison bypasses the keybind system entirely.

## Required change

Accept the character as well as the key position. The mod already does this correctly at `Screens/QudUX_InventoryScreen.cs:607`:

```csharp
keys == Keys.OemQuestion || ch == '?'
```

`QudUX_GameStatsScreen` does not currently read the typed character. `ConsoleLib.Console.Keyboard.Char` holds it — `QudUX_CharacterTileScreen.cs:200` reads it with `Convert.ToChar(Keyboard.Char)`. Use whichever of those two patterns fits this screen with the least disruption, and keep the existing `Keys.OemQuestion` test so English layouts are unaffected.

Guard the character conversion: `Keyboard.Char` can be 0 for non-character keys, and `Convert.ToChar(0)` is a valid but meaningless char — make sure it cannot match.

## Non-goals

- **Do not introduce the rebindable-command system.** The correct long-term fix is to route this through the game's command table, but that pattern does not exist anywhere in the mod and adding it is far beyond a bug fix.
- Do not audit or convert the mod's other hardcoded keys. There are dozens; they work on English layouts and are out of scope. Note them in your hand-back if you want them tracked.
- Do not change what `ShowQuickKeys()` displays.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- The help opens from both the `?` character and the `Keys.OemQuestion` position.
- No other key handler in the screen changes behavior.

## Re-validates

`../uat/08`, which recorded this failure. Test on the German layout that reproduced it; confirm on an English layout that nothing regressed.
