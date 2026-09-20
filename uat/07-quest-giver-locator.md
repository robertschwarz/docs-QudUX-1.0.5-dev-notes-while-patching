# UAT-07 — Help Locate Village Quest Givers

## What was fixed
The conversation feature that lets a villager tell you where to find another village's quest-giver used to fail to build at all (a compile error: `RemoveEffect` was called with a string instead of the actual effect object). The line now correctly finds and removes the *existing* highlight effect before reapplying it, so the feature compiles and — critically — works correctly if you ask for directions a second time while the highlight is still active, instead of crashing or stacking duplicate highlights.

- Fixed file: `Parts and Effects/QudUX_ConversationHelper.cs:463` (`ApplyQuestGiverEffect`)
- Effect applied: `Parts and Effects/QudUX_QuestGiverVision.cs`

## Feature under test
In any conversation with a village NPC, QudUX adds an extra dialogue choice that asks the NPC to point you toward another villager who has a quest to give (or one whose quest you've already started). Picking it pops up a short flavor line and makes that other villager's tile flash on the map so you can find them.

Where it lives: this is **not** a menu — it's an extra line of dialogue that appears automatically inside normal NPC conversations, at the very end of the choice list, only when there's another quest-giving villager in the same zone.

## Preconditions
- QudUX option (Options menu → QudUX category): **"Add conversation option to help locate village quest givers"** = **Yes** (this is the default; `Concepts/Options.cs` → `Options.Conversations.FindQuestGivers`).
- No specific game options required.
- Fresh or early-game save with `qudux-32-test-uat-character`. You need to be able to reach a village that currently has an active "dynamic quest" (a procedurally generated village side-quest, e.g. "find a site for us"). See Setup below — the exact NPC and village are chosen **randomly per playthrough** by the game (`XRL.World.ZoneBuilders.*_FabricateQuestGiver.BuildZone` picks a random eligible named villager the first time a village zone is built), so this script cannot name the NPC in advance. ⚠️ Whether Joppa specifically already has one of these quests active for a brand-new character is not something we could confirm by reading code alone — treat the Joppa attempt in Setup as a first try, and use the wish fallback if it doesn't pan out.

## Setup
1. Load `qudux-32-test-uat-character` and make sure you're in or near Joppa (the normal starting village).
2. **Try Joppa first (optional, ⚠️ unverified whether it will already have one):** walk around Joppa and talk to a few of the named villagers (not generic guards/farmers — the notable/unique NPCs). If one of them offers to give you a quest (e.g. "find a site," "recover an item," "interact with something"), that's your quest giver — skip to Setup step 4.
3. **Guaranteed fallback (verified from source):** open the wish prompt (see `uat/00-setup.md`) and enter:
   ```
   godynamicquest:site
   ```
   This calls `XRL.World.DynamicQuestsGameState.Wish` (`[WishCommand("godynamicquest", ...)]`), which finds (building it if needed) a village with an unclaimed "find a site" dynamic quest and teleports you next to its quest-giving NPC directly. Note the NPC's name and the zone/village name for your evidence notes — they will vary by run.
4. Talk to that quest-giving NPC. Accept the quest by selecting the choice that starts **"Yes. I will locate ... as you ask."** (exact wording confirmed in `FindASpecificSiteDynamicQuestTemplate_FabricateQuestGiver.addQuestConversationToGiver`). This marks the quest as started (`Game.HasQuest` becomes true), which is required for the deterministic dialogue line used in the tests below. End the conversation.
5. Locate a **second, different** NPC in the same village (any other talkable villager works — it does not need to be another quest giver). This is who you'll actually talk to in the test steps.

## Test steps

1. **Action:** Start a conversation with the second NPC (not the quest giver).
   **Expected:** The conversation's opening list of choices includes a line reading exactly `I'm looking for <QuestGiverName>.` (built by `QudUX_ConversationHelper.StatementLocationOf`; `<QuestGiverName>` is the quest giver's display name, e.g. "I'm looking for Argyve.").
   **Evidence:** 📸 `UAT-07-01-choice-present.png` — full dialogue choice list visible, including this line, in `qudux-mod-docs/uat/evidence/07/`.

2. **Action:** Select the "I'm looking for..." choice.
   **Expected:** A popup appears with one of three random lines (from `ApplyQuestGiverEffect`): "...points you in the right direction.", "...gives you a rough layout of the area.", or "...gestures disinterestedly, sending you on your way." The conversation then ends.
   **Evidence:** 📸 `UAT-07-02-popup.png` — the popup text visible on screen.

3. **Action:** Travel back to where the quest giver is standing (don't talk to them yet) and look at their tile.
   **Expected:** The quest-giver's tile is now visibly flashing/highlighted: normally it alternates between their real sprite and a green `*` marker (`QudUX_QuestGiverVision.FinalRender`, color code `&G`); if they're out of sight/unlit it instead flashes a dim gray `*`/silhouette. This is the "vision" effect made visible.
   **Evidence:** 🎥 `UAT-07-03-highlight.mp4` — a few seconds of video showing the tile visibly alternating/flashing on the quest giver's location. (A single screenshot may miss the flash, since it's a blink between two draw states — video is safer.)

4. **Action (the fixed path):** While the highlight from step 3 is still active, go talk to an NPC again (the same second NPC, or a third one) and select the "I'm looking for `<QuestGiverName>`." choice a **second time**.
   **Expected:** No crash, no error popup, no freeze. A popup shows again (one of the same three random lines). Go back and check the quest giver's tile: there is still exactly **one** flashing `*` marker on them — not two overlapping markers, not a stuck/broken sprite. This exercises the exact line that was fixed (the old effect instance is now correctly found and removed via `GetEffect<QudUX_QuestGiverVision>()` before the new one is applied).
   **Evidence:** 📸 `UAT-07-04-no-duplicate.png` — quest giver's tile after the second ask, showing a single marker (or single flash frame), no stacked graphics, no error dialog on screen.

5. **Action (expiry / end condition):** Walk up to the highlighted quest giver and start a conversation with them directly.
   **Expected:** The highlight disappears immediately — `QudUX_QuestGiverVision.FireEvent` removes the effect the instant a conversation with the player begins (`BeginConversation` → `BadListener()` → `RemoveEffect(this)`). After backing out of the conversation, their tile renders normally with no more flashing.
   **Evidence:** 📸 `UAT-07-05-expired.png` — quest giver's tile shown normal/unhighlighted immediately after ending that conversation.

## Edge / negative cases

6. **Action (cancel):** Repeat Setup step 5 flow but this time, when the "I'm looking for..." choice is shown, back out of the conversation instead of selecting it (Escape or the "Live and drink."/exit choice, per your keybind).
   **Expected:** No popup, no highlight ever appears on the quest giver. Nothing is applied.
   **Evidence:** 📸 `UAT-07-06-cancelled.png` — quest giver's tile unhighlighted after backing out without selecting the choice.

7. **Action (option off):** Open Options → QudUX category, set **"Add conversation option to help locate village quest givers"** to **No**. Start a conversation with any villager in the same zone as an unstarted/active quest giver.
   **Expected:** The "I'm looking for..." (or the "How can I find...?" / "Can you help me track down...?" / "Do you know where...is located?" variant for not-yet-started quests) choice does **not** appear anywhere in that NPC's dialogue at all.
   **Evidence:** 📸 `UAT-07-07-option-off.png` — full dialogue choice list with the option disabled, showing the line is absent. Remember to set the option back to Yes afterward if you want to keep testing.

## Log check

After running the steps above, open `%USERPROFILE%\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log` (see `uat/00-setup.md` for details) and search for:

- `QudUX_ConversationHelper`
- `QudUX_QuestGiverVision`
- `ApplyQuestGiverEffect`
- `AddChoiceToIdentifyQuestGivers`
- `RemoveEffect`
- `Exception`
- `QudUX: (Error)` — this is the exact literal prefix the mod's own catch blocks log to, e.g. `QudUX: (Error) Encountered exception while adding conversation choices to identify village quest givers.`

**FAIL** if any of the following appear in the log at a timestamp matching your test: a stack trace or `CS1503`/`cannot convert` style error, any `QudUX: (Error)` line whose message mentions "identify village quest givers", or any unhandled exception naming `QudUX_ConversationHelper` or `QudUX_QuestGiverVision`.

## Fail indicators (visible in-game)

- The "I'm looking for..." (or "How can I find...?" style) choice never appears even with the option on and a valid quest giver present in the zone.
- Selecting the choice a second time (step 4) throws a visible error, freezes the conversation, or crashes to desktop.
- Two or more flashing markers appear on/near the same quest giver after asking twice (stacked effect instead of refreshed effect).
- The highlight never disappears after talking to the quest giver directly (step 5), or never disappears at all even after leaving it well past a play session.
- The choice still appears after turning the option off (step 7).

## Results

| Step | PASS/FAIL | Evidence file | Notes |
|------|-----------|----------------|-------|
| 1. Choice present | | | |
| 2. Popup on ask | | | |
| 3. Highlight visible | | | |
| 4. Ask again (fixed path) — no duplicate/crash | | | |
| 5. Highlight ends on conversation | | | |
| 6. Cancel — no effect applied | | | |
| 7. Option off — choice absent | | | |

### Problems found

For each problem, copy this block and fill it in:

```
Steps to reproduce:
1.
2.
3.

Expected:

Actual:

Evidence: (file name(s) in qudux-mod-docs/uat/evidence/07/)

Player.log excerpt:
```

## Patch-note correlation (2023–2026)

### 1. Relevant patch entries

**The modding/event-system rewrite (the actual trigger for this whole class of bug):**

> `== 207.31: Spring Molting beta ==` — [Released May 10, 2024.]
> "We added an entire new UI. (ie, we completed work on the modern UI, for folks who've been following our progress)."
> "[modding] MinEvents now support registration with prioritized execution and optional serialization."
> "[modding] Any class can now register for a MinEvent by implementing the IEventHandler interface."
> "[modding] Event handlers now use event registrars to handle both registration and unregistration with a single method implementation."
— `2024.wiki` lines 754–829.

**Cause.** This is the single build that (a) shipped the modern UI in full and (b) replaced the old `IPart.Register(GameObject)` registration model with the new `Register(GameObject, IEventRegistrar)` / event-registrar model. Both halves matter here: the modern UI ships the `ConversationUI` internals QudUX's `Patch_XRL_UI_ConversationUI` transpiler targets (see §2), and the event-registrar change is what the game's `Player.log` is complaining about when it logs (CONFIRMED, from the live `Player.log`):
```
QudUX_ConversationHelper.cs(36,30): warning CS0672: Member 'QudUX_ConversationHelper.Register(GameObject)' overrides obsolete member 'IPart.Register(GameObject)'.
QudUX_ConversationHelper.cs(39,13): warning CS0618: 'IPart.Register(GameObject)' is obsolete: 'Use Register(GameObject, IEventRegistrar)'
```
QudUX never migrated `QudUX_ConversationHelper.Register(GameObject Object)` to the new signature. Decompiling `XRL.World.IPart.ApplyRegistrar` (game 1.0.5, `Assembly-CSharp.dll`) shows the old override is **still invoked** (`Register(Object); Register(Object, EventRegistrar.Get(Object, this));`), so this alone is not fatal — but it is the dated proof that the mod is running on a superseded registration API through a code path the game only keeps for backward compatibility, and it lines up exactly with the timing of the UI rewrite that also broke the mod's own compensating Harmony patch (§2).

**The conversation event split (architecture context, related but not independently fatal):** No patch-note line names `BeforeConversationEvent`/`BeginConversationEvent` directly (this is an internal engine detail, not called out in the notes), but decompiling `XRL.World.Parts.DynamicQuestSignpostConversation.HandleEvent` and `XRL.UI.ConversationUI.InternalConversation` (1.0.5) shows the vanilla "which villager needs your help" intro node is now built during `BeforeConversationEvent.Check(...)`, one step *before* `BeginConversationEvent.Check(...)` (the event QudUX's own Harmony patch listens to). Traced against `ConversationUI.InternalConversation`, `BeforeConversationEvent` still fires strictly before `BeginConversationEvent` in 1.0.5, so this ordering is **not** the break — ⚠️ noting it here only because it shows the conversation pipeline has genuinely been restructured into typed, pooled `MinEvent`s since whichever earlier version the mod's own code comment ("used to ensure our conversation helper can get an event AFTER the game applies its own dynamic quest giver conversation") was written against.

**Unrelated but worth knowing:**
> `== Build 210.24 ==` — [Released October 3, 2025] "[modding] Conversation parts now have a disposal step after the conversation is over and no more events will fire." — `2025.wiki` line 161. Not implicated: QudUX doesn't retain conversation-part references past the triggering event.

**Nothing found for:** "quest giver" / "signpost" as a named feature, "waypoint", "compass", "minimap", or a native map-marker/journal system pointing at village quest givers, in any of 2023.wiki, 2024.wiki, 2025.wiki, or index.wiki (2026, builds 211.33–212.17). Stated plainly: **the patch notes never announce a native replacement for this feature.** The only native "guidance" mechanic is the pre-existing `DynamicQuestSignpostConversation` direction text (see §3), which is not new in this window and is not documented as changing.

### 2. Root cause of the broken feature

**CONFIRMED**, established two ways — by decompiling the current call graph, and by cross-checking it against the other UAT result in this same file (UAT-01):

- `Parts and Effects/QudUX_ConversationHelper.cs:305` `AddChoiceToIdentifyQuestGivers(Conversation, GameObject)` — the method that builds the "I'm looking for X" / "How can I find X?" choice — is called from exactly **one** place: inside `FireEvent(Event E)` (`QudUX_ConversationHelper.cs:42`), gated on `E.ID == "PlayerBeginConversation"` (`QudUX_ConversationHelper.cs:44`, call at line 78).
- `"PlayerBeginConversation"` is **not a real game event**. A full-text search of the decompiled 1.0.5 `Assembly-CSharp.dll` (all ~5,400 source files) turns up the literal string `"PlayerBeginConversation"` in exactly one place: QudUX's own `Harmony Patches/Patch_XRL_World_BeginConversationEvent.cs:18`, a `[HarmonyPostfix]` on `XRL.World.BeginConversationEvent.Check` that manufactures the event by hand:
  ```csharp
  if (GameObject.Validate(ref Actor) && Actor.IsPlayer() && Actor.HasRegisteredEvent("PlayerBeginConversation"))
      Actor.FireEvent(Event.New("PlayerBeginConversation", "Conversation", Conversation, "Speaker", SpeakingWith));
  ```
  The comment on that file explains why: "Used to ensure our conversation helper can get an event AFTER the game applies its own dynamic quest giver conversation (which strips out the conversation and rebuilds it)." This is entirely a mod-internal bridge; the vanilla game never fires `"PlayerBeginConversation"` under any name.
- The bridge depends on the player's body already having `Actor.HasRegisteredEvent("PlayerBeginConversation") == true`. QudUX has (or had) **two** ways to guarantee that: (1) `QudUX_ConversationHelper.AllowStaticRegistration() == true` (`QudUX_ConversationHelper.cs`, override returns `true`), which is supposed to make the engine call `Register(GameObject)` on every object including the player; and (2) a defensive, belt-and-suspenders `Harmony Patches/Patch_XRL_UI_ConversationUI.cs` `[HarmonyPrefix]` that explicitly ran `player.RequirePart<QudUX_ConversationHelper>()` every time a conversation started, forcing the part (and its `"PlayerBeginConversation"` registration) onto the player's body regardless of whatever the static-registration bookkeeping did.
- **`Patch_XRL_UI_ConversationUI.cs` is entirely commented out** in the current mod source (the whole class body, lines 12–93, is wrapped in `//`). It was disabled together with its `[HarmonyTranspiler]`, which patches IL in `ConversationUI.HaveConversation` to draw the speaker's tile in the conversation title bar — a feature that itself likely broke against the 1.0.5 modern-UI `ConversationUI` rewrite (§1) and got commented out wholesale rather than having only the transpiler disabled. That means the `RequirePart<QudUX_ConversationHelper>()` safety net for the player's own registration was disabled as **collateral damage**, not as an intentional fix.
- **Cross-check against the sibling feature (UAT-01, same file):** `AddChoiceToRestockers` is invoked from the *same* dead `"PlayerBeginConversation"` branch (`QudUX_ConversationHelper.cs:65`) **and** from a second, fully independent trigger: `SetTraderInteraction` (`QudUX_ConversationHelper.cs:90`), called from `Harmony Patches/Patch_XRL_UI_TradeUI.cs:20`, a `[HarmonyPrefix]` on `XRL.UI.TradeUI.ShowTradeScreen` — nothing to do with conversation events at all. UAT-01's own recorded result is: *"initially no text… but present after first trade and exiting out of trade window."* That is exactly what you'd see if `"PlayerBeginConversation"` never fires: the restock choice only ever appears via the independent `TradeUI` hook, never on first greeting. `AddChoiceToIdentifyQuestGivers` has **no equivalent second path** — it only exists inside the dead branch — so for it the result isn't "delayed," it's "never."
- Net effect: `AddChoiceToIdentifyQuestGivers` is live, uncorrupted code that is simply never reached in 1.0.5, because the only event that calls it is a mod-manufactured one whose delivery mechanism (the `Patch_XRL_UI_ConversationUI` prefix) was disabled as a side effect of disabling an unrelated, broken transpiler in the same file.

**What is CONFIRMED vs HYPOTHESIS:**
- CONFIRMED: `"PlayerBeginConversation"` does not exist anywhere in vanilla 1.0.5 code; it is 100% a QudUX-manufactured string event (`Patch_XRL_World_BeginConversationEvent.cs:18`).
- CONFIRMED: `AddChoiceToIdentifyQuestGivers` has no caller other than that dead branch (`QudUX_ConversationHelper.cs:44,78`).
- CONFIRMED: `Patch_XRL_UI_ConversationUI.cs`, which used to force-register the player's `QudUX_ConversationHelper` part on every conversation via `RequirePart<>`, is entirely commented out in the current source tree.
- CONFIRMED (behavioral cross-check): UAT-01's own observed result ("present after first trade") is exactly the symptom predicted if `"PlayerBeginConversation"` never fires, corroborating UAT-07's "never appears" for the code path with no alternate trigger.
- HYPOTHESIS ⚠️ unverified: the *exact* failure point was not runtime-traced (no debugger attached to a live game session). It could be (a) `AllowStaticRegistration` alone genuinely does not register the string event on the player's body without the disabled `RequirePart<>` call, or (b) it does register it and the Harmony postfix on `BeginConversationEvent.Check` itself silently fails to bind (no confirming or denying entry appears in `Player.log`'s named "Applying pre-game patches..." list — but that list is a separate, explicitly-logged transpiler-safety mechanism used only for `Scores`, `AbilityManager`, `Look`, `MagneticPulse`, `Physics.HandleEvent`, `GameObject.Move`, and `QudHistoryFactory`; ordinary `[HarmonyPrefix]`/`[HarmonyPostfix]` patches like this one aren't logged either way). Either way the observable result is identical and the fix is the same (see §5).
- Separately noted, **not** the cause of "never appears" but a real latent bug worth fixing at the same time: `AddChoiceToIdentifyQuestGivers` (`QudUX_ConversationHelper.cs:305`) sets `c.Actions = new Dictionary<string,string>{{ "ApplyNewQuestGiverEffect", null }}` in **both** the "already-started quest" branch (which should use the sibling delegate `ApplyActiveQuestGiverEffect`, defined right below it) and the "not-yet-started quest" branch. This only matters once the choice is showing again — it would apply the vision effect to the wrong quest-giver list when the "I'm looking for X" (started-quest) phrasing is used — so it's a correctness bug to fix in the same pass, not the visibility bug the UAT reported.

### 3. Obsolescence verdict: **STILL VALUABLE**

The base game's only native guidance toward village quest givers is the pre-existing `DynamicQuestSignpostConversation` part (`XRL.World.Parts.DynamicQuestSignpostConversation.HandleEvent(BeforeConversationEvent)`, decompiled from 1.0.5 `Assembly-CSharp.dll`): it inserts one intro node per conversation with directional flavor text built from `ParentObject.DescribeDirectionToward(item, General: true)` (e.g., "...also east of you."). This is transient dialogue text only — no journal entry, no map marker, no compass/waypoint, no persistent highlight. A full grep of `2023.wiki`, `2024.wiki`, `2025.wiki`, and `index.wiki` (2026, builds 211.33–212.17) for "journal", "waypoint", "points of interest", "marker", "compass", and "minimap" turns up plenty of unrelated points-of-interest work (ovens, spacetime vortices, sultan statues) but **nothing** that adds native map-marker or journal tracking specifically for village quest givers. **No patch note in this window announces such a feature**, so it is fair to state plainly: nothing has superseded this.

QudUX's value-add over the vanilla text is twofold and both parts are still meaningful if the bug is fixed: (1) it lets a *different*, non-quest-giving villager relay the same directional information (the vanilla text only appears from NPCs the game itself picks to carry the signpost node), and (2) `QudUX_QuestGiverVision` (`Parts and Effects/QudUX_QuestGiverVision.cs`) adds a genuinely new capability the base game has no equivalent for: a temporary flashing highlight (green `*`, or dim gray if unseen/unlit) directly on the target NPC's world tile for 250 turns. Nothing in the vanilla directional-text system does this.

### 4. Modern-UI interaction

This bug is **not** a modern-UI-vs-classic-UI issue, unlike several of the other UAT findings in this file (e.g. UAT-08's callout/modal freeze, UAT-10's missing autoloot modal). `ConversationUI.InternalConversation` dispatches `BeforeConversationEvent.Check` and `BeginConversationEvent.Check` identically regardless of UI mode; only the later rendering step branches on `UIManager.UseNewPopups` (`Render()` vs `RenderClassic()`), by which point the choice list (or its absence) is already fixed. Since `AddChoiceToIdentifyQuestGivers` never runs at all (§2), the choice is missing from the underlying `Node.Elements` list before either renderer sees it — it would be equally absent under old UI. The UAT notes don't record which UI mode was active for UAT-07 specifically, but ⚠️ unverified/moot: based on the code path, the result would not have differed either way. This distinguishes UAT-07 from the genuinely UI-mode-sensitive items elsewhere in this document.

### 5. Recommendation: **fix**

This is a mod regression, not a game-side removal — no native replacement exists (§3) and the underlying choice-injection API (`Node.AddChoice`, `Node.Elements`, `IConversationElement.GetElementByID`) is unchanged and already proven working in 1.0.5 by the restock feature (UAT-01) once it's triggered at all. Concretely:

1. Re-enable (or narrowly re-implement) the `player.RequirePart<QudUX_ConversationHelper>()` call that `Harmony Patches/Patch_XRL_UI_ConversationUI.cs` used to perform on every conversation start, without re-enabling the broken tile-drawing `[HarmonyTranspiler]` in the same file — split that file into two patch classes so a future transpiler failure can't take out an unrelated registration guarantee again. Simplest correct fix: add a one-line `[HarmonyPrefix]` on `XRL.World.BeginConversationEvent.Check` (or reuse the existing `Patch_XRL_World_BeginConversationEvent.cs`) that calls `Actor?.RequirePart<QudUX_ConversationHelper>()` before checking `HasRegisteredEvent`, removing the dependency on `AllowStaticRegistration` alone.
2. While in that file, fix the `"ApplyNewQuestGiverEffect"` vs `"ApplyActiveQuestGiverEffect"` action-key bug noted in §2 (`QudUX_ConversationHelper.cs:305` area) so the already-started-quest branch highlights the correct quest giver list.
3. Consider migrating `QudUX_ConversationHelper.Register(GameObject)` to the non-obsolete `Register(GameObject, IEventRegistrar)` overload while touching this file, since the game has been warning about it since at least build 207.31 (2024) and the obsolete path, while still called by `IPart.ApplyRegistrar` today, has no stated removal date guarantee (unlike the `pBrain`/`pRender`/`pPhysics` obsoletions elsewhere in the mod, which explicitly say "Will not be removed before Q1 2025" — this one carries no such assurance).
4. No UI-mode gating is needed (§4) — this is not a modern-UI-specific fix.

Files/options involved: `Parts and Effects/QudUX_ConversationHelper.cs`, `Parts and Effects/QudUX_QuestGiverVision.cs`, `Harmony Patches/Patch_XRL_UI_ConversationUI.cs`, `Harmony Patches/Patch_XRL_World_BeginConversationEvent.cs`, `Harmony Patches/Patch_XRL_UI_TradeUI.cs` (reference implementation of the working alternate-trigger pattern), option `Options.Conversations.FindQuestGivers` (`Concepts/Options.cs`, unaffected by this bug — it's checked correctly at `QudUX_ConversationHelper.cs`'s `AddChoiceToIdentifyQuestGivers` entry, it just never gets reached).

### 6. Confidence + gaps

**Verified (CONFIRMED):**
- Full-text absence of `"PlayerBeginConversation"` across the entire decompiled 1.0.5 `Assembly-CSharp.dll` except QudUX's own patch.
- `AddChoiceToIdentifyQuestGivers`'s single call site and its gating condition, by direct source read.
- `Patch_XRL_UI_ConversationUI.cs` being fully commented out, by direct source read.
- `IPart.ApplyRegistrar` still invoking the obsolete `Register(GameObject)` overload in 1.0.5 (so the obsolescence itself doesn't independently break anything).
- `BeforeConversationEvent`/`BeginConversationEvent` firing order in `ConversationUI.InternalConversation` (Before, then Begin) — ruling out event-ordering as the cause.
- `DynamicQuestSignpostConversation`'s direction-text-only behavior and its absence from any StreamingAssets XML (it's applied purely from code, via `BeforeConversationEvent`), consistent with the existing UAT-07 doc's note that quest-giver assignment is code-driven (`*_FabricateQuestGiver.BuildZone`).
- Build 207.31 (May 10, 2024) as the version that shipped both the completed modern UI and the `IEventRegistrar`/`MinEvent` modding overhaul, via direct patch-note text.
- `GameObject.HasProperty`/`SetStringProperty`/`GetObjectsWithProperty` all reading/writing the same underlying `Property` dictionary in 1.0.5 — ruled out as a red herring (the mod's `GetObjectsWithProperty` call and the vanilla part's `HasProperty` predicate are consistent, not a source of the bug).

**Gaps:**
- No live game session was run; nothing here was confirmed by attaching a debugger or adding temporary logging to a running instance. The `Player.log` on disk shows QudUX loaded and its named "pre-game patches" list (a separate, explicitly-logged transpiler-safety mechanism) applying with 4 known failures unrelated to conversations (`Scores`, `AbilityManager`, `Look`, `MagneticPulse`), but it does not log success/failure for ordinary attribute-based Harmony patches like `Patch_XRL_World_BeginConversationEvent` or the now-disabled `Patch_XRL_UI_ConversationUI`, so it neither confirms nor refutes the exact failure point beyond what static analysis shows.
- Whether `AllowStaticRegistration` alone (without the disabled `RequirePart<>` call) is sufficient to register the player's body for `"PlayerBeginConversation"` was not traced through the engine's static-part-registration bootstrap code (a large, generically-named subsystem in `GameObject.cs`/`GameObjectFactory`); marked ⚠️ unverified. It doesn't change the recommended fix either way.
- Did not check whether any *other* QudUX feature besides the restock choice also silently depends on the same dead `"PlayerBeginConversation"` bridge — out of scope for this UAT topic, but worth a follow-up sweep given the mechanism is shared.
