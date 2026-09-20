# Step 01 — Add a non-blocking key helper

**Lane A · Depends on: nothing · Owns: `Utilities/InputsUtilities.cs`**

Read `00-README.md` first.

## Goal

Add one shared helper that reads a key **without blocking forever**. This step only adds the helper. Step 02 makes the screens use it.

## Why

Every QudUX legacy screen calls `Keyboard.getvk(...)`, which waits on a key queue the game stops feeding while a modern-UI window is visible. The screen then waits forever and the player has to kill the game. This is the mod's worst bug: it is reachable from the sprite menu and the scoreboard, and it costs the player their session.

## Current state

`Utilities/InputsUtilities.cs` is 31 lines and holds only `Keys` comparison extensions (`IsMouseEvent`, `Is`, `IsControl`). There is no input-reading wrapper anywhere in the mod — each screen calls the engine directly. You are creating that wrapper.

## Verified API facts

Check these yourself before relying on them; all are in the decompiled assembly.

- `ConsoleLib.Console.Keyboard.getvk(bool MapDirectionToArrows, bool pumpActions = false, bool wait = true)` — with `wait: false` it returns `Keys.None` immediately when no key is queued, instead of blocking.
- `Keyboard.kbhit()` reports whether input is pending, and waits up to 200 ms internally when the game thread does not have focus — useful as a throttle.
- `Keyboard.KeyEvent` is a `ManualResetEvent`; `Keyboard.IdleWait()` waits on it with the animation-rate timeout.
- `Qud.UI.UIManager.instance` is public static, and `PassthroughOnTop()` is a public instance method that returns false when a visible modern-UI window is swallowing input.

## Required change

Add a public static method to `InputsUtilities` — suggested shape, adjust naming to match the file's style:

```csharp
public static Keys WaitForKey(bool mapDirectionsToKeypad, bool pumpActions = false)
```

Behavior:

1. Loop, calling `Keyboard.getvk(mapDirectionsToKeypad, pumpActions, wait: false)`.
2. On any key other than `Keys.None`, return it immediately.
3. Otherwise idle briefly before the next poll — `Keyboard.kbhit()` or a short `KeyEvent.WaitOne(ms)`. **Never hot-spin**: a tight loop with no wait pegs a core and tanks the frame rate.
4. Track how long input has been starved. Once it passes a threshold (~5 seconds) **and** `UIManager.instance?.PassthroughOnTop() == false`, stop waiting and return `Keys.Escape`, so the calling screen exits through the path it already has for Escape.
5. Log the bail-out once through the mod's existing `Utilities/Logger`, with enough detail to recognize it in `Player.log`. Do not log per poll.

Guard every `UIManager` access against null — it is a Unity singleton and can be null early in startup or outside a running game.

## Non-goals

- Do not modify any screen. That is step 02.
- Do not introduce the game's rebindable-command system.
- Do not change how keys are interpreted once returned.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- The helper is `public` (step 02 wires it up; an unused-private warning is not acceptable).
- The threshold and the log message live in one obvious place so they can be tuned.

## Notes for the hand-back

State which idle mechanism you chose, why, and the threshold value. Step 02 and the in-game freeze test both depend on those choices.
