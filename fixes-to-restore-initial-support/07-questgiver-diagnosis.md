# Step 07 — Diagnose why the quest-giver locator never appears

**Lane B · Depends on: nothing · Owns: `Parts and Effects/QudUX_ConversationHelper.cs`, `Harmony Patches/Patch_XRL_World_BeginConversationEvent.cs`**

Read `00-README.md` first.

## Goal

Find out, with runtime evidence, why the "help locate village quest givers" conversation choice never shows up. Then fix what the evidence points at — **and only that**.

This is a diagnosis step. Finishing with a precise, evidence-backed cause and no code change is a valid outcome. Guessing is not.

## Why it is framed this way

UAT found the choice absent from every NPC. A first analysis blamed the Harmony patch binding. **That was wrong**, and so was every other static hypothesis. All of these were checked against the decompiled 1.0.5 assembly and came back intact:

| Hypothesis | Verdict |
|---|---|
| Patch cannot bind to `BeginConversationEvent.Check` | **False.** Exactly one `Check` overload exists, and Harmony matches postfix parameters by name. `Actor`, `SpeakingWith` and `Conversation` all still exist on the 10-parameter signature. |
| `Check` is no longer called | **False.** `ConversationUI` calls it (around line 410 of the decompiled type). |
| The part is not attached to the player | **False.** `PartsAdding/QudUX_PartAdder_OnLoad.cs:20` and `QudUX_PartAdder_Mutator.cs:14` both attach it. |
| The string event is never registered | **Not shown.** `IPart.ApplyRegistrar` still calls the obsolete `Register(GameObject)` overload, which is where `RegisterPartEvent(this, "PlayerBeginConversation")` runs. |
| The conversation node ID changed | **False.** `XRL.World.Parts.DynamicQuestSignpostConversation` still creates `*DynamicQuestSignpostConversationIntro`. |
| The quest-giver property changed | **False.** It still tags and queries `GivesDynamicQuest`. |

A sibling feature in the same file — the restock choice — **does** work, but only through `Patch_XRL_UI_TradeUI`, which calls `SetTraderInteraction` directly and never needs the event. So the event path is unproven in both directions: nothing shows it works either.

## The chain to instrument

1. `Patch_XRL_World_BeginConversationEvent.Postfix` — is it entered at all?
2. `Actor.HasRegisteredEvent("PlayerBeginConversation")` — true or false?
3. `QudUX_ConversationHelper.FireEvent` — is the `E.ID == "PlayerBeginConversation"` branch reached?
4. The gate at `QudUX_ConversationHelper.cs:73`: `questID == string.Empty || XRLCore.Core.Game.FinishedQuests.ContainsKey(questID)`. The choice is only offered by an NPC who is **not** an active quest giver — talking to the quest giver themself is expected to show nothing.
5. `AddChoiceToIdentifyQuestGivers` (line 305) early exits, in order:
   - `Options.Conversations.FindQuestGivers` false → returns
   - the zone scan `GetObjectsWithProperty("GivesDynamicQuest")` yields nothing → both holder lists empty → returns
   - `convo.GetElementByID("Start")` null → active-quest branch skipped
   - `convo.GetElementByID("*DynamicQuestSignpostConversationIntro")` null → new-quest branch skipped
6. If a `Choice` is added, does the conversation UI actually render it? The game's own part rebuilds nodes on `BeforeConversationEvent`, and the mod's postfix runs on `BeginConversationEvent`, which fires after — verify that ordering holds in 1.0.5 rather than trusting the mod's comment.

## Method

- Add temporary logging at each numbered point via `Utilities/Logger` — entry, the boolean, the counts, and which branch exited. Note that `FireEvent` already wraps both calls in `try/catch` that logs to `Debug.Log`, so an exception is being swallowed quietly; make sure your logging would reveal one.
- Build, `./move-mod.ps1`, play, and talk to: a villager who is not a quest giver while a quest giver exists in the same zone (this is the intended trigger), and a quest giver. Joppa works — see `../uat/07` for the setup.
- Read `Player.log`, find the first point in the chain that does not happen, and stop there. That is the cause.
- Fix that one thing. Strip the temporary logging afterwards, or keep it behind the mod's existing logger if it is genuinely useful.

## Non-goals

- Do not re-add a `RequirePart<QudUX_ConversationHelper>()` call — the part is already attached twice over.
- Do not revive `Harmony Patches/Patch_XRL_UI_ConversationUI.cs`. It is fully commented out, its transpiler targets IL that 1.0.5 moved, and step 15 deletes it.
- Do not rewrite the choice-injection approach because a redesign looks cleaner.
- Do not touch the action-key bug at lines 348 and 358 — step 08 owns it, and it is a different bug.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- Your hand-back names the exact failing link with the `Player.log` line that proves it.
- If fixed: the choice appears in-game, with a screenshot path or log line as proof.
- If not fixed: the diagnosis is written up precisely enough that the next agent starts from your last confirmed point, not from scratch.

## Re-validates

`../uat/07`. Re-run `../uat/01` afterwards to confirm the restock choice did not regress.
