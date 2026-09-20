# Error 5 — `Restocker` type not found

**File:** `Parts and Effects/QudUX_ConversationHelper.cs:166`

**Error:**
```
error CS0246: The type or namespace name 'Restocker' could not be found
```

## Status: FIXED ✅

DLL reflection confirms `Restocker` no longer exists. `GenericInventoryRestocker` is the only restocker type in 1.0.5.

The original code had two branches:
- `if (speaker.HasPart("Restocker"))` — used `r.NextRestockTick - XRLCore.CurrentTurn`
- `else if (speaker.HasPart("GenericInventoryRestocker"))` — used `r.RestockFrequency - (CurrentTurn - r.LastRestockTick)`

The `Restocker` branch was removed entirely. The `GenericInventoryRestocker` branch was promoted from `else if` to `if`.

```diff
- if (speaker.HasPart("Restocker"))
- {
-     _debugSegmentCounter = 7;
-     Restocker r = speaker.GetPart<Restocker>();
-     ticksRemaining = r.NextRestockTick - XRLCore.CurrentTurn;
-     _debugSegmentCounter = 8;
- }
- else if (speaker.HasPart("GenericInventoryRestocker"))
+ if (speaker.HasPart("GenericInventoryRestocker"))
```

## Agent review (1.0.5)

**Verdict: CONFIRMED FIXED** (no change needed, patch = 0 lines)

**Verified:**
- `ilspycmd -l c` over full `Assembly-CSharp.dll`, grep `-i restock`: only hits are `XRL.World.Parts.GenericInventoryRestocker` (+ its compiler-generated `<>c__DisplayClass24_0`). No `Restocker` type anywhere in 1.0.5. Confirms branch removal is correct, not just a workaround.
- Decompiled `GenericInventoryRestocker` (via `-t`): has `public long LastRestockTick;`, `public long RestockFrequency = 6000L;`, `public int Chance = 100;`. Field names/types used in the surviving code (`r.RestockFrequency`, `r.LastRestockTick`, both `long`) match exactly — same semantics as pre-1.0.5 (tick-based countdown), no signature drift.
- `Chance` field exists but isn't referenced by the surviving code path — that's pre-existing behavior (the code already only used `bChanceBasedRestock` as a flag for dialog text, never read `Chance`), not something introduced by this fix. Out of scope for issue 05.
- Grepped `ObjectBlueprints/Creatures.xml` (StreamingAssets/Base) for `restocker`: every vendor part reference is `GenericInventoryRestocker` (Tier1-8 Wares, Village Apothecary tiers, farmer inventories, etc.) — zero uses of a bare `Restocker` part. So no vendor silently loses restock dialog from dropping that branch.
- Post-if/else code path: `ticksRemaining` is assigned on both remaining paths (`if` branch assigns; `else` returns false before use) — compiler doesn't flag definite-assignment issues.
- Lines 11 (using directive, issue 01) and ~463 (RemoveEffect, issue 07) in this file untouched — confirmed by inspecting the file; only the restock-timer section (~140-170) was ever in play, and it was already correctly patched in baseline.

**Diff applied:** none (`patches/05.patch` is empty — baseline fix from the doc was already correct).

**Build result:** `dotnet build Mods.csproj` — zero errors from this file/section. Only remaining error in the tree is the pre-existing, out-of-scope `Brain.Factions` issue (11) in `QudUX_LegendaryInteractionListener.cs`.

**Runtime risks / open questions:** none identified. `bChanceBasedRestock` is now always `true` when reached (single surviving branch), which only affects flavor-text wording ("I can't make any guarantees" caveat) — cosmetic, matches original intent for the generic/chance-based restocker.

**Status:** FIXED ✅ (unchanged)
