# UAT 05: Merchant restock timer

> Read `uat/00-setup.md` first for install, save discipline, wish prompt, key reference, and log location.
> Read `uat/01-conversation-helper-loads.md` first too — it already smoke-tests that the "ask about restock" choice appears/disappears correctly (option on/off, trade-required gate, choice text variants, basic node structure) using a wish-spawned generic tinker. **This script does not repeat any of that.** It goes deeper on one thing: whether the *time-until-restock* the dialog reports is actually correct — does it read sensibly, does it count down as turns pass, and does it reset when a restock actually happens — using a real village merchant instead of a spawned one.

## What was fixed

The code that computes "how long until this merchant restocks" referenced a game type (`Restocker`) that no longer exists in game version 1.0.5, which broke the mod build entirely. The fix removed the dead branch and kept the working one (`GenericInventoryRestocker`), which every merchant in the game actually uses. No behavior change was intended — this script proves the timer math still produces sensible, decreasing values on 1.0.5.

## Feature under test

`QudUX_ConversationHelper.AddChoiceToRestockers` (`Parts and Effects/QudUX_ConversationHelper.cs`, ~lines 96–303) computes:
```
ticksRemaining = r.RestockFrequency - (XRLCore.CurrentTurn - r.LastRestockTick)
daysTillRestock = ticksRemaining / Calendar.turnsPerDay   // turnsPerDay = 1200 (confirmed: Assembly-CSharp XRL.World.Calendar.TurnsPerDay = 1200)
```
and turns that into flavor-text dialog in the **Conversation** screen, reached via the "ask about restock" choice (e.g. "Any new wares on the way?"). The reply text bucket by `daysTillRestock`:
- `>= 9 days`: "business is booming" / "not rotating stock" flavor — no time estimate given.
- `< 9 days`: one of 4 random narrative themes (dromad caravan / water baron / apprentice / arconauts) with a `daysTillRestockPhrase` ("in a matter of hours" / "by this time tomorrow" / "within a day or two" / "in about two days' time" / "in about three days' time" / "in four or five days" / "in about a week, give or take"), **plus** an uncertainty caveat line ("I can't make any guarantees" or equivalent) — this caveat is now always appended in 1.0.5, since the only surviving code path is the chance-based (`GenericInventoryRestocker`) branch.

## Merchant used for this test

**Nima Ruda**, Joppa's apothecary. Confirmed in `ObjectBlueprints/Creatures.xml`:
```xml
<object Name="Nima Ruda" Inherits="NPC">
  ...
  <part Name="GenericInventoryRestocker" Table="Village Apothecary 0" Chance="71" />
  ...
</object>
```
No `RestockFrequency` override on her blueprint, so she uses the class default confirmed by decompile: `RestockFrequency = 6000L` ticks = **5.0 days** exactly (6000 / 1200). Her `ConversationScript` is `Nima Ruda` (confirmed in `Conversations.xml`), with `Start` node choices "What wares do you offer?" / "Where are you from?" / "Live and drink." — no quest gates, so it's safe to talk to her on a fresh character.

She's in Joppa, the starting village — no travel needed for `qudux-32-test-uat-character`.

## Preconditions

- QudUX option (Options menu → **QudUX** category): **"Allow asking merchants when they will restock"** = **Yes** (default; per `uat/01`, already verified to gate this feature — just confirm it wasn't left off from a previous script).
- No relevant game options.
- Save state: `qudux-32-test-uat-character`, standing in Joppa.

## Setup

1. Load `qudux-32-test-uat-character` (per `00-setup.md`).
2. Locate Nima Ruda in Joppa (female NPC, apothecary — she's usually near the huts with the other named Joppa villagers like Argyve and Elder Bob/Irudad).

## Test steps

1. **Action:** Face Nima Ruda and press `c` (Talk). Select "What wares do you offer?" then start trade with `Tab` (or `T` — `CmdStartTrade`, confirmed in `Commands.xml`).
   **Expected:** Trade screen opens showing her stock (nostrums/herbalism goods per "Village Apothecary 0" table).
   **Evidence:** 📸 `UAT-05-01-trade-screen-baseline.png` — trade screen with her inventory visible (for later comparison in step 6).

2. **Action:** Press `Esc` to leave the trade screen, back to the conversation. Select the new restock choice (e.g. "Any new wares on the way?").
   **Expected:** A dialog line appears mentioning a wait of roughly **4–5 days, or "about a week"** (since `RestockFrequency` = 5.0 days and the character has barely aged), naming one of the 4 narrative themes (caravan / water baron / apprentice / arconauts), **and** an uncertainty caveat sentence ("...I can't make any guarantees" or similar, appended because the surviving code path is always chance-based in 1.0.5).
   **Evidence:** 📸 `UAT-05-02-initial-timer.png` — full dialog text visible, legible enough to read the day estimate and the caveat sentence.

3. **Action:** Select "Live and drink." to end the conversation.
   **Expected:** Returns to the map.
   **Evidence:** none required.

4. **Action:** Open the wait menu (`Ctrl+`` ` `` — `CmdWaitMenu`, confirmed in `Commands.xml`) and choose **"Wait a number of turns"** (`CmdWaitN`). Enter **2500** turns and confirm. ⚠️ unverified — confirm in game: the exact prompt text/UI for entering the turn count (the command's existence and that it advances the game clock by the given turn count is confirmed in `Commands.xml`; the dialog wording that asks for the number is not).
   **Expected:** The game advances roughly 2 in-game days (2500 turns ÷ 1200 turns/day ≈ 2.08 days); no crash, no popup errors. If a wandering creature interrupts the wait early, repeat "Wait a number of turns" until total elapsed is at least ~2000 turns (check the in-game date/clock).
   **Evidence:** 📸 `UAT-05-04-after-wait.png` — the in-game date/clock display (or journal) showing time has advanced roughly 2 days versus step 1–2.

5. **Action:** Talk to Nima Ruda again (`c`) — **no need to re-open trade**, since `ZoneTradersTradedWith` persists for the whole Joppa visit (only cleared on a zone change) — and select the restock choice again.
   **Expected:** Same narrative theme as step 2 (waiting period hasn't reset), but the day-estimate phrase is now **shorter** than step 2's — e.g. if step 2 said "in four or five days," this should now say "in about three days' time" or shorter, since ~2 days' worth of ticks were consumed. The reading must be strictly less than step 2's, not equal or longer.
   **Evidence:** 📸 `UAT-05-05-decreased-timer.png` — dialog text showing the new, shorter estimate. Caption/note which phrase appeared in step 2 vs. here for the results table.

6. **Action:** End the conversation. Open the wish prompt (`Ctrl+W`) and enter:
   ```
   restock
   ```
   This is a built-in game cheat (confirmed by decompiling `XRL.World.Capabilities.Wishing`: `Wish == "restock"` calls `GenericInventoryRestocker.PerformRestock()` and sets `LastRestockTick = The.Game.TimeTicks` on every restocker in the current zone) — it force-triggers an actual restock on Nima Ruda (and any other Joppa restockers) right now, so you don't have to wait out the remaining ~3 days for real.
   **Expected:** No error popup; message log may show item-related text as her stock regenerates.
   **Evidence:** 📸 `UAT-05-06-wish-restock.png` — wish prompt/confirmation, or the message log right after.

7. **Action:** Talk to Nima Ruda, start trade (`Tab`) again.
   **Expected:** Trade screen opens; compare to step 1's screenshot — inventory should be freshly rolled from "Village Apothecary 0" (contents may differ, quantities may differ, or it may simply be a clean rebuild — the key proof is no crash and the trade screen still populates correctly after a forced restock).
   **Evidence:** 📸 `UAT-05-07-trade-screen-after-restock.png` — trade screen post-restock, for side-by-side comparison with step 1.

8. **Action:** Leave trade (`Esc`), ask about restock one more time.
   **Expected:** The timer has reset — the day estimate should be back to roughly the **step 2** ballpark (4–5 days / "about a week"), not the shorter step-5 reading. This proves `LastRestockTick` was actually updated by the forced restock, not just the display text. The narrative theme may have changed (a new waiting period can re-roll `TraderDialogGenData`) — that's expected, not a bug.
   **Evidence:** 📸 `UAT-05-08-timer-reset.png` — dialog showing the reset, full-length estimate.

## Edge/negative cases

- Option gating (on/off) and the "must trade before the choice appears" gate are already covered by `uat/01` — not repeated here. If you toggled the option off/on while working through this script, restore it to **Yes** before moving to the next script (per `00-setup.md`).
- **Non-merchant NPC:** talk to Elder Bob (displayed name **Irudad**, blueprint `ElderBob`) — confirmed in `Creatures.xml` to have no `GenericInventoryRestocker` part and no `Merchant` tag. Even after trying to interact with him, no restock-flavored choice should ever appear (he has no trade option at all, so this should hold trivially).
  **Evidence:** 📸 `UAT-05-E1-non-merchant-no-choice.png` — his conversation screen, showing no wares/restock-related choice.

## Log check

After finishing, search `Player.log` (path/command in `00-setup.md`) for:
- `Encountered exception while adding conversation choice to merchant to ask about restock duration` — **FAIL** (from the `try/catch` around `AddChoiceToRestockers` in the `PlayerBeginConversation` handler).
- `Encountered exception in AddChoiceToRestockers` — **FAIL** (from the `try/catch` inside the method itself; note which `debugSegment` number it reports, useful for triage).
- Any `NullReferenceException` or `Exception` near `QudUX_ConversationHelper`, `GenericInventoryRestocker`, or `TraderDialogGenData` — **FAIL**.
- A clean log with no `QudUX`-tagged exceptions after all 8 steps is a **PASS** on this axis.

## Fail indicators

- The restock dialog throws, freezes, or shows blank/garbled text instead of a day estimate.
- Step 2's initial estimate is nonsensical (e.g., negative, "9+ days" when `RestockFrequency` is only 5 days, or missing the uncertainty caveat entirely).
- Step 5's estimate is **not** shorter than step 2's (timer isn't actually counting down).
- Step 8's estimate is **not** back near step 2's ballpark after a forced restock (timer didn't reset — `LastRestockTick` not being updated).
- The trade screen fails to open or crashes in step 7 (post-restock inventory rebuild broken).
- The non-merchant NPC (Elder Bob) somehow shows a restock-related choice.
- Any `QudUX`-tagged exception in `Player.log`.

## Results

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
| 1 — Trade screen baseline | | | |
| 2 — Initial timer reading | | | |
| 4 — Time advanced (~2 days) | | | |
| 5 — Timer decreased | | | |
| 6 — Wish `restock` | | | |
| 7 — Trade screen after restock | | | |
| 8 — Timer reset | | | |
| E1 — Non-merchant no choice | | | |
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

Sources: `2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026, builds 211.33–212.17) grepped for `restock`, `Restocker`, `merchant`, `trade`, `vendor`, `shop`. Cross-checked against decompiled `Assembly-CSharp.dll` (ilspycmd) and `ObjectBlueprints/Creatures.xml`.

### 1. Relevant patch entries

**Causal — explains why the mod's `Restocker` branch broke in 1.0.5:**

> `== 206.50 ==` [Released December 22, 2023] — "* [modding] Builder-based merchant inventory has been converted to population tables using GenericInventoryRestocker."
(`2023.wiki` line 138)

This is the earliest evidence of the migration away from the old builder-based restocker toward `GenericInventoryRestocker`. It doesn't say `Restocker` (the class) was deleted on this date — only that new merchant inventory started being defined via population tables + `GenericInventoryRestocker`. Decompile confirms the end state: `XRL.World.Parts.Restocker` **does not exist** in the current 1.0.5 `Assembly-CSharp.dll` (ilspycmd: `Could not find type definition XRL.World.Parts.Restocker in type system`), while `XRL.World.Parts.GenericInventoryRestocker` exists and is the type actually used by every blueprint checked in `Creatures.xml` (20+ hits, e.g. `<part Name="GenericInventoryRestocker" Table="Tier1Wares" />`). Zero blueprints reference a bare `Restocker` part anywhere in `StreamingAssets/Base`. This matches `git diff` on the mod source: the fix removed the `speaker.HasPart("Restocker")` / `GetPart<Restocker>()` branch and the `using XRL.World.Encounters.EncounterObjectBuilders;` import, keeping only the `GenericInventoryRestocker` branch. **I could not find an exact patch note pinpointing the build where the `Restocker` class was physically deleted from the assembly** — the 206.50 entry is the start of the migration, not a confirmed removal date. ⚠️ unverified: exact build number of final `Restocker` class removal.

**Related but not causal:**

> `== 206.37 ==` (2024.wiki, build header preceding line 1100) — "* Fixed a bug that caused dynamic village merchants to restock twice." / "* Fixed a bug that caused mid and high-tier dynamic village merchants to have less stock than intended." / "* Fixed a bug that caused dynamic village merchants to sometimes spawn with the wrong items in stock." / "* [modding] GenericInventoryRestocker now supports combining multiple population tables of stock by providing a comma separated list." / "* [debug] Added a 'restock' wish to force all restocking merchants on the map to refresh their inventories."

These are `GenericInventoryRestocker` maturity fixes (2024), not related to the UAT's 4–5-day / 1–2-day observation, but confirm `GenericInventoryRestocker` was the actively developed system going forward. The `'restock'` wish this entry introduces is the same one this UAT script uses in step 6 (confirmed still present via decompiled `XRL.World.Capabilities.Wishing`).

> `2023.wiki` line 126 — "* Fixed a bug that caused some merchants to fail to restock until their zone had been visited multiple times." — related (general restock reliability), not causal to the timer-math bug.

> `2024.wiki` line 193 / `2025.wiki` line 90 — "Restocking merchants now more reliably trade away unimportant items sold to them." — unrelated to restock *timing*; affects what merchants do with items you sell them.

**Unrelated but worth knowing (per task's explicit flag):**

> `Build 211.36 (beta)` [Released March 1, 2026] — "* Merchants now allow you to trade with them you after you buy their entire inventory."
(`index.wiki` line 40)

Checked the full 211.36 entry and its neighbors (211.35, 212.17) line by line — this is about the trade screen no longer locking you out when a merchant's inventory hits zero items; it does not touch `RestockFrequency`, `LastRestockTick`, or any restock cadence. No sibling entry in 211.33–212.17 changes restock mechanics or the `Restocker`/`GenericInventoryRestocker` parts. Confirmed unrelated.

**Restocker → GenericInventoryRestocker removal timeline, summarized:** migration started (builder-based → population tables + `GenericInventoryRestocker`) December 22, 2023 (206.50); `GenericInventoryRestocker` feature-matured through January 2024 (206.37); by the current 1.0.5 build the old `Restocker` class is fully gone from the assembly and absent from all shipped blueprints. The precise build that deleted the class is not identifiable from the available wiki pages (no "removed Restocker part" line found in any of 2023/2024/2025/index.wiki).

### 2. Obsolescence verdict: **STILL VALUABLE**

Decompiled `GenericInventoryRestocker` (full type dump) has no player-facing description/look-text hook — its only informational event handler is `HandleEvent(GetDebugInternalsEvent)`, which is debug-only (surfaces `LastRestockTick`/`RestockFrequency`/`Chance`/`Tables` to modders/debuggers, not to players). There is no `GetShortDescription`, `GetInventoryActionsEvent`, or trade-screen hook that prints restock timing. I found no wiki entry across 2023–2026 adding a native restock-countdown display anywhere (trade screen, look text, or conversation). The base game still has no player-facing way to learn how long until a merchant restocks — you either wish-cheat it, dig into the wiki's known frequencies, or just come back later. QudUX's conversation choice is the only surfaced source of this information for a normal player, so the feature is not made redundant by any native API found.

### 3. Modern-UI interaction: UI-independent (verified)

The feature is implemented as `Conversation`/`Node`/`Choice` objects added to the game's existing dialog-tree data structure (`AddChoiceToRestockers`, `Parts and Effects/QudUX_ConversationHelper.cs`), triggered off the `PlayerBeginConversation` part event — this is the same conversation engine used for all NPC dialog, not a bespoke render layer. Grepped all four patch-note files for `modern UI` / `new UI` / `UI overhaul` / `legacy UI`: found extensive modern-UI rework entries for the **trade UI**, character sheet, ability bar, equipment screen, and message log (e.g. `2024.wiki` line 598: "The new trade UI now stays open with items selected when cancelling the 'Offer' dialog...") but **zero entries describing a modern-UI rework of the conversation/dialog-tree screen itself**. This is consistent with the UAT author's own notes: UAT 05 is one of only two topics (with UAT 1) that carries no modern-UI caveat, unlike UAT 2/4/6/8 which explicitly flag modern-UI-only breakage or native-menu obsolescence. Verdict: this feature is UI-mode-independent — it should work identically whether the player has modern UI on or off, because it only touches conversation data, not screen rendering. No `⚠️ unverified` tag needed here since it's grounded in both the source code path and the absence of any conflicting patch note.

### 4. Recommendation: **keep as is**

- Files/options involved: `Parts and Effects/QudUX_ConversationHelper.cs` (`AddChoiceToRestockers`, already fixed to drop the dead `Restocker` branch), `Options.xml` (`QudUX_OptionAskAboutRestock`, DisplayText "Allow asking merchants when they will restock", default Yes), `Concepts/Options.cs` (`Options.Conversations.AskAboutRestock`).
- No UI-mode gating needed — this is not part of the legacy/modern feature-flag split the user is planning for other QudUX features (TileMaker, QuickPickup, recipe menu, scoreboard). It's conversation data, engine-agnostic.
- No native API to lean on for simplification — `GenericInventoryRestocker` exposes the needed fields (`RestockFrequency`, `LastRestockTick`) but has no built-in text/UI presentation to delegate to; the mod's flavor-text generation is additive, not a duplicate of something the base game already renders.
- The already-completed fix (drop `Restocker`, keep `GenericInventoryRestocker`) is correct and sufficient — matches what shipped blueprints actually use.

### 5. Confidence + gaps

**Verified directly:**
- `XRL.World.Parts.Restocker` absent from 1.0.5 `Assembly-CSharp.dll` (ilspycmd decompile attempt fails with "could not find type definition").
- `GenericInventoryRestocker` full source decompiled; confirmed fields (`RestockFrequency` default 6000L = 5 days), confirmed no player-facing text output, confirmed `StartTradeEvent`/`TurnTick` restock logic.
- Zero bare `Restocker` part references in `StreamingAssets/Base` (`grep -rn 'Name="Restocker"'` empty); 20+ `GenericInventoryRestocker` references.
- Mod's own `git diff` on `QudUX_ConversationHelper.cs` shows exactly the branch removal described in the task.
- All four patch-note files fully grepped for the given keyword list plus modern/legacy UI terms.
- Build 211.36 entry and full surrounding 2026 entries read in full — confirmed unrelated to restock cadence.

**Gaps / unverified:**
- ⚠️ unverified: exact build/date the `Restocker` class was physically removed from the compiled assembly (only the 206.50 migration-start note was found; no explicit "removed Restocker" line exists in the wiki text available locally).
- ⚠️ unverified: whether Steam patch notes not mirrored to the wiki (e.g. a hotfix) removed `Restocker` — did not use any of the 2 permitted web fetches for this, since the task's evidence bar (decompile + blueprint grep) already conclusively shows current absence, and the exact removal date isn't load-bearing for the obsolescence verdict or recommendation.
- Did not test in-game; all findings are static (decompile + XML + patch-note text), consistent with this being a research/correlation task rather than a UAT execution task.


