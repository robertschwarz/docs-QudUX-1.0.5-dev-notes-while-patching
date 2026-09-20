# UAT 01: Conversation helper loads

> Read `uat/00-setup.md` first for install, save discipline, wish prompt, and log location.

## What was fixed

A leftover, unused `using` line in `QudUX_ConversationHelper.cs` referenced a namespace that no longer exists in game version 1.0.5, which broke the mod build. The line was deleted; it wasn't used for anything, so there's no behavior change. This script proves the conversation part still loads and its conversation options still work on 1.0.5.

## Feature under test

`QudUX_ConversationHelper` (`Parts and Effects/QudUX_ConversationHelper.cs`) is a globally-registered part (`AllowStaticRegistration() = true`) that listens for the `PlayerBeginConversation` event on every NPC and, when conditions are met, injects extra dialogue choices into the in-game **Conversation** screen (the full-screen text dialogue that opens when you talk to an NPC).

It adds two independent features, gated by two separate QudUX options:
- **Ask about restock** (`AddChoiceToRestockers`) — adds an "ask about restock" choice to a merchant's `Start` node, but only after you've viewed that merchant's trade screen at least once in the current conversation session (`ZoneTradersTradedWith`).
- **Identify quest givers** (`AddChoiceToIdentifyQuestGivers`) — adds a choice to ask an NPC where nearby dynamic-quest givers are located.

This script only smoke-tests the **restock** path, since it's the simplest to trigger reliably and proves the part fires, adds a choice, and the choice renders correctly. The quest-giver path is deep-tested in UAT-07 (quest-giver vision) and merchant restock timing/dialog accuracy in UAT-05 — don't duplicate those here.

## Preconditions

- QudUX option (Options menu → **QudUX** category): **"Allow asking merchants when they will restock"** = **Yes** (this is the default).
- No other QudUX or game options are relevant to this test.
- Save state: any fresh save works. No quest state, reputation, or location requirements — this test spawns its own NPC via wish so it's independent of where you are.

## Setup

1. Load `qudux-32-test-uat-character`.
2. Open the wish prompt (`Ctrl+W`) and enter:
   ```
   HumanTinker1
   ```
   This spawns a generic human tinker (blueprint confirmed in `ObjectBlueprints/Creatures.xml`) adjacent to you. It has `GenericInventoryRestocker` and uses the `tinker` conversation (`Conversations.xml`), which has a single, unconditional `Start` node — no quest gates, so it won't give confusing/blocked dialogue on a fresh character.

## Test steps

1. **Action:** Open Options (`Esc` → Options → **QudUX** category). Confirm **"Allow asking merchants when they will restock"** is set to **Yes**.
   **Expected:** Checkbox reads "Yes".
   **Evidence:** 📸 `UAT-01-01-options-on.png` — the QudUX options category with this checkbox visible and set to Yes.

2. **Action:** Face the spawned tinker and press `c` (Talk).
   **Expected:** The Conversation screen opens with the `tinker` flavor text (one of several random lines, e.g. "Use your gadgets hard, don't you? I can fix them.") and exactly **one** choice: "Live and drink." No QudUX restock choice yet — you haven't viewed the trader's goods this session.
   **Evidence:** 📸 `UAT-01-02-baseline-no-choice.png` — full conversation screen showing only "Live and drink." as a choice.

3. **Action:** While still in the conversation, press `Tab` (or `T`) — bound to **"Start trade from conversation"** (`CmdStartTrade` in `Commands.xml`).
   **Expected:** The Trade screen opens, showing the tinker's inventory (Pocketed Vest, Sandals, Telescopic Monocle, Steel Utility Knife, Wrench, Basic Toolkit, etc.).
   **Evidence:** 📸 `UAT-01-03-trade-screen.png` — trade screen with the tinker's inventory visible.

4. **Action:** Press `Esc` to leave the trade screen. If this doesn't return you to the conversation screen automatically, press `c` on the tinker again to re-open the conversation.
   **Expected:** The conversation's `Start` node now shows an **additional** choice — exactly one of these three (randomly chosen each time the node regenerates):
   - "Any new wares on the way?"
   - "Do you have anything else to sell?"
   - "Can you let me know if you get any new stock?"

   ...alongside "Live and drink.". This proves `QudUX_ConversationHelper.SetTraderInteraction` fired from the Harmony trade-screen patch and `AddChoiceToRestockers` added the choice.
   **Evidence:** 📸 `UAT-01-04-choice-appears.png` — conversation screen with both the new restock choice and "Live and drink." visible together.

5. **Action:** Select the new restock choice (e.g. "Any new wares on the way?").
   **Expected:** A new dialogue node appears with narrative restock text (exact wording varies — it'll be a merchant-flavor line about incoming stock, e.g. mentioning a dromad caravan, a water baron, or an apprentice's contacts), followed by exactly two choices: **"I have more to ask"** and **"Live and drink."**
   **Evidence:** 📸 `UAT-01-05-restock-node.png` — the restock dialogue node with its text and both choices visible.

6. **Action:** Select "I have more to ask".
   **Expected:** Returns to the `Start` node; the restock choice is still present (no duplicate copies, no crash).
   **Evidence:** 📸 `UAT-01-06-loop-back.png` — Start node showing the restock choice exactly once.

7. **Action:** Select "Live and drink." to end the conversation.
   **Expected:** Conversation closes cleanly, back to the game map.
   **Evidence:** none required.

## Edge case: option OFF → choice absent

8. **Action:** Open Options → QudUX and set **"Allow asking merchants when they will restock"** to **No**. Wish-spawn a second tinker (`Ctrl+W` → `HumanTinker1`), talk to it (`c`), start trade from the conversation (`Tab`), then `Esc` back to the conversation.
   **Expected:** No restock choice appears, even though the trade screen was viewed — only "Live and drink." Confirms the option gate (`Options.Conversations.AskAboutRestock`) works.
   **Evidence:** 📸 `UAT-01-08-option-off-no-choice.png` — conversation screen after trading, showing only "Live and drink.".
9. **Action:** Set the option back to **Yes** before moving to the next script (per `00-setup.md` save discipline).
   **Expected:** Restored to default.
   **Evidence:** none required.

## Log check

After finishing, search `Player.log` (see `00-setup.md` for path/command) for:
- `Exception` anywhere near `QudUX_ConversationHelper`, `AddChoiceToRestockers`, `AddChoiceToIdentifyQuestGivers`, or `SetTraderInteraction` — any hit is a **FAIL**.
- `PlayerBeginConversation` combined with an error/stack trace — **FAIL**.
- Any `MissingMethodException`, `TypeLoadException`, or `CS0234`/`EncounterObjectBuilders` reference — would indicate the original build error resurfaced — **FAIL**.
- A clean log with no `QudUX`-tagged exceptions is a **PASS** on this axis.

## Fail indicators

- Talking to the tinker throws an exception or freezes the game.
- The restock choice never appears even after viewing the trade screen (step 4).
- The restock choice appears on the very first conversation, before any trade screen was opened (step 2) — means the `ZoneTradersTradedWith` gate isn't working.
- The restock node's two choices ("I have more to ask" / "Live and drink.") are missing, duplicated, or lead somewhere unexpected.
- Turning the option to "No" doesn't suppress the choice (step 8).
- Any `QudUX`-tagged exception in `Player.log`.

## Results

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
| 1 — Options on | | | |
| 2 — Baseline, no choice | | | |
| 3 — Trade screen opens | | | |
| 4 — Choice appears | | | |
| 5 — Restock node | | | |
| 6 — Loop back | | | |
| 7 — End conversation | | | |
| 8 — Option off, no choice | | | |
| 9 — Option restored | | | |
| Log check | | | |

### Problems found

For each problem, copy this block:

```
**Steps to reproduce:**


**Expected:**


**Actual:**


**Evidence:** (screenshot/video filename)


**Player.log excerpt:**
```

## Patch-note correlation (2023–2026)

### 1. Relevant patch entries

None of the entries below describe a native "ask about restock" conversation choice being added — I found **no patch note in 2023–2026 that gives merchants a built-in dialogue option to ask when they'll restock**. Everything relevant is either about the underlying restock mechanics the mod reads (`GenericInventoryRestocker`), or about the trade/conversation UI plumbing the mod hooks into.

**Directly related (mechanism the mod depends on), not causal to the observed behavior:**

- Build 206.48, Dec 8 2023 (`2023.wiki:136`): `[modding] There is now an intproperty TradeCount that is incremented on objects each time the player enters the trade screen with them.` — same trade-screen-visit tracking concept the mod reimplements itself via `ZoneTradersTradedWith`/`SetTraderInteraction`. Related, not causal — the mod does not read this native property, it keeps its own list.
- Build 206.48, Dec 8 2023 (`2023.wiki:138`): `[modding] Builder-based merchant inventory has been converted to population tables using GenericInventoryRestocker.` — this is the part `AddChoiceToRestockers` reads (`speaker.GetPart<GenericInventoryRestocker>()`, `Parts and Effects/QudUX_ConversationHelper.cs:165`). Confirms when this part became the standard restocker.
- Build 206.61, Jan 26 2024 (`2024.wiki:1107`): `[modding] GenericInventoryRestocker now supports combining multiple population tables of stock by providing a comma separated list.` — same part, unrelated to the dialogue trigger.
- Build 206.63, Feb 2 2024 (`2024.wiki:1100-1102`): three restock-reliability bugfixes (`Fixed a bug that caused dynamic village merchants to restock twice.` etc.) — restock-timing bugfixes, not conversation-related.
- 1.0.4 (build 210.10), May 22 2025 (`2025.wiki:90`): `Restocking merchants should more reliably dispose of unimportant items sold to them.` — restock behavior, unrelated to the ask-about-restock choice.
- 1.0.4 (build 210.10), May 22 2025 (`2025.wiki:161`): `[modding] Conversation parts now have a disposal step after the conversation is over and no more events will fire.` — general conversation-lifecycle change (adds a teardown step); worth knowing for modders but does not explain the observed "choice appears only after leaving/reentering the conversation" behavior — that behavior is fully explained by the mod's own code (see below), not a patch change.
- Build 207.69, Jun 14 2024 (`2024.wiki:598,601`): `The new trade UI now stays open with items selected when cancelling the "Offer" dialog, making the behavior between the console and new UIs the same.` / `Greatly improved the performance of the trade screen.` — confirms a "new" (modern) trade UI existed as a distinct codepath from "console" as of mid-2024. Relevant background for the modern-UI question below, not causal.
- Build 211.36 (beta), Mar 1 2026 (`index.wiki:40`): `Merchants now allow you to trade with them you after you buy their entire inventory.` — related to trading generally, not to the restock-inquiry choice.

**Unrelated but worth knowing:**
- 1.0.4 (build 210.10), May 22 2025 (`2025.wiki:104`): `Fixed a bug that caused some epilogue conversations to be inaccessible when marooned.` — different, scripted conversations, not merchant dialogue.
- Build 212.17 (experimental), Mar 27 2026 (`index.wiki:21`): `Companion encumbrance is now visible on the Trade screen.` — Trade screen cosmetic addition, unrelated to conversation choices.

**Conclusion on cause of observed behavior:** the "absent at first, appears after first trade and exiting trade window" behavior is not explained by any patch note — it is the mod's own intentional gate, confirmed in source: `Parts and Effects/QudUX_ConversationHelper.cs:130-134` explicitly requires `ZoneTradersTradedWith.Contains(speaker)` before adding the choice, and that list is only populated by `SetTraderInteraction`, which is invoked from `Harmony Patches/Patch_XRL_UI_TradeUI.cs:16-21` — a Harmony `Prefix` on `XRL.UI.TradeUI.ShowTradeScreen`. This is UAT PASS behavior working exactly as designed, not a regression or a patch-introduced change.

### 2. Obsolescence verdict: **STILL VALUABLE**

- No patch note in the 2023–2026 range adds a native "ask merchant when they'll restock" dialogue option. The base game's own restock-related changes are all about the restocker's internal reliability/timing (see above), never about surfacing that info to the player through conversation.
- Decompiled `XRL.World.Parts.GenericInventoryRestocker` (via ilspycmd on the current `Assembly-CSharp.dll`) still exposes `LastRestockTick` and `RestockFrequency` exactly as the mod's field access expects (`Parts and Effects/QudUX_ConversationHelper.cs:165-166`), confirming the mod's read of restock timing is still valid on 1.0.5.
- The mod adds real value on top: narrative flavor text per-trader (including a bespoke Sparafucile branch), a stable per-trader dialogue variant via `TraderDialogGenData` (so the same trader doesn't reroll their line every conversation), and a natural-language "days till restock" phrase — none of which the base game provides.

### 3. Modern-UI interaction

Decompiled evidence shows this feature is **not** modern-UI-gated, unlike the other UAT findings (TileMaker, QuickPickup, recipe menu):

- `XRL.UI.TradeUI.ShowTradeScreen(GameObject Trader, ...)` — the exact method QudUX Harmony-patches — is a single shared static entry point. Decompiling it shows an internal branch: `if (Options.ModernUI) { OfferStatus result = TradeScreen.show(Trader, costMultiple, screenMode).Result; ... return; } ... GameManager.Instance.PushGameView("ConsoleTrade");`. Because QudUX's hook is a Harmony **Prefix** (`Harmony Patches/Patch_XRL_UI_TradeUI.cs:16-21`), it runs *before* that branch is reached — so `SetTraderInteraction(Trader)` fires identically whether the player has modern UI on or off.
- Decompiling `XRL.UI.ConversationUI` (the class that renders the conversation screen and choices) turned up **no** `Options.ModernUI` branch anywhere in the type — choice rendering goes through a single `Popup.ShowConversation(...)` call (line ~521 of the decompiled type) regardless of UI mode. This means the conversation screen itself was not split into separate classic/modern implementations the way Trade and Inventory were.
- Net effect: this feature's trigger (Harmony prefix on `ShowTradeScreen`) and its rendering (`ConversationUI`/`Popup.ShowConversation`) are both UI-mode-agnostic in the current 1.0.5 assembly. I could not test this in-game (that's the human UAT's job), but the code path gives no reason to expect a modern-UI-specific failure, unlike the file's sibling gap already noted in UAT-07 (quest-giver locator, out of scope here).

### 4. Recommendation

**Keep as is.** Do not gate this file behind the planned UI-mode feature flag — the decompiled evidence shows both the trigger point (`TradeUI.ShowTradeScreen`) and the render point (`ConversationUI`) are shared, UI-mode-agnostic code paths in 1.0.5, unlike the modern-UI casualties in UAT-02/03/04. No native API replaces the "ask about restock" dialogue, so there's nothing to simplify onto. Files involved if this ever needs revisiting: `Parts and Effects/QudUX_ConversationHelper.cs` (feature logic), `Harmony Patches/Patch_XRL_UI_TradeUI.cs` (trigger), `Concepts/Options.cs` (`Options.Conversations.AskAboutRestock` gate).

### 5. Confidence + gaps

**Verified:**
- All quoted patch-note lines pulled verbatim from the local `.wiki` files with build/date resolved from the nearest preceding `==`-level header.
- `GenericInventoryRestocker.LastRestockTick`/`RestockFrequency` fields exist in the current `Assembly-CSharp.dll` via ilspycmd decompile.
- `TradeUI.ShowTradeScreen` signature (`GameObject Trader, float _costMultiple = 1f, TradeScreenMode screenMode = TradeScreenMode.Trade`) matches the Harmony patch target, and its body's `Options.ModernUI` branch was read directly from decompiled IL/C#.
- `ConversationUI` type dump was grepped for any `ModernUI` branch; none found.

**Gaps:**
- I did not run the game myself; the "works" / "absent then appears" observation is the user's own UAT result, not something I reproduced. ⚠️ unverified: whether `Popup.ShowConversation` itself has zero modern-UI-specific rendering quirks beyond the absence of a branch (a lack of an `Options.ModernUI` check is strong but not 100% proof of pixel-identical behavior in both modes).
- Did not check Steam/itch.io devlogs beyond the local wiki mirrors (no web fetch was needed — the local `.wiki` files fully covered the keyword search).
- Did not cross-reference UAT-05 (restock countdown accuracy) or UAT-07 (quest-giver locator) patch history, since those are explicitly out of scope per this UAT's own "don't duplicate those here" note.

