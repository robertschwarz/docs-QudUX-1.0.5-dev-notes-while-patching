# UAT 04: Recipe menu shows correct ingredient quantities

> Read `uat/00-setup.md` first (install, save discipline, wish prompt = `Ctrl+W`, evidence conventions). This script does not repeat those steps.

**What was fixed:** QudUX's revamped "Cook from a recipe" menu shows how many of each ingredient you're carrying, e.g. `cured dawnglider tail(2)`. A game-version rename (`CookingGamestate` → `CookingGameState`) broke the mod build entirely on 1.0.5; it's been renamed in the mod source so the mod compiles and this counter works again. This script proves the counter itself is correct, not just that it displays.

## 1. Feature under test

- File: `Screens/QudUX_RecipeSelectionScreen.cs`, method `GetBriefIngredientList()` (~lines 387–423). Draws the ingredient line under the recipe list, appending `{{K|(N)}}` (a dim gray count) after each ingredient's name, where `N` = `CookingGameState.GetIngredientQuantity(...)` = how many of that ingredient you're currently holding.
- Reached via: campfire → `Cook` → `Cook from a recipe.` → QudUX's custom full-screen recipe picker (`[UIView("QudUX:CookRecipes")]`), which replaces the vanilla popup list via a Harmony transpiler (`Harmony Patches/Patch_XRL_World_Parts_Campfire.cs`, `Transpiler_CookFromRecipe`, patching `Campfire.CookFromRecipe`).
- Two code branches draw this line depending on whether any ingredient in the recipe needs more than 1 serving (`maxAmount == 1` vs the `else` at line ~412). **Branch coverage note:** every recipe shipped in the game (and every recipe QudUX's own wish helpers generate) uses `amount = 1` for each ingredient — confirmed by decompiling every `CookingRecipe` subclass (`HotandSpiny`, `AppleMatz`, `MushroomCider`, `GoatAndSweetLeaf`, `TongueAndCheek`, `BoneBabka`, `MahLahSoup`, `ThePorridge`, `CloacaSurprise`, `CrystalDelight`) and `CookingRecipe.FromIngredients()`. None ever construct a component with `amount > 1`; the file's own comment agrees ("I am 99% sure a recipe can never call for an amount > 1"). **The `else` branch (line ~412–421) is not reachable through any known in-game recipe or wish and is not covered by a play step below** — it's a defensive branch for a case the game currently never produces. If you find a way to trigger it, note it in Problems found, but its absence from this script is expected, not a gap.

## 2. Preconditions

- QudUX option (Options menu, **QudUX** section): **"Use revamped cooking menus (requires restart)"** = **Yes** (this is the default; if you changed it, restart the game before continuing).
- Character: `qudux-32-test-uat-character`, fresh load, clean baseline save per `00-setup.md`.
- No other game options need to change for this script.

## 3. Setup

All wishes go through the wish prompt (`Ctrl+W`).

1. Wish `masterchef`. This is a vanilla debug wish (`XRL.World.Capabilities.Wishing`) that: grants the `CookingAndGathering` skill + subskills, guarantees you know the recipe **"Hot and Spiny"** (`XRL.World.Skills.Cooking.HotandSpiny`, which needs exactly 1× `Cured Dawnglider Tail` and 1× `Spine Fruit Jam`), teaches several extra random recipes, and drops some unrelated ingredients (Fermented Yuckwheat Stem, Voider Gland Paste, misc mid-tier ingredients) in your inventory. You'll see "Added cooking knowledge."
2. Open your inventory (`i`) and confirm you have **0** "cured dawnglider tail" and **0** "spine fruit jam". (The random freebies from step 1 don't include either by design, but if you got unlucky, drop any you have with `d` before continuing.)
3. Wish `Campfire` (exact blueprint name, confirmed in `ObjectBlueprints/Furniture.xml`). It spawns in an empty cell next to you.
4. If, later, choosing "Cook from a recipe." tells you "You aren't hungry. Instead, you relax by the warmth of the fire.", wish `hungry` and retry — this is a vanilla debug wish that resets your cooking/hunger counter.

## 4. Test steps

**Step 1 — Zero ingredients shows `(0)`**
- Action: Walk onto/next to the Campfire and press `Space` (`Use`). Choose `Cook` (`c`), then `Cook from a recipe.` (`r`). If "Hot and Spiny" isn't visible in the list, select the `Show N hidden recipes missing ingredients` entry at the bottom to reveal it (this is vanilla behavior — recipes you can't currently afford are hidden by default). Scroll to **Hot and Spiny** with `2`/`8`.
- Expected: The ingredient line under the recipe reads `1 serving each of cured dawnglider tail(0) and spine fruit jam(0)` — both counts in dim gray, both `0`.
- Evidence: 📸 `UAT-04-01-zero-quantities.png` — full recipe screen with Hot and Spiny selected, ingredient line with both `(0)` values legible.

**Step 2 — Gain ingredients, inventory matches**
- Action: Press `Esc` to back out of the recipe screen and the cook menu (don't cook yet). Wish `Cured Dawnglider Tail` **twice** (2 separate wishes) and `Spine Fruit Jam` **three times** (3 separate wishes) — each wish drops exactly 1 item in a nearby empty cell (typing the exact singular blueprint name always spawns exactly 1). Walk to each item and press `g` (`Get`) to pick it up — 5 pickups total.
- Expected: Opening inventory (`i`) shows exactly 2× "cured dawnglider tail" and 3× "spine fruit jam".
- Evidence: 📸 `UAT-04-02-inventory-counts.png` — inventory screen with both stacks and their counts legible.

**Step 3 — Recipe screen count matches inventory**
- Action: Use the Campfire again → `Cook` (`c`) → `Cook from a recipe.` (`r`) → select **Hot and Spiny**.
- Expected: Ingredient line now reads `...cured dawnglider tail(2) and spine fruit jam(3)` — matching the inventory counts from Step 2 exactly.
- Evidence: 📸 `UAT-04-03-quantities-match-inventory.png` — recipe screen, both counts legible, ideally in the same screenshot/frame as noting they match Step 2's inventory (or reference the two files together in your notes).

**Step 4 — Quantities update after cooking**
- Action: With Hot and Spiny still selected, press `Space` or `Enter` to cook. Confirm `Yes` at the "Cook Hot and Spiny?" prompt. After the popup resolves, use the Campfire a third time → `Cook` (`c`) → `Cook from a recipe.` (`r`) → select **Hot and Spiny** again.
- Expected: Cooking consumes exactly 1 of each ingredient (recipe requires 1 each). The re-opened recipe screen now shows `cured dawnglider tail(1) and spine fruit jam(2)` — each count reduced by exactly 1 from Step 3.
- Evidence: 📸 `UAT-04-04-quantities-after-cook.png` — recipe screen after cooking, both counts legible and reduced by 1.

## 5. Edge / negative cases

**Step 5 — Escape/cancel doesn't consume or crash**
- Action: From the recipe list (any recipe highlighted, not yet confirmed), press `Esc` (or Numpad `5`).
- Expected: Menu closes immediately, no popup, no item consumed, no error. You're returned to normal play (or the `Cook` sub-menu, if it's still open).
- Evidence: 📸 `UAT-04-05-escape-cancel.png` — screen right after escaping, showing you're back in the game with the campfire menu gone and no error popup.

**Step 6 — Option off → feature absent**
- Action: Open Options → QudUX section → set **"Use revamped cooking menus (requires restart)"** to **No**. Restart the game, reload `qudux-32-test-uat-character`. Use the Campfire → `Cook` → `Cook from a recipe.`.
- Expected: You get the **vanilla** "Choose a recipe" popup list (a simple text popup, not QudUX's full-screen bordered menu) with **no `(N)` quantity markers** anywhere. This confirms the quantity feature is entirely gone when the option is off, not just hidden.
- Evidence: 📸 `UAT-04-06-option-off-vanilla-menu.png` — the vanilla popup list, showing no gray parenthetical counts.
- **After this step:** turn the option back to **Yes** and restart before moving to the next UAT script (per `00-setup.md` save discipline).

## 6. Log check

After finishing (still using the same Player.log — don't relaunch until you've copied it per `00-setup.md`), search `Player.log`:

```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Freehold Games\CavesOfQud\Player.log" -Pattern "Exception|CookingGamestate|CookingGameState|GetIngredientQuantity|RecipeSelectionScreen|CookFromRecipe|Campfire" 
```

- **FAIL** if any line contains `Exception` with a stack trace mentioning `QudUX`, `RecipeSelectionScreen`, `CookingGameState`, or `Campfire`.
- **FAIL** if you see `CookingGamestate does not contain a definition` or any `CS0117`-style build error text (would indicate the fix regressed).
- **FAIL** if the patch startup line for `Campfire` reads `Failed` instead of `Patched successfully.` (logged once near the top of the log, prefixed `Campfire...`).
- Informational only, not a fail: the "Added cooking knowledge." player message and normal cooking flavor messages.

## 7. Fail indicators

- Ingredient count shown in the recipe screen does **not** match your actual inventory count for that ingredient (Steps 1, 3, 4).
- Count doesn't decrease after cooking (Step 4), or decreases by the wrong amount.
- Game crashes, freezes, or throws a visible error popup when opening the recipe screen, selecting a recipe, or cooking.
- Escaping the recipe screen consumes an ingredient or leaves you stuck in a menu loop.
- With the option off (Step 6), QudUX's bordered full-screen recipe menu still appears (should be vanilla popup instead).
- Any `QudUX`-tagged exception in Player.log.

## 8. Results

| Step | PASS/FAIL | Evidence file | Notes |
|---|---|---|---|
| 1 — Zero ingredients | | | |
| 2 — Gain ingredients / inventory matches | | | |
| 3 — Recipe count matches inventory | | | |
| 4 — Count updates after cooking | | | |
| 5 — Escape/cancel | | | |
| 6 — Option off → vanilla menu | | | |
| Log check | | | |

### Problems found

For each problem, copy this block:

```
Steps to reproduce:
1.
2.
3.

Expected:

Actual:

Evidence: qudux-mod-docs/uat/evidence/04/<file>

Player.log excerpt:
```
```

## Patch-note correlation (2023–2026)

**UAT source:** `qudux_UAT.md` line 54-58 — *"# UAT 4.1: qudux menu for recipes is obsolete / it works, but the native menu of modern ui makes it obsolute. / native menu: [image-25.png]"*.

### 1. Relevant patch entries

Searched `2023.wiki`, `2024.wiki`, `2025.wiki`, `index.wiki` (2026, builds 211.33–212.17) for `cook|recipe|campfire|chef|ingredient|meal`. Cooking-related hits found (quoted verbatim):

- **2023.wiki:93** — `* Nostrums options in the cooking menu are now grayed out if they don't do anything.` — related but not causal: this is the *ingredients/nostrums* sub-menu (`Campfire.CookFromIngredients`, patched separately by `Transpiler_CookFromIngredients`), not the recipe menu this UAT covers.
- **2023.wiki:1455** — `* Eating a plant or fungi-based meal at an oven as a carnivore no longer provides satiation and may make you ill.` — unrelated (game balance, not UI).
- **2023.wiki:1466** — `* Legendary chefs no longer occasionally spawn without an oven.` — unrelated.
- **2024.wiki:308-310** — three "Fixed an issue that caused [mutations/effects/skills] added from cooking recipes to not be applied properly" — unrelated to the recipe *menu*; these are recipe-effect application bugs.
- **2024.wiki:323** — `* You may now cook with objects from adjacent cells (eg, open bodies of water).` — unrelated to menu UI.
- **2025.wiki:60, 139, 179** — crash/phrase-generation fixes for "cooking" — unrelated to the menu.
- **index.wiki:46 (Build 211.36, beta, Released March 1, 2026)** — `* Cooking with wild rice or canned Have-it-All now picks a random effect from the correct pool of effects rather using the pool of the ingredient you first cooked with this play session.` — unrelated to the menu (recipe-effect RNG bug).
- **index.wiki:212.17 (experimental, Released March 27, 2026)** — the "internal string format is now UTF16" / `TextBuilder` modding-API rewrite entry. **Worth knowing, not confirmed causal**: this is the build era where the mod's own commit log and this file's "What was fixed" note say `CookingGamestate` → `CookingGameState` broke the mod build. I could not find a patch-note line that explicitly names this class rename — flagging as ⚠️ unverified that 212.17 is the exact build that renamed it, though it is the most plausible candidate given the timing and the fact this build is described as a large modding-API/string-handling rewrite.

**No patch-note entry — in 2023, 2024, 2025, or the 2026 index — documents a change to the recipe-selection menu's UI, its ingredient-count display, or its favorite/sort behavior.** This is an explicit "found nothing" per the task instructions: the menu's *content* format (ingredient names with a parenthetical owned-count) is not called out anywhere in three years of patch notes as a new or changed feature.

- **2024.wiki:757-763 (Build 207.31 "Spring Molting" beta, Released May 10, 2024)** — `=== UI === / We added an entire new UI. (ie, we completed work on the modern UI, for folks who've been following our progress). ... here's a partial list of new screens: trade, quests, message log, character sheet, skills, equipment & inventory, tinkering, game summary, every journal tab, status effects, reputation, world generation, interact nearby, books, in-game terminals.` — **this is the actual cause** of the UAT-observed obsolescence, even though "cooking"/"recipe" is not named in the partial screen list. The cooking recipe list is not a bespoke screen — it is rendered through the generic `Popup.PickOption` API (see §2/§3 below), which was retrofitted with a "new popup" rendering path (`UIManager.UseNewPopups`) as part of this same modern-UI effort. Every popup-based menu in the game, including the vanilla recipe list, inherited the modern rendering without a dedicated changelog line, which explains why grepping for "recipe"/"cooking" turns up nothing about the UI itself.

### 2. Obsolescence verdict: **PARTLY OBSOLETE** (feature-by-feature; effectively FULLY OBSOLETE under modern UI)

Decompiled (`ilspycmd`, `Assembly-CSharp.dll`) `XRL.World.Skills.Cooking.CookingRecipe.GetAnnotatedDisplayName(bool oneline, bool showQuantity)`:

```
foreach (ICookingRecipeComponent component in Components)
{
    ...
    stringBuilder.Append(component.getDisplayName());
    if (showQuantity)
    {
        stringBuilder.Append("{{K|(");
        stringBuilder.Append(CookingGameState.GetIngredientQuantity(component));
        stringBuilder.Append(")}}");
    }
    ...
}
```
called by `GetCampfireDescription()`: `return GetAnnotatedDisplayName(oneline: false, showQuantity: true) + "\n\n" + GetDescription();`

And `XRL.World.Parts.Campfire.CookFromRecipe()` builds its option list directly from this: `list.Add(new Tuple<string, CookingRecipe>(knownRecipy.GetCampfireDescription() + "\n\n", knownRecipy));`, then calls `Popup.PickOption("Choose a recipe", ...)`.

This means the **vanilla, unpatched game already renders `{{K|(N)}}` — the exact same dim-gray parenthetical count, produced by the exact same `CookingGameState.GetIngredientQuantity()` call — next to every ingredient in the native recipe list**, feature-for-feature identical to `QudUX_RecipeSelectionScreen_Extensions.GetBriefIngredientList()` (`Screens/QudUX_RecipeSelectionScreen.cs:409,419`). This is not gated behind `Options.ModernUI` — it's in `CookingRecipe` itself, so it renders in **both** the legacy ASCII popup and the modern Unity popup.

Feature-by-feature vs. the native list entry (`GetCampfireDescription()`):
| Feature | QudUX `QudUX_RecipeSelectionScreen` | Native (`Campfire.CookFromRecipe` + `CookingRecipe`) |
|---|---|---|
| Ingredient owned-count `(N)` | Yes | **Yes — identical, same API, verified in decompile** |
| Favorite marking | `{{R|\u0003name}}` prefix, explicit favorite-first 3-tier sort (`RecipeComparator`: has-ingredients → favorite → alphabetical) | `GetAnnotatedDisplayName` also prefixes `{{R|\u0003}}` for favorites; list is sorted with `ColorUtility.CompareExceptFormattingAndCase(a.Item1, b.Item1)` — since the `\u0003` favorite marker survives formatting-strip (it's content, not a `{{...}}` tag) and sorts before ordinary letters, favorites float to the top natively too, without a dedicated 3-tier comparator |
| Recipes missing ingredients | Grayed (`{{K|name}}`), shown inline, still selectable (shows "you don't have enough" popup) | Hidden by default behind a "Show N hidden recipes missing ingredients" toggle entry (a different UX for the same goal — keeping affordable recipes to the front) |
| Effect description visible while browsing | Yes, dedicated pane, always visible for the highlighted recipe | Yes — it's baked into the same list-entry string (`GetDescription()` appended after ingredients) |
| Favorite/Forget without leaving the list | `F` / `D` hotkeys, single screen | Requires a second popup (`Cook`/`Add or Remove favorite`/`Forget`/`Back`) after picking a recipe |
| Single always-visible split list+detail layout | Yes (list pane + ingredient pane + description pane) | No — one scrollable list of full text blocks; not split into panes |

**Verdict rationale:** The headline feature this UAT file exists to test (`(N)` ingredient counts) is **already fully native** — confirmed by decompile, not by the patch notes (no changelog entry documents it, so its origin date in the base game is unverified — ⚠️ unverified whether this was always present or added recently). The remaining QudUX value-adds (single-pane layout, in-list hotkeys, no second popup) are minor UX polish, not missing functionality. Under the modern UI specifically, the native popup (image-25 in the UAT doc) already presents this cleanly in a scrollable graphical list, matching the user's own finding.

### 3. Modern-UI interaction

- The dispatch point is `XRL.UI.Popup.PickOption(...)`, decompiled at line ~1709: `if (UIManager.UseNewPopups || ForceNewPopup) { ... WaitNewPopupMessage(...) }` — else it falls through to the legacy `ScrapBuffer`/`ScreenBuffer` ASCII rendering. `UIManager.UseNewPopups` tracks the game's global `Options.ModernUI` setting (confirmed elsewhere in the same decompiled `Popup` class, e.g. `else if (!GameManager.IsOnGameContext() || Options.ModernUI)` at line 648 of the decompile).
- **QudUX's own gate is separate and does NOT check UI mode.** `Harmony Patches/Patch_XRL_World_Parts_Campfire.cs` `[HarmonyPrepare] Prepare()` only checks `Options.UI.UseQudUXCookMenus` (the mod's own "Use revamped cooking menus" checkbox) before installing the `Transpiler_CookFromRecipe` transpiler on `Campfire.CookFromRecipe`. It has no equivalent to the guard already present in `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs:16`: `&& !(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)` (where `GameOptions` = `using GameOptions = XRL.UI.Options;`). So with the QudUX option on, the mod's ASCII `[UIView("QudUX:CookRecipes", ForceFullscreen: true, ...)]` screen is pushed via `GameManager.Instance.PushGameView(...)` **unconditionally**, regardless of `Options.ModernUI`.
- The user's UAT note says the QudUX menu "works" even so — consistent with `PushGameView`/`IScreen` being a console-canvas view that Qud can still render as an overlay even with modern UI active (same mechanism the mod's other legacy-only screens use), but it sits alongside/instead-of the native modern popup rather than adapting to it. This matches the broader UAT-wide pattern (see file header: "core finding: some functionality is obsolete with the modern UI... merits a feature-flag style implementation") — QudUX's recipe screen is not modern-UI-aware, it simply overrides the vanilla method's control flow outright.
- Net effect: with the QudUX option ON, players get the QudUX ASCII screen in *both* UI modes (never the native popup) — even though, per §2, the modern native popup already offers equivalent information and a nicer presentation.

### 4. Recommendation: **gate behind a UI-mode flag** (leaning on the native `Popup.PickOption`/`GetCampfireDescription()` path when modern UI is on)

- Files/options involved:
  - `Harmony Patches/Patch_XRL_World_Parts_Campfire.cs` — `Prepare()` (line 27-37) currently checks only `Options.UI.UseQudUXCookMenus`. Add `&& !GameOptions.ModernUI` (or equivalent using-alias, matching the existing pattern in `Patch_XRL_UI_InventoryScreen.cs:16`) so the transpiler — and therefore `QudUX_RecipeSelectionScreen` — only installs under legacy/classic UI.
  - `Screens/QudUX_RecipeSelectionScreen.cs` — no functional change needed; it already works standalone under legacy UI per Step 6 of this UAT script (option off → vanilla popup, no `(N)` markers, confirming the mod is the sole source of that display under legacy UI *when the option is on*).
  - `Options.xml:11` — the `QudUX_OptionUseCookMenus` checkbox itself; consider updating its `DisplayText` to note it only applies to classic/legacy UI, mirroring the README's existing caveat for the abilities screen ("This only works in 'classic' UI mode...").
- Do **not** remove: legacy-UI players get no ingredient-count display, no favorite-priority sort, and no single-pane browsing without QudUX's screen (the vanilla ASCII popup still shows `(N)` per §2, but confirming that requires re-checking Step 6 evidence specifically for counts in the vanilla *ASCII* popup, which this UAT script did not capture — see gaps below). Since the user's standing plan is "legacy-UI users keep working features," and this screen is confirmed functional (UAT: "it works"), only modern-UI users should stop getting it once native parity is confirmed.
- Simplification option on top of the gate: since `CookingRecipe.GetCampfireDescription()` / `GetAnnotatedDisplayName(showQuantity:true)` already produces the ingredient-count string, a legacy-UI-only QudUX screen could in principle call that native method directly instead of maintaining its own parallel `GetBriefIngredientList()` — reducing future breakage risk from renames like `CookingGamestate`→`CookingGameState` recurring. This is an optional follow-up, not required for the obsolescence fix.

### 5. Confidence + gaps

**Verified (decompiled code / patch notes, not assumed):**
- `CookingRecipe.GetAnnotatedDisplayName`/`GetCampfireDescription` already emit `{{K|(N)}}` ingredient counts via `CookingGameState.GetIngredientQuantity()`, unconditionally (not gated on UI mode) — decompiled directly from `Assembly-CSharp.dll`.
- `Campfire.CookFromRecipe()` feeds those same descriptions into `Popup.PickOption`.
- `Popup.PickOption` branches on `UIManager.UseNewPopups` (tied to `Options.ModernUI`) between the legacy ASCII list and the modern Unity popup (`WaitNewPopupMessage`) — this is the "native menu" the user screenshotted.
- QudUX's own gate (`Patch_XRL_World_Parts_Campfire.cs` `Prepare()`) checks only its own option, not `Options.ModernUI` — confirmed by reading the mod source directly.
- Patch note 2024.wiki:757-763 (Build 207.31, May 10 2024) is the "modern UI completed" milestone that (indirectly, via the shared popup system) brought the native recipe-list rendering up to parity.
- No patch note in 2023–2026 documents a change to the recipe menu's content/format specifically.

**Gaps / unverified:**
- ⚠️ Unverified: the exact build/date the `CookingGamestate` → `CookingGameState` rename landed, and whether it's the same build that (if ever) *added* `showQuantity` support to `GetAnnotatedDisplayName`. No patch note names either change explicitly; Build 212.17's UTF16/TextBuilder modding rewrite (index.wiki, Released March 27, 2026) is circumstantially the closest match in timing but is not confirmed.
- ⚠️ Unverified: whether the native ingredient-count display has *always* existed (i.e., predates QudUX's own recipe screen) or was added at some point in 2023-2026 without a changelog entry — I found no patch note either way, so I cannot date this native feature's origin.
- Not independently re-tested in-game: I did not launch Caves of Qud to confirm the modern-UI popup screenshot (image-25, referenced in `qudux_UAT.md`) visually matches the decompiled `GetCampfireDescription()` output; the correlation here is from static decompilation only, not a live comparison screenshot pair.
- The existing Step 6 evidence in this file (option off → vanilla menu) doesn't have its screenshot re-inspected here for whether `(N)` counts are visible in the *legacy ASCII* popup specifically (as opposed to modern) — worth confirming during the actual UAT re-run before shipping the recommended gate, so legacy-UI users aren't left with two screens both lacking the sort/favorite polish incorrectly assumed still needed.
```
