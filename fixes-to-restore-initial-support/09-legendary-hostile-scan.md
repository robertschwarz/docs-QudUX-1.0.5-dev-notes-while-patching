# Step 09 — Make batch marking find hostile legendaries

**Lane C · Depends on: nothing · Owns: `Parts and Effects/QudUX_LegendaryInteractionListener.cs` (`BatchMarkLegendary`)**

Read `00-README.md` first.

## Goal

Make the "Add zone heroes to journal" command find legendary creatures that are hostile to the player — which is most of them.

## Why

`BatchMarkLegendary` builds its list from the points-of-interest system:

```csharp
List<PointOfInterest> legendaryCreatures = GetPointsOfInterestEvent.GetFor(The.Player)
.Where(
    point =>
    (point.Object.HasProperty("Hero") || point.Object.GetStringProperty("Role") == "Hero") && point.Object.HasPart(typeof(GivesRep))
).ToList();
```

A creature only reaches that list if `GetPointsOfInterestEvent.StandardChecks` passes, and those checks require `!Object2.IsHostileTowards(gameObject)`. Points of interest are a "notable friendly thing nearby" feature. Hostile legendaries are excluded by design, so the count is 0 and the player gets "You haven't noticed any legendary creature here." even while standing next to one. UAT 11.3 reproduced exactly that.

The single-creature path does not have this problem: it runs from the interaction menu on a creature the player already targeted.

## Required change

Replace the points-of-interest query with a direct zone scan that keeps the same two predicates — `Hero` property or `Role == "Hero"`, **and** a `GivesRep` part — but does not filter on hostility.

Points to settle by reading the decompiled 1.0.5 code, not by assuming:

- Which zone-enumeration call to use. `Zone.GetObjects(Predicate<GameObject>)` is what the game's own `DynamicQuestSignpostConversation` uses. Confirm the overload and its namespace.
- Whether to require visibility. "You haven't noticed" implies the player should not be able to journal a creature they have never seen. Check what the game offers — something in the `IsVisible` / seen-cell family — and prefer a check that matches the message's promise. If nothing suitable exists, mark it `⚠️ unverified` and say what you chose.
- Keep the existing message for the genuinely-empty case.

The result feeds the existing count-based branches (0 / 1 / many), so hand back a `List<GameObject>` shape the rest of the method already expects, or adjust the three branches consistently.

## Non-goals

- Do not extend this beyond the current zone.
- Do not change the journal-writing call — step 10 owns idempotency and step 11 owns the writes themselves.
- Do not change the single-creature interaction-menu path.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- In a zone with a hostile legendary, the command finds it.
- In a zone with none, the existing message still appears.

## Re-validates

`../uat/11`, step 7 specifically. Spawn with `testhero:Baboon` or `testhero:Snapjaw` — both are hostile, which is why the UAT never saw this work.
