# Step 10 — Stop batch marking from duplicating journal notes

**Lane C · Depends on: 09 · Owns: `Parts and Effects/QudUX_LegendaryInteractionListener.cs` (the multi-match loop)**

Read `00-README.md` first.

## Goal

Make repeated "Add zone heroes to journal" runs harmless.

## Why

The single-creature path is guarded: `HandleEvent(OwnerGetInventoryActionsEvent)` calls `JournalAPI.GetMapNote(MakeSecretId(E.Object))` and offers either "Mark" or "Remove Marked", so it never adds twice.

The multi-match branch of `BatchMarkLegendary` has no such check. It loops and calls `JournalAPI.AddMapNote(...)` for every match, every time:

```csharp
for(int i=0; i<legendaryCreatures.Count; i++)
{
    target = legendaryCreatures[i].Object;
    string entryText = $"{target.DisplayNameOnlyDirect}{ParentheticalListOfRelations(target)}";
    string secret = MakeSecretId(target);
    JournalAPI.AddMapNote(target.CurrentZone.ZoneID, entryText, "Legendary Creatures", secretId: secret, revealed: true, sold: true, silent: true);
    ...
}
```

Re-running the command in an already-marked zone re-adds every note under a reused `secretId`. UAT saw duplicate-entry-id complaints in the log, and the command is trivially re-runnable — it is a keybind, not a one-shot.

## Required change

Check `JournalAPI.GetMapNote(secret)` before each `AddMapNote`, and skip creatures that already have a note.

Also make the summary popup honest: it currently lists everything it looped over. Report how many were newly added versus already present, or list only the new ones. Keep it to one popup.

Note the single-creature branch (`legendaryCreatures.Count == 1`) delegates to `MarkLegendaryLocation`, which has **no** internal existence check of its own — it relies on the caller. After step 09 changes how the list is built, confirm that branch still cannot double-add, and guard it if it can.

## Known limitation — report, do not fix

`MakeSecretId` is `$"QudUX_{DisplayNameOnlyDirectAndStripped}_{ZoneID}"`. Two identically-named legendaries in one zone collide on that id. Fixing it means changing the id scheme, which would orphan notes in existing saves — that is a migration, not a bug fix. Record it for branch 2.

Same for the interaction-menu action being one-way rather than a toggle: `ToggleLegendaryLocationMarker` exists but has no callers. Note it; step 15 decides whether it gets deleted.

## Non-goals

- Do not change the id scheme.
- Do not restructure the 0 / 1 / many branching beyond what step 09 required.
- Do not chase the "notes never appear in the journal" problem — that is step 11.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- Running the command twice in the same zone produces no duplicate notes and no duplicate-id warnings in `Player.log`.
- The popup text matches what actually happened.

## Re-validates

`../uat/11`. Note that steps 9-10 of that script cannot fully pass until step 11 explains why notes do not show up at all.
