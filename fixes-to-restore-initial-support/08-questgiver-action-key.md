# Step 08 — Fix the quest-giver action key

**Lane B · Depends on: 07 · Owns: `Parts and Effects/QudUX_ConversationHelper.cs` (lines ~340-380)**

Read `00-README.md` first.

## Goal

Make the "already-started quest" conversation choice run the handler meant for it.

## Why

`AddChoiceToIdentifyQuestGivers` builds two choices and gives both the same action key:

```csharp
// line ~348 — ActiveQuestHolders branch ("I'm looking for X")
c.Actions = new Dictionary<string, string>{{ "ApplyNewQuestGiverEffect", null}};

// line ~358 — NewQuestHolders branch ("How can I find X?")
c.Actions = new Dictionary<string, string>{{ "ApplyNewQuestGiverEffect", null}};
```

The first should be `"ApplyActiveQuestGiverEffect"`. The delegate already exists further down the same file (around line 438) and is currently unreachable. As written, asking about an **active** quest giver highlights the `NewQuestHolders` list instead — the wrong creatures, or none at all.

There is a knock-on effect. `RemoveOldQudUXChoices` (line ~363) strips stale choices by looking for exactly that key:

```csharp
if (cChoice != null && cChoice.Actions != null && cChoice.Actions.ContainsKey("ApplyActiveQuestGiverEffect"))
```

Since nothing ever sets it, that cleanup never matches and stale choices from a previous conversation are never removed. Fixing line 348 also restores the cleanup.

## Required change

Change the `ActiveQuestHolders` branch (line ~348) to use `"ApplyActiveQuestGiverEffect"`. Leave the `NewQuestHolders` branch (line ~358) on `"ApplyNewQuestGiverEffect"`.

Before editing, confirm against the file that the delegate name matches exactly, and that it is registered wherever the conversation system looks action keys up. A key with no registered handler is the same silent no-op in the other direction.

Then check whether `RemoveOldQudUXChoices` should also strip `"ApplyNewQuestGiverEffect"` choices — with only the active key listed, new-quest choices may still accumulate. Report what you find; make the change only if the file makes the intent unambiguous.

## Non-goals

- Do not restructure `AddChoiceToIdentifyQuestGivers`.
- Do not change the player-visible choice text.
- Do not touch the event chain — step 07 owns that.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- The two branches use different, correct action keys.
- Your hand-back states whether the effect-applying delegates are reachable now, and what the outcome of the `RemoveOldQudUXChoices` question was.

## Testing caveat

This bug only becomes observable once step 07 makes the choice appear at all. If 07 ends undiagnosed, make the code change anyway — it is correct on its face — and say plainly in the hand-back that it is **unverified in-game**.

## Re-validates

`../uat/07`, the active-quest variant specifically: accept a quest, then ask a different villager where that quest giver is.
