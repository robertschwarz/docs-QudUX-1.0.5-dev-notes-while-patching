# Error 8 — `Scoreboard2.Load()` is now async

**File:** `Screen Extenders/EnhancedScoreBoard.cs:36`

**Error:**
```
error CS0029: Cannot implicitly convert type 'Task<XRL.Core.Scoreboard2>' to 'XRL.Core.Scoreboard2'
```

## Status: FIXED ✅

`Scoreboard2.Load()` now returns `Task<Scoreboard2>`. Blocked synchronously with `.GetAwaiter().GetResult()` since `Init()` is a sync method.

```diff
- Scoreboard2 highScoreData = Scoreboard2.Load();
+ Scoreboard2 highScoreData = Scoreboard2.Load().GetAwaiter().GetResult();
```

## Agent review (1.0.5)

**Verdict: CONFIRMED FIXED** — baseline diff is correct, no change made.

Deadlock hypothesis (blocking on an await whose continuation posts back to the Unity main thread) does not hold for this call, verified against decompiled 1.0.5 code:

- Decompiled `XRL.Core.Scoreboard2.Load()` (Assembly-CSharp.dll) awaits `UpgradeScoreIfNecessary()` and `Platform.IO.File.ExistsAsync`/`ReadAllJsonAsync`, none of which use `ConfigureAwait(false)`.
- Those bottom out in `Platform.IO.Blob.*Async` (also in Assembly-CSharp), which wrap a `TaskCompletionSource` around `State.GetStorage()` callback APIs.
- The registered `IStorage` backend for this build is `LaundryBear.PlatformServices.None.Storage` (`LaundryBear.Platform.None.dll`) — e.g. `FileExists`/`LoadBlob` call `System.IO.File.Exists`/`ReadAllBytes` synchronously and invoke the callback **inline, on the calling thread**, before returning. So every `TaskCompletionSource` here is completed synchronously — the async state machine in `Load()` never actually suspends/yields, meaning no continuation is ever posted to `SynchronizationContext.Current`. `GetAwaiter().GetResult()` is called on an already-completed (or trivially-completing) task, not one waiting on the main thread's own message pump. No deadlock.
- This exact blocking pattern is also what the game's own `XRL.Core.Scores.Scoreboard` getter does natively in 1.0.5: `Task<Scoreboard2> task = Scoreboard2.Load(); task.Wait(); _Scores = task.Result;` — called synchronously from `Scores.Show()`, the very method QudUX's `Patch_XRL_Core_Scores` Harmony-transpiles. `Scores.Add()` similarly does `Save().Wait()`. Blocking sync-over-async on `Scoreboard2` is the game's own established pattern here, not something QudUX introduced.
- `EnhancedScoreboard.Init()` is called the same way (sync, from `QudUX_GameStatsScreen.FillTables`, itself invoked while showing a screen on the UI/main thread) — consistent with the game's own usage.
- Minor plus: `.GetAwaiter().GetResult()` (baseline) rethrows the original exception directly, vs. the game's `.Wait()`/`.Result` which would wrap it in `AggregateException`. Since the call is already inside a `try/catch (Exception ex)`, `GetAwaiter().GetResult()` is arguably slightly better here.

**Diff applied:** none — baseline patch is unchanged and correct.

**Build result:** clean for this file. Only remaining build error is the known, unrelated issue 11 (`Brain.Factions` in `QudUX_LegendaryInteractionListener.cs`).

**Runtime risks / open questions:**
- This safety analysis is specific to the desktop/Steam build's `None` storage backend (confirmed present and registered in the shipped Managed dir). If a platform build ever swaps in a truly-async `IStorage` implementation (e.g. cloud save backend) that yields to a background thread before completing, this call *would* regain deadlock risk. Out of scope for this fix — flagging for awareness only, no code change warranted since the game's own equivalent code has the identical exposure.
- Blocking the main thread for the duration of the (synchronous, local-disk) file read is a minor responsiveness cost, same as vanilla behavior — not a regression introduced by this fix.
