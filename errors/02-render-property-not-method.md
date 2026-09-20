# Error 2 — `Render` is now a property, not a method

**File:** `Utilities/TileMaker.cs:155`

**Error:**
```
error CS1955: Non-invocable member 'GameObject.Render' cannot be used like a method.
```

## Status: CORRECTED ✅

`Render` is now a property (the Render component part) on `GameObject`, not a callable method.
`RenderEvent` no longer extends `Event` in 1.0.5 (base type is `System.Object`), so `FireEvent(renderData)` also fails.

`GameObject` now has explicit methods for rendering that accept `RenderEvent` directly:
- `ComponentRender(RenderEvent E)` ← correct replacement
- `OverlayRender(RenderEvent E)`
- `FinalRender(RenderEvent E)`

```diff
- go.Render(renderData);
+ go.ComponentRender(renderData);
```

## Agent review (1.0.5)

**Verdict: CORRECTED.** Baseline fix compiled but was incomplete — `ComponentRender` alone does not reproduce the old `go.Render(renderData)` pipeline's behavior.

**What was verified (decompiled 1.0.5 `Assembly-CSharp.dll`):**

- `XRL.World.GameObject` exposes three render-stage methods: `ComponentRender(RenderEvent)`, `OverlayRender(RenderEvent)`, `FinalRender(RenderEvent)` — each just dispatches to parts registered for that stage's event ID. No single method wraps "the full pipeline" generically.
- `GameObject.RenderForUI(string Context = null, bool AsIfKnown = false)` is the game's own canonical "give me the full rendered tile for one object" helper (used e.g. by `Weapon.RenderForUI()` for combat-juice tile lookups — the same kind of use case as `TileMaker`). Its body:
  ```
  ComponentRender(_contextRender);
  FinalRender(_contextRender);
  ```
  It deliberately does **not** call `OverlayRender` — that stage is only invoked from `Cell.cs` for the "Alt" (stacked/composite multi-object) cell-rendering path, not for a single object's own tile.
- Confirmed in `Cell.cs`'s per-object render loop: `ComponentRender` and `FinalRender` are called for every rendered object; `OverlayRender` is called only in the `Alt`-cell branch. This matches `RenderForUI`'s choice of stages and confirms `FinalRender` is part of the normal single-object pipeline, not an edge case.

**Why not just call `go.RenderForUI()` directly:** `TileMaker.Initialize` builds its own `RenderEvent` and pre-populates `Tile`/`ColorString` with custom logic (Examiner "Unknown" tile override for unidentified items) before invoking render. `RenderForUI()` unconditionally overwrites `_contextRender.Tile`/`ColorString`/etc. from `Render` directly, which would silently drop the Examiner-unknown-tile behavior. So the correct minimal fix keeps `TileMaker`'s own `RenderEvent` construction and just adds the missing pipeline stage.

**Exact diff applied** (on top of baseline's `Render` → `ComponentRender` swap):
```diff
             if (renderOK) //we can't render blueprint-created objects, because the game will throw errors trying to check their current cell
             {
                 go.ComponentRender(renderData);
+                go.FinalRender(renderData);
             }
```

**Build result:** `dotnet build Mods.csproj` — zero errors from `TileMaker.cs`/render lines. Only remaining error is the known, out-of-scope `Brain.Factions` issue (#11, `QudUX_LegendaryInteractionListener.cs`), unchanged and untouched.

**Runtime risks / open questions:**
- `FinalRender` may call `IsPlayerControlled`/HP-color logic indirectly through registered part handlers (mirrors what `ComponentRender` already does for status colors) — low risk, same object, same event instance, no new side effects beyond what stock game code does for the same object in `Cell.cs`/`RenderForUI`.
- Not verified against a live game session (no test harness available in this task) — verified via decompiled bytecode/static analysis only, per task scope.
