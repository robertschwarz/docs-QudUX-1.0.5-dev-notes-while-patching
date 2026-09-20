# Error 7 — `RemoveEffect` takes `Effect`, not `string`

**File:** `Parts and Effects/QudUX_ConversationHelper.cs:471`

**Error:**
```
error CS1503: Argument 1: cannot convert from 'string' to 'XRL.World.Effect'
```

## Status: FIXED ✅

`RemoveEffect` no longer accepts a string name. Get the live effect instance first via `GetEffect<T>()`.

```diff
- questGiver.RemoveEffect("QudUX_QuestGiverVision");
+ questGiver.RemoveEffect(questGiver.GetEffect<QudUX_QuestGiverVision>());
```

## Agent review (1.0.5)

**Verdict: CONFIRMED FIXED** (no change made — baseline patch is correct).

**Verified against decompiled 1.0.5 `XRL.World.GameObject` / `XRL.World.Effect`:**

- `RemoveEffect(Effect E, bool NeedStackCheck = true)` exists and is the only single-effect-instance overload:
  ```csharp
  public bool RemoveEffect(Effect E, bool NeedStackCheck = true)
  {
      if (E == null || _Effects == null) return false;
      int num = Array.IndexOf(_Effects.GetArray(), E, 0, _Effects.Count);
      if (num == -1) return false;
      RemoveEffectAt(num, NeedStackCheck);
      return true;
  }
  ```
  Null-safe — passing a `null` `Effect` (e.g. if `GetEffect<T>()` finds nothing) just returns `false`, no exception. So the surrounding `HasEffect(...)` guard is belt-and-suspenders, not strictly required, but harmless.
- `GetEffect<T>()` (line ~8683) matches by exact type (`GetType() == typeof(T)`) and returns the live instance or `null` — correct replacement for the removed string-name lookup, and type-safe.
- `HasEffect(string EffectType)` (line ~8321) **still exists** in 1.0.5 and matches on `effect.ClassName` where `Effect.ClassName => _ClassName ?? (_ClassName = GetType().Name)` — confirmed this resolves correctly for mod effect classes like `QudUX_QuestGiverVision` (defaults to the runtime type name, no registration needed). One subtlety: `HasEffect(string)` additionally requires `effect.Duration > 0`, while `GetEffect<T>()` has no duration filter — a non-issue here since only one instance of this effect is ever applied to a quest-giver at a time.
- Old pre-1.0.5 `RemoveEffect(string)` semantics (single first-match removal) are preserved: `RemoveEffect(Effect)` removes exactly the one instance passed in, which is the same effect `HasEffect`/`GetEffect` located. Also checked `RemoveEffect<T>()` (line ~6323) and `RemoveEffect(Type)` (line ~6305) as alternates — both remove only the *first* type-match in the effect list (same first-match semantics as the old string API), so any of the three would be behaviorally equivalent here; `RemoveEffect(GetEffect<T>())` is the most explicit/minimal-diff choice and was kept.
- `QudUX_QuestGiverVision` lives in `namespace XRL.World.Effects` (`Parts and Effects/QudUX_QuestGiverVision.cs`); `QudUX_ConversationHelper.cs` already has `using XRL.World.Effects;` at the top, so the type resolves with no new using needed.

**Build result:** `dotnet build Mods.csproj` — zero errors on this file/line. Only remaining baseline error is `Brain.Factions` in `QudUX_LegendaryInteractionListener.cs` (issue 11, not this issue), unaffected.

**Diff applied:** none — `git diff` against baseline for this issue's scope is empty.

**Runtime risks / open questions:** none identified. The `HasEffect` guard combined with the null-safe `RemoveEffect(Effect)` means even a race where the effect is removed between the check and the call would not throw.
