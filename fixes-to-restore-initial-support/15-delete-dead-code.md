# Step 15 — Delete code that does nothing

**Lane E · Depends on: nothing · Owns: the files and option chain below**

Read `00-README.md` first.

## Goal

Remove code that cannot execute. Not code that is obsolete — code that is **dead**: commented out in full, or wired to nothing.

## The distinction that matters

| | Meaning | Branch 1 action |
|---|---|---|
| **Dead** | Cannot run. Commented out, or has no caller and no trigger. | Delete now (this step). |
| **Obsolete** | Runs fine, but the base game now does the same job — usually only under the modern UI. | Leave the code alone; add an explanatory comment (step 16). Branch 2 decides. |

Deleting obsolete-but-working code here would break it for legacy-UI players, who are exactly the people it still serves.

## Delete

1. **`Screens/QudUX_BuildLibraryScreen.cs`** — every line commented out. Check `Concepts/Options.cs` and `Options.xml` for `QudUX_OptionUseBuildLibrary`, and the commented hook near `Harmony Patches/Patch_XRL_UI_CreateCharacter.cs:50`. If nothing live remains, remove the option too rather than leaving a setting that does nothing.
2. **`Harmony Patches/Patch_XRL_UI_ConversationUI.cs`** — every line commented out since 2021. It held a transpiler that drew the speaker's tile in the conversation title bar, plus a prefix/finalizer pair. Its transpiler targeted IL that 1.0.5 has since moved, so it would not reattach as-is. Step 07 has already been told not to revive it.
3. **The `TrackLocations` option chain** — dead end to end:
   - `Concepts/Options.cs:33`, the `Exploration.TrackLocations` property
   - `Options.xml:4`, the option itself, commented out
   - `Harmony Patches/Patch_XRL_UI_JournalScreen.cs:19`, the only consumer, also commented out

   Because the XML option does not exist, `GetOption` returns empty and the file's `|| IsNullOrEmpty(...)` fallback makes the property always evaluate true — a setting the player cannot control and nothing reads.
4. **`ToggleLegendaryLocationMarker`** in `Parts and Effects/QudUX_LegendaryInteractionListener.cs` — **only if** a repo-wide search proves it has no callers, including command bindings in `Commands.xml`. That file belongs to lane C; coordinate rather than editing it under them, or hand this one item to lane C and note it.

Before each deletion, grep the whole repo for the type, method and option name. "It looks commented out" is not enough; `Commands.xml`, `ObjectBlueprints.xml` and the `PartsAdding/` files can reference things by string name.

## Do not delete

Anything that still runs, including features the base game has superseded: the QudUX inventory screen, the cooking and ingredient screens with their campfire patch, the conversation-portrait path in `Utilities/TileMaker.cs`, and the inventory filter. All of those still work under the legacy UI. Step 16 comments them; branch 2 decides their fate.

## Record what you remove

Deletions have to be recoverable. For each file or block removed, note the path and the commit SHA it was last present in (`git log -1 --format=%H -- <path>` before you delete, or the branch head SHA). Step 17 puts those SHAs in the inventory doc so branch 3 can pull the conversation-portrait transpiler back if it wants it.

## Non-goals

- No refactoring, renaming or reformatting of surviving code.
- Do not delete unused-looking members outside this list. Report them.
- Do not touch files owned by lanes A-D.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors, and no new warnings.
- The mod still loads in-game and the mod list shows no error state.
- Every deletion has a recorded path and SHA.
- `grep -rn "BuildLibrary\|TrackLocations" --include=*.cs --include=*.xml .` returns only intentional survivors, each explained in your hand-back.

## Re-validates

Nothing directly. The check is that the mod loads and the UAT scripts for untouched features still behave — `../uat/01` and `../uat/05` are quick smoke tests.
