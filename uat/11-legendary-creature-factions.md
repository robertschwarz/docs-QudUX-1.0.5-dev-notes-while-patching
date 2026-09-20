# UAT 11: Legendary Creature Faction Relations (Journal Note)

> See `00-setup.md` for install, baseline save, wish prompt (`Ctrl+W`), and log location. This script assumes that's done.

## What was fixed

Marking a legendary ("hero") creature's location in your journal used to crash the mod's build outright, because it read faction membership from a game API (`Brain.FactionMembership`) that Caves of Qud renamed in 1.0.5. The fix reads the creature's real faction memberships from the new API (`Brain.Allegiance`), so the journal note's "(Loved by X, Admired by Y...)" tag is produced again instead of not compiling at all.

## Feature under test

`Parts and Effects/QudUX_LegendaryInteractionListener.cs`. It's a part QudUX adds to the **player** (`PartsAdding/QudUX_PartAdder_OnLoad.cs`) that listens for `OwnerGetInventoryActionsEvent`/`InventoryActionEvent`. Any nearby object that has the `Hero` property or `Role == "Hero"` (i.e. was hero-ified by the game's own `HeroMaker`, which is how village/lair "legendary" quest creatures are built) **and** has a `GivesRep` part gets two extra actions:

- **"Mark Legendary Location in Journal"** → `MarkLegendaryLocation()`: adds a journal map note (`Journal > Locations > Legendary Creatures` — the category label and `Locations` tab name are both hard-coded from the base game's `JournalScreen.STR_LOCATIONS = "Locations"`), whose entry text is `<creature name><parenthetical relation list>`, then shows a popup confirming it. The parenthetical is built by `ParentheticalListOfRelations()` — this is the method the fix touched. It walks the creature's own faction memberships (`Brain.Allegiance`, cast to `IDictionary<string,int>` to reach `.Keys`) and emits `"Loved by {{C|<faction>}}"` for each **visible** one, then walks `GivesRep.relatedFactions` (a separate, randomly-generated 1–3-entry list) and emits `"Admired by"/"Disliked by"/"Hated by" {{C|<faction>}}"` per entry, joined into one yellow-and-cyan parenthetical, e.g. `(Loved by baboons, Admired by apes)`.
- **"Remove Marked Legendary Location from Journal"** → shown instead, once a note already exists for that creature.

There's also a whole-zone version, `QudUX_LegendaryInteractionListener.BatchMarkLegendary()`, wired to a QudUX-only command in the mod's own `Commands.xml`:
```xml
<command ID="QudUX_BatchAddLegendaryEntry" DisplayText="Add zone heroes to journal" Category="QudUX" Layer="Adventure" Auto="Down"></command>
```
It has **no default keybind** — it must be bound manually (Options → Keybinds → category `QudUX` → "Add zone heroes to journal"). It marks every hero-tagged, `GivesRep`-bearing creature currently registered as a point of interest, in one popup + one journal write per creature — same `ParentheticalListOfRelations()` code path.

UI locations: the two per-creature actions appear in that creature's interaction/action list (right-click it, or the base game's "Interact nearby" command — Options → Keybinds → category "Adventuring" if you haven't bound it; the action also works at range/telepathically since QudUX registers it with `WorksAtDistance: true, WorksTelepathically: true`). The resulting note lives in **Journal (`J`) → Locations tab → "Legendary Creatures" category**.

## Preconditions

- No QudUX option gates this feature (checked `Options.xml` / `Concepts/Options.cs` — nothing there mentions "Legendary" or "Hero").
- No specific game option required. Optional: the base game's "Reveal current faction reputation on hover" style option (`Options.GivesRepShowsCurrentRep`) only affects the *Look* tooltip used for comparison in Step 4, appending a `(number)` after the faction name there — it has no effect on the QudUX journal text either way, so leave it at whatever it currently is.
- Save state: fresh baseline (`qudux-32-test-uat-character`, per `00-setup.md`), in a zone with open ground next to you (e.g. just outside a building) so two spawned creatures don't block each other.

## Setup

1. Open the wish prompt (`Ctrl+W`) and enter:
   ```
   testhero:Baboon
   ```
   This is confirmed in the decompiled 1.0.5 `XRL.World.Capabilities.Wishing` handler: `Wish.StartsWith("testhero:")` splits on `:`, creates the given blueprint, runs it through `HeroMaker.MakeHero(...)`, and places it on the tile directly **east** of you, active. `Baboon` (`Parts and Effects` n/a — game blueprint, `ObjectBlueprints/Creatures.xml:1596`, inherits `BaseBaboon` which sets `Brain Factions="Baboons-100"`) is non-hostile (`Hostile="False"`) and belongs to the visible faction `Baboons` (`Factions.xml` — no `Visible="false"` override, `DisplayName="baboons"`).
2. Step one tile away (e.g. move north) so the next hero doesn't try to spawn on the same cell as the first.
3. Open the wish prompt again and enter:
   ```
   testhero:Snapjaw
   ```
   `Snapjaw` (`ObjectBlueprints/Creatures.xml:2407-2411`) sets `Brain Factions="Snapjaws-100"` directly and is also non-hostile (`Hostile="false"`). `Snapjaws` is likewise a visible faction (`Factions.xml`, `DisplayName="snapjaws"`).
4. Confirm both hero creatures are now standing near you and near each other (both non-hostile, so this is safe).
5. `F5` to save (optional but recommended before poking at journal state).

## Test steps

**Step 1 — Action appears on a legendary creature**
Action: Open the action/interaction list on the Baboon hero (right-click it, or your bound "Interact nearby" key while adjacent).
Expected: the list includes **"Mark Legendary Location in Journal"**. This confirms the creature satisfies `HasProperty("Hero")` (or `Role=="Hero"`) *and* `HasPart(GivesRep)` — both set by `HeroMaker.MakeHero`.
Evidence: 📸 `UAT-11-01-action-menu.png` — action menu open on the Baboon hero, "Mark Legendary Location in Journal" entry visible.

**Step 2 — Mark the Baboon hero**
Action: Select "Mark Legendary Location in Journal".
Expected: a popup: `You note the location of <name> in the Locations > Legendary Creatures section of your journal.` No error dialog, no crash to desktop.
Evidence: 📸 `UAT-11-02-mark-popup-baboon.png` — full popup text visible.

**Step 3 — Journal entry text is well-formed**
Action: Open Journal (`J`) → **Locations** tab → **Legendary Creatures** category.
Expected: one entry reading `<Baboon hero name>(Loved by baboons[, Admired/Disliked/Hated by <faction>]...)`. The faction name(s) must be real, readable words (not blank, not `{{...}}` raw markup, not truncated), and the "Loved by" faction must be **baboons**.
Evidence: 📸 `UAT-11-03-journal-entry-baboon.png` — entry legible enough to read the entire parenthetical.

**Step 4 — Cross-check against the creature's own description**
Action: Press `L` (Look), target the Baboon hero, and read its description tooltip.
Expected: the tooltip includes a base-game-generated line "Loved by baboons." (from `GivesRep.AppendReputationDescription`, unrelated code to the fix) naming the **same** faction as the journal note's "Loved by" segment — confirms QudUX is reading the creature's real faction data, not stale/garbled data.
Evidence: 📸 `UAT-11-04-look-description-baboon.png`.

**Step 5 — Mark the second (different-faction) legendary**
Action: Repeat Steps 1–2 on the Snapjaw hero.
Expected: separate popup naming the Snapjaw hero, same message format as Step 2.
Evidence: 📸 `UAT-11-05-mark-popup-snapjaw.png`.

**Step 6 — Compare both entries side by side**
Action: Reopen Journal → Locations → Legendary Creatures.
Expected: **two** entries now — the Baboon hero showing `(Loved by baboons...)` and the Snapjaw hero showing `(Loved by snapjaws...)`. The faction names must differ between the two entries and each must match that creature's actual blueprint faction. This is the key regression check: it proves the fix reads **per-creature** `Brain.Allegiance`, not a cached/shared value.
Evidence: 📸 `UAT-11-06-journal-both-entries.png` — both entries visible together.

**Step 7 — Batch action (whole-zone)**
Action: With both heroes still nearby, go to Options → Keybinds → category **QudUX** → find **"Add zone heroes to journal"** and bind it to a free key (e.g. Numpad 1), then press it.
Expected: one popup: `You note the location of the following creatures in your journal:` followed by both creatures' short names on separate lines. No crash.
⚠️ unverified — confirm in game: whether both heroes are close/visible enough to register as "points of interest" at your current position; if the popup instead says "You haven't noticed any legendary creature here," step closer to both and retry.
Evidence: 🎥 `UAT-11-07-batch-mark.mp4` — record the keypress through the popup appearing, both names visible.

**Step 8 — Edge case: creature with no owned (visible) faction**
Action: Open the wish prompt and enter:
```
animatedhero
```
This hero-ifies a random `Animatable`-tagged inanimate object (confirmed in decompiled `Wishing`/`EncountersAPI.GetAnAnimatedObject`) — such objects typically have no `Factions=` set on `Brain`, so `Brain.Allegiance` is empty. Mark its location the same way as Step 2.
Expected: no exception, no crash. The parenthetical should have **no** "Loved by ..." segment (nothing to loop over), but may still show 1–3 random "Admired by/Disliked by/Hated by \<faction\>" entries — that's a separate, unaffected code path (`GivesRep.relatedFactions`), not a failure. This step specifically exercises the fixed line (`((IDictionary<string,int>)Brain.Allegiance).Keys`) against a genuinely empty collection.
Evidence: 📸 `UAT-11-08-edge-empty-faction.png` — journal entry for the animated hero.

**Step 9 — Unmark (cancel path)**
Action: Open the Baboon hero's action list again and select "Remove Marked Legendary Location from Journal".
Expected: popup `Your journal entry for <name> has been deleted.` and the entry disappears from Journal → Locations → Legendary Creatures.
Evidence: 📸 `UAT-11-09-unmark-baboon.png` — Journal screen with only the Snapjaw (and animated-hero) entries remaining.

## Edge/negative cases

- Covered above: empty/invisible-faction creature (Step 8), unmark/removal (Step 9).
- If you re-select "Mark Legendary Location in Journal" on a creature that already has a note, the mod should instead be offering "Remove..." (the two actions are mutually exclusive per `OwnerGetInventoryActionsEvent` handler) — if both appear at once, or neither does, that's a FAIL.

## Log check

After finishing, search `Player.log` (per `00-setup.md`) for:
```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "Exception|QudUX|LegendaryInteractionListener|ParentheticalListOfRelations|Allegiance|FactionMembership"
```
**FAIL** if any of the following appear:
- Any `Exception` (especially `NullReferenceException`, `InvalidCastException`, or `KeyNotFoundException`) whose stack trace mentions `QudUX_LegendaryInteractionListener`, `ParentheticalListOfRelations`, `GivesRep`, `Brain`, or `Allegiance`.
- A reference to `FactionMembership` (the old, removed API) anywhere in an exception — would indicate the fix regressed to the broken code path.

## Fail indicators

- "Mark Legendary Location in Journal" action never appears on a hero creature with `GivesRep`.
- Popup or journal entry text is missing the parenthetical, shows empty parentheses `()`, shows raw `{{C|...}}` markup instead of colored text, or is truncated/garbled.
- The two creatures (Steps 3 and 6) show identical, wrong, or swapped faction names.
- Game freezes, shows an error dialog, or crashes to desktop on mark/unmark/batch-mark.
- Player.log shows an exception per the Log check above.

## Results form

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |
| 9 | | | |

### Problems found

For each problem:

```
Steps to reproduce:
1.
2.

Expected:

Actual:

Evidence: uat/evidence/11/<file>

Player.log excerpt:
```

## Patch-note correlation (2023–2026)

### 1. Relevant patch entries

Searched `2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026, builds 211.33-212.17) for: journal, note, sultan, legendary, hero, lair, "points of interest", map marker, location, gossip, observation, chronology.

- `* [modding] Added JournalAPI.AddVillageNote to add journal notes to a village's history.` -- Build 207.87, released August 2, 2024 (`2024.wiki:443`). **Related, worth knowing.** This is additive (a brand-new method for village-history notes), not a change to `AddMapNote`/`GetMapNote`/`DeleteMapNote`, the methods QudUX actually calls. It shows the `JournalAPI` surface was actively being extended around this time, but does not itself break anything QudUX uses.
- `* Unidentified objects no longer show up as points of interest.` -- 2023 (`2023.wiki:989`). **Related, worth knowing.** Confirms the `Understood()` gate inside `GetPointsOfInterestEvent.StandardChecks` (still present in the decompiled 1.0.5 assembly) is long-standing base-game behavior, not a new 1.0.5 regression. It matters for 11.3 as one of several gates a legendary creature must clear to be considered "noticed."
- `* When navigating to a point of interest that's a creature, you now more reliably go to that creature's new location if they move.` -- 2023 (`2023.wiki:1742`). **Worth knowing**, unrelated to the failure -- confirms creatures-as-POI is an established, intentional base-game concept QudUX is building on top of, not something it invented.
- `* [modding] The internal string format is now UTF16. This means any characters you were using from CP437 in your .cs files need to be converted...` -- Build 212.17 (experimental), released March 27, 2026 (`index.wiki`, under `=== Build 212.17 (experimental) ===`). **Worth knowing, not a cause here.** `QudUX_LegendaryInteractionListener.cs` only emits plain ASCII and `{{Color|...}}` markup tokens (no literal CP437 glyphs), so this encoding change doesn't appear to touch this file. Flagged because it's the single most invasive 1.0.5-era text/modding change found in the whole 2023-2026 window and could explain garbling elsewhere in the mod's journal-adjacent text.
- No entry in any of the four files documents a signature change, rename, deprecation, or behavior change to `JournalAPI.AddMapNote`, `JournalAPI.GetMapNote`, `JournalAPI.DeleteMapNote`, `JournalAPI.RevealMapNote`, `GetPointsOfInterestEvent`, `GivesRep`, or `HeroMaker`. **Explicitly: nothing found.** The 2026 entries (builds 211.33-212.17) in particular contain zero hits for "journal", "legendary", "hero", "point of interest", or "map marker" outside the boilerplate index text -- i.e. nothing in the current beta/experimental cycle touches this system at all.
- No entry anywhere in 2023-2026 announces the base game auto-tracking legendary creatures, lairs, or bosses in the journal or on the map by itself. Searched explicitly for "legendary...journal", "boss tracker", "notorious", "notable creature" phrasing -- no hits.

### 2. Root cause for 11.2 (journal not written) -- HYPOTHESIS (not confirmed)

Compared the mod's calls against the decompiled 1.0.5 `Qud.API.JournalAPI`:

- Mod call: `Parts and Effects/QudUX_LegendaryInteractionListener.cs:53` -- `JournalAPI.AddMapNote(target.CurrentZone.ZoneID, entryText, "Legendary Creatures", secretId: secret, revealed: true, sold: true, silent: true);`
- Decompiled 1.0.5 signature: `public static void AddMapNote(string ZoneID, string text, string category = "general", string[] attributes = null, string secretId = null, bool revealed = false, bool sold = false, long time = -1L, bool silent = false)` -- parameter names, order, and defaults all still match the mod's named-argument call. This is not a renamed or resignatured method.
- Traced the method body: it builds a `JournalMapNote`, adds it to `MapNotes`/`mapNotesByZone`, calls `AddedNote()`, and -- because `revealed: true` is passed -- calls `RevealMapNote(journalMapNote, silent, "Exploration")` (or `journalMapNote.Reveal(...)` directly if the static `sorting` flag is false), which sets the note's `Revealed` flag either way. `JournalScreen.GetMapNoteCategories()` only lists categories from notes where `Revealed == true`, so "Legendary Creatures" should appear as a category once one note is revealed. Static analysis of the current API does not show a broken or no-op call.
- `ZoneID` assignment goes through `JournalMapNote.ZoneID`'s setter, which calls `XRL.World.ZoneID.Parse(...)` -- this parser is wrapped in try/catch and logs via `MetricsManager.LogError` rather than throwing, so a malformed zone ID string would not crash the call, only silently produce a garbled world/parasang/zone position (would not itself blank the text entry or its category).
- `JournalAPI.GetMapNote(secretId)` (used for the toggle-detection and for `UnmarkLegendaryLocation`) is a simple linear scan of `MapNotes` by `.ID == secretId` -- unchanged shape, no new required category/ID scheme.

**Conclusion:** no renamed method, no changed signature, no new required category enum was found in the decompiled 1.0.5 `JournalAPI`. The mod's `AddMapNote`/`RevealMapNote` call chain looks structurally sound against 1.0.5. Two explanations remain open, and could not be distinguished without a `Player.log` from the failing session (not available in this pass):
  - (a) an exception is thrown before `AddMapNote` runs (e.g., somewhere in the `OwnerGetInventoryActionsEvent`/`InventoryActionEvent` dispatch chain that hands `E.Item` to `MarkLegendaryLocation`), which would also explain why the UAT notes no success popup was screenshotted for 11.1 despite the menu item existing and being clickable; or
  - (b) this is not actually specific to the "Legendary Creatures" category at all. The UAT note for 11.2 explicitly says "(all others are also empty)" -- i.e. every journal category (Chronology, Observations, Sultans, Villages, General, Recipes) was empty, not just Locations -> Legendary Creatures. That is either a mundane artifact of testing on a fresh/short-lived save with no other journal-worthy events yet, or evidence of a systemic issue with `JournalScreen`/`JournalAPI` in this install that has nothing to do with `QudUX_LegendaryInteractionListener.cs` specifically (out of scope for this file-level investigation -- worth a separate, dedicated UAT pass on the vanilla Journal screen itself, with no QudUX interaction involved, to isolate it).
- Tag: HYPOTHESIS. No CONFIRMED break was found in the journal-writing API itself.

### 3. Root cause for 11.3 ("you haven't noticed any legendary creature here") -- CONFIRMED (mechanism) / HYPOTHESIS (why it fired in this specific test)

- Mod check, `QudUX_LegendaryInteractionListener.cs:58-69` (`BatchMarkLegendary`): builds its candidate list from `GetPointsOfInterestEvent.GetFor(The.Player)`, filters to `(point.Object.HasProperty("Hero") || point.Object.GetStringProperty("Role") == "Hero") && point.Object.HasPart(typeof(GivesRep))`, and if the filtered list is empty, shows `"You haven't noticed any legendary creature here."` (line 68) -- this is the exact string reported in the UAT.
- **Hero/Role detection is CONFIRMED intact.** Decompiled `XRL.World.HeroMaker.MakeHero` (the code path that actually produces a "legendary" creature at runtime) does:
  ```
  BaseCreature.SetIntProperty("Hero", 1);
  BaseCreature.SetStringProperty("Role", "Hero");
  ```
  and adds `BaseCreature.AddPart(new GivesRep());`. `GameObject.HasProperty` checks the `Property`/`IntProperty` dictionaries directly, and `GetStringProperty` reads the same `Property` dictionary -- both match how `HeroMaker` writes these fields (as real runtime properties, not just static-XML tags). So the mod's Hero-detection condition is compatible with 1.0.5's actual hero-creation code; this is not the break.
- **The actual gate that fails is `GetPointsOfInterestEvent` itself.** `GetPointsOfInterestEvent.GetFor(GameObject Actor, Zone Z = null)` -- signature unchanged, matches the mod's call -- dispatches the event to the zone, and each candidate object is added to the POI list only via a part's own event handler. For hero/legendary creatures that part is `GivesRep.HandleEvent(GetPointsOfInterestEvent E)`: `if (E.StandardChecks(this, E.Actor)) { E.Add(ParentObject, ...); }`. `GetPointsOfInterestEvent.StandardChecks` requires all of:
  - the object isn't the actor itself,
  - the object does not itself have an `Interesting` part (unless the calling part's `Name == "Interesting"`),
  - `Render.Visible` is true,
  - `CurrentCell.IsExplored()` is true,
  - the object isn't already in the POI list,
  - `!Object2.IsHostileTowards(gameObject)` -- i.e. the object must NOT be hostile to the player,
  - the object doesn't lead the actor (`IsLedBy`),
  - `Object2.Understood()` is true (matches the 2023 patch note above).
  If the creature the tester expected to batch-mark was hostile -- which is the common case for wild "legendary" monsters (as opposed to friendly quest-giver heroes) -- `IsHostileTowards` alone silences it from `GetPointsOfInterestEvent`'s result entirely, `legendaryCreatures.Count == 0`, and the mod prints exactly the observed message. CONFIRMED as the mechanism in the current decompiled 1.0.5 binary.
- Whether this hostile-exclusion is new in 1.0.5 (a regression) or has always gated `GetPointsOfInterestEvent` this way could not be confirmed -- no patch note in 2023-2026 mentions changing POI hostility filtering, and no older binary was available to diff against. HYPOTHESIS that this specific check is what tripped the tester's run (plausible and consistent with all evidence, but not independently verified against the tester's exact test creature).
- Corroborating detail: the mod's own existing UAT test script for this exact file (Steps 1-9, above) is explicitly written against non-hostile wished-in test heroes (`testhero:Baboon`, `testhero:Snapjaw`, both `Hostile="False"` per blueprint), and Step 7 already flags: "unverified -- confirm in game: whether both heroes are close/visible enough to register as 'points of interest'... if the popup instead says 'You haven't noticed any legendary creature here,' step closer to both and retry." -- i.e. the script's own author anticipated this exact failure mode as a visibility/POI-registration issue, not a crash.

### 4. Obsolescence verdict: STILL VALUABLE

No patch note in 2023, 2024, 2025, or the 2026 entries (builds 211.33-212.17) shows the base game adding native journal or map tracking of legendary/hero creatures or their lairs. The vanilla `JournalScreen` tab list is still `{ Locations, Observations, Sultans, Villages, Chronology, General, Recipes }` -- there is no "Legendary Creatures" or "Bestiary" tab, and no code path found that auto-writes a map note when the player encounters a hero-tagged creature (the only autowritten map/journal content found in the decompiled `JournalAPI`/`GivesRep` is the water-ritual "slew \<legendary creature\>" accomplishment on death, an unrelated combat-log feature, not a location marker). The base game's own hero/lair-related mechanic remains `GivesRep.HandleEvent(CanGiveDirectionsEvent)` -- asking a non-hostile, non-Lost/Confused hero for directions -- which is a one-off dialogue hint, not a persistent, revisitable journal entry. QudUX's feature (a permanent, categorized, revisitable "Legendary Creatures" journal page, plus a whole-zone batch command) is still something the base game does not do. This is a repair/bug-fix job, not a "the base game already covers this" retirement.

### 5. Modern-UI interaction

`Parts and Effects/QudUX_LegendaryInteractionListener.cs` hooks the base `OwnerGetInventoryActionsEvent`/`InventoryActionEvent` action-list mechanism directly (no Harmony patch, no custom screen), so it should surface through both classic and Modern UI action menus without an explicit compatibility shim -- this part of the design is UI-agnostic and doesn't need a feature-flag decision on those grounds. The only UI-adjacent code actually touching journal display, `Harmony Patches/Patch_XRL_UI_JournalScreen.cs` (adds an explored/unexplored indicator prefix to Locations-tab entries, gated by `Options.Exploration.TrackLocations`), is entirely commented out in the current source -- none of its `[HarmonyPatch]` attributes are live, so it is not presently patching anything and is not a candidate cause for 11.2's empty categories (it's inert, not broken). A feature-flag implementation for UAT 11 should:
  - Gate the "Mark Legendary Location"/"Add zone heroes to journal" actions behind a single `Options.cs` flag (there currently is none -- the existing doc notes "No QudUX option gates this feature"), so it can be disabled wholesale if the underlying journal-write path can't be fixed quickly.
  - Once 11.2's root cause is confirmed (via a `Player.log` capture -- see Gaps below), decide whether the fix belongs in `JournalUtilities.cs` (as a wrapped/defended `AddMapNote` call with try/catch + a `Logger` line so future failures aren't silent) or is purely a base-game/save-state artifact outside QudUX's control.
  - For 11.3, either accept the hostile-creature limitation as documented behavior (update the popup text to something like "You haven't noticed any non-hostile legendary creature here" so the message stops implying a bug), or extend `BatchMarkLegendary`'s filter to also include zone-resident heroes regardless of `IsHostileTowards` by not routing through `GetPointsOfInterestEvent` at all -- e.g. scanning the zone's objects directly for `Hero`-tagged, `GivesRep`-bearing creatures the player has line-of-sight to, bypassing the hostility gate.

### 6. Recommendation

- Fix, don't remove or rebuild on a different native API -- no equivalent native API exists to rebuild on (see section 4); the base game still has nothing that replaces this feature.
- Concretely:
  1. Instrument `MarkLegendaryLocation` (`QudUX_LegendaryInteractionListener.cs:49-56`) and `BatchMarkLegendary` (`:58-90`) with a try/catch + `QudUX.Utilities.Logger.Log(...)` around the `JournalAPI.AddMapNote` call, then re-run 11.1/11.2 once with `Player.log` open, to convert the current HYPOTHESIS on 11.2 into a CONFIRMED root cause (this file's own "Log check" section already documents the exact `Select-String` command to use).
  2. For 11.3, decide product-wise whether "legendary creatures I'm currently fighting" should be batch-markable; if yes, stop relying solely on `GetPointsOfInterestEvent.GetFor` (which hard-excludes hostiles via `StandardChecks`) and instead iterate the zone's objects directly for `Hero`/`GivesRep` matches, applying only the visibility/explored checks QudUX actually wants.
  3. Files involved: `Parts and Effects/QudUX_LegendaryInteractionListener.cs` (both fixes), `Utilities/JournalUtilities.cs` (if a shared defensive-logging helper for journal writes is added), `Concepts/Options.cs`/`Options.xml` (if a feature flag is added per section 5).
- Small usability item (11.1 side note): "Mark Legendary Location in Journal" is a one-way `AddMapNote` call with no check for an existing note before writing a new one when re-clicked (only the menu label toggles between Mark/Unmark, based on `JournalAPI.GetMapNote(secretId) == null` at menu-build time, lines 24-31) -- repeated clicks before the menu refreshes can, per `JournalAPI.AddedNote`, log a `MetricsManager.LogError("Duplicate entry ID: ...")` for the reused `secretId` without preventing the duplicate `JournalMapNote` from being appended to `MapNotes`. Recommend either disabling/hiding the action once a note exists (instead of relying on menu-refresh timing) or making `MarkLegendaryLocation` itself check `GetMapNote` before adding.

### 7. Confidence + gaps

- High confidence / CONFIRMED: `HeroMaker.MakeHero`'s property-writing (matches mod's Hero-detection); `GetPointsOfInterestEvent`/`GivesRep`'s hostile-exclusion mechanism as the code-level explanation for the 11.3 message; `AddMapNote`/`GetFor`/`GetMapNote` signatures all unchanged vs. what the mod calls; no 2023-2026 patch note documents a breaking change to any of these APIs; no native base-game legendary-creature journal/map tracking exists as of build 212.17.
- HYPOTHESIS / gaps: exact root cause of 11.2 (journal staying empty) -- could not confirm from static code alone; needs a `Player.log` capture from an actual failing run (this file's own "Log check" section has the command ready). Whether the 11.3 hostile-exclusion is a genuine regression introduced during 1.0/1.0.5 development, versus pre-existing behavior the mod never handled -- no older assembly was available to diff. Whether "(all others are also empty)" in 11.2 reflects a fresh save with no other journal content yet, versus a systemic `JournalScreen` problem unrelated to this file -- out of this topic's scope, worth a dedicated vanilla-journal UAT pass.
- Web use: none required -- all needed patch-note and API evidence was available locally.
