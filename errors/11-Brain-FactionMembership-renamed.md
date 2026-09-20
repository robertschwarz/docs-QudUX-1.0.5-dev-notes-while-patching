# Error 11 — `Brain.FactionMembership` renamed

**File:** `Parts and Effects/QudUX_LegendaryInteractionListener.cs:128`

**Error:**
```
error CS1061: 'Brain' does not contain a definition for 'FactionMembership'
```

## Status: FIXED ✅

### What was tried

DLL reflection found `Brain.Factions` (string property) and `Brain.FillFactionMembership(IDictionary<string,int> Map, string Spec)` (static).

Attempted fix:
```csharp
var factionMap = new Dictionary<string, int>();
Brain.FillFactionMembership(factionMap, legendaryCreature.Brain.Factions);
foreach (string key in factionMap.Keys)
```

**This fails** because `Brain.Factions` has **no getter** (write-only property). Error:
```
error CS0154: The property or indexer 'Brain.Factions' cannot be used in this context because it lacks the get accessor
```

### What's known (from DLL reflection)

Brain public fields:
- `StringMap<int> FactionFeelings` — maps faction → feeling value (readable field)
- `PartyCollection PartyMembers`

Brain methods related to factions:
- `static FillFactionMembership(IDictionary<string,int> Map, string Spec)` — parses a spec string into a map
- `static GetPrimaryFaction(string FactionSpec)` / `GetPrimaryFaction(bool Base)`
- `static ExtractFaction(string spec)`
- `bool IsFactionMember(string Faction)`
- `string GetPrimaryFactionName(bool VisibleOnly, bool Formatted, bool Base)`

### Pending work

Need to determine:
1. Whether `Brain.FactionFeelings` keys ARE the faction memberships (or if it's feelings-only)
2. If not, find the readable backing field or method that returns which factions the creature belongs to

Likely candidates:
- Iterate `legendaryCreature.Brain.FactionFeelings.Keys` — if this contains faction memberships
- Find a backing field (private/internal) that `Factions` setter writes to

Current broken code in repo:
```csharp
var factionMap = new Dictionary<string, int>();
Brain.FillFactionMembership(factionMap, legendaryCreature.Brain.Factions);  // FAILS: no getter
foreach (string key in factionMap.Keys)
```

Original code that needs to be replaced:
```csharp
foreach (string key in legendaryCreature.Brain.FactionMembership.Keys)
```

## Agent review (1.0.5)

**Verdict: FIXED**

### What was verified (decompiled 1.0.5 `Assembly-CSharp.dll`)

- `XRL.World.Parts.Brain` field: `public AllegianceSet Allegiance = new AllegianceSet();` — this is the live, per-creature faction-membership map (weight per faction), i.e. the direct successor of the old `FactionMembership` dictionary. Evidence it's membership (not feelings):
  ```csharp
  public void SetFactionMembership(string Faction, int Feeling)
  {
      Allegiance[Faction] = Feeling;
  }
  ```
  and the write-only `Factions` property setter itself writes into the base allegiance set:
  ```csharp
  public string Factions
  {
      set { FillFactionMembership(FindAllegiance(0), value); }
  }
  ```
  (`FindAllegiance(0)` returns the base `AllegianceSet` — same object graph as `Brain.Allegiance`.)

- `FactionFeelings` (`StringMap<int>`) is confirmed to be a *separate* concept — opinions/reputation, not membership — used in `GetBaseFactionFeeling`/`ClearHostileFactionFeelings`, distinct from `Allegiance`. Correctly avoided.

- `XRL.World.AI.AllegianceSet : StringMap<int>` — confirmed dictionary-like: `sealed class AllegianceSet : StringMap<int>`. The game itself enumerates it as key/value pairs elsewhere in `Brain`, e.g.:
  ```csharp
  foreach (KeyValuePair<string, int> item in Allegiance)
  ```
  (in `InSameFactionAs`). `StringMap<T>` implements `IDictionary<string, T>` but `Keys` is an **explicit** interface implementation (`ICollection<string> IDictionary<string, T>.Keys`), so it's not visible on the concrete/derived type directly — must go through the `IDictionary<string,int>` interface to reach `.Keys`.

### Final code

```csharp
foreach (string key in ((IDictionary<string, int>)legendaryCreature.Brain.Allegiance).Keys)
```

Replaces the broken baseline attempt (`Brain.FillFactionMembership(factionMap, legendaryCreature.Brain.Factions)` — failed because `Factions` has no getter). Loop body below this line is untouched.

### Build result

`dotnet build Mods.csproj -nologo -v q` → 0 errors.

### Runtime risks / open questions

- Not runtime-tested in-game (no live Qud process in this environment) — only compile-verified against the 1.0.5 decompiled API surface.
- Semantic equivalence assumed: old `FactionMembership` (per-faction membership weight) ≈ new `Allegiance` (per-faction weight, `AllegianceLevel` derived from value). This matches how `SetFactionMembership` writes into `Allegiance` and how the old `Factions` setter routed into the same base allegiance set, but the exact value semantics (weight thresholds via `GetAllegianceLevel`) weren't independently re-verified against 1.0.4 behavior — low risk since this code path only reads keys (faction names) for a "Loved by {faction}" display string, not the numeric values.
