# Step 11 — Find out why journal notes never appear

**Lane C · Depends on: nothing (runs alongside 09/10) · Owns: investigation; code changes only if proven**

Read `00-README.md` first.

## Goal

Explain why marking a legendary creature reports success but leaves the journal empty.

**This is time-boxed.** If you cannot prove a cause, write up what you eliminated and stop. A documented dead end is worth more than a speculative patch to journal writes.

## Why

UAT 11.2: the player marks a creature, the confirmation popup appears, and the journal shows nothing — and the tester noted **all** journal categories looked empty, which may mean the problem is wider than this feature.

Static analysis found nothing wrong. The mod calls:

```csharp
JournalAPI.AddMapNote(target.CurrentZone.ZoneID, entryText, "Legendary Creatures",
                      secretId: secret, revealed: true, sold: true, silent: true);
```

and the 1.0.5 `JournalAPI.AddMapNote` / `GetMapNote` / `DeleteMapNote` signatures and bodies look compatible. So the answer is at runtime, or in how the journal screen filters what it displays.

Also relevant: `Utilities/JournalUtilities.cs` is **not** what its name suggests. It holds ruin-name generation and a zone-cache check, no journal write helpers. Do not go looking there.

## Lines of inquiry

Work outward from the write:

1. **Does the write happen and succeed?** Log immediately after `AddMapNote`, then read back with `GetMapNote(secret)` in the same frame. If the read-back finds it, the write is fine and the problem is display or persistence.
2. **Does it survive a save/load?** Map notes are game state. Mark, save, reload, read back.
3. **Is the category real?** The mod passes the literal `"Legendary Creatures"` and the popup references `JournalScreen.STR_LOCATIONS`. Decompile the journal screen and check how it groups notes, whether a category must be known in advance, and whether an unknown category silently renders nothing.
4. **Do the flags mean what the mod assumes?** Decompile `AddMapNote` and check `revealed`, `sold` and `silent` in 1.0.5. `sold: true` in particular reads like "already sold to the player by a gossip" — confirm it does not exclude the note from display.
5. **Is the whole journal empty, or just this category?** Check a vanilla-generated note (a village quest, a gossip-revealed location). If vanilla notes are missing too, this is not a QudUX bug at all, and that conclusion ends the step.

## Method

Temporary logging through `Utilities/Logger`, build, `./move-mod.ps1`, reproduce with `../uat/11`, then read `Player.log`. Copy the log aside before relaunching — it is overwritten each start.

## Non-goals

- Do not change the journal call "to see if it helps". Every hypothesis gets tested before it gets coded.
- Do not add a QudUX journal screen or a fallback display.
- Do not touch the batch-marking logic — steps 09 and 10 own it.

## Acceptance

One of:

- **Cause found:** named precisely, with the log evidence, plus the minimal fix if it is in the mod's code.
- **Not the mod:** evidence that vanilla notes behave the same way, which reclassifies this as an upstream issue.
- **Time box hit:** a written list of what was eliminated and what to try next, in this file or the hand-back.

In all three cases `dotnet build Mods.csproj` must still report 0 errors and any temporary logging must be removed or made permanent deliberately.

## Re-validates

`../uat/11`, steps 3-6.
