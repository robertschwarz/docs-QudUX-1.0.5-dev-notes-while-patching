# Step 16 — Mark obsolete features in the source

**Lane E · Depends on: lanes A-D merged · Owns: header comments only, in the files listed below**

Read `00-README.md` first.

Run this **after** steps 01-14 are merged. It touches files those lanes own, so running it early guarantees conflicts.

## Goal

Leave a short, factual note at the top of each feature the base game has superseded, so the next person does not have to re-derive the research.

No behavior changes. Comments only.

## Why

Several QudUX features still work but duplicate something the base game added after the mod's last update. That is only true under the **modern UI** — legacy-UI players still rely on them. Branch 2 decides what goes; until then the knowledge should live next to the code, not only in a docs folder.

## What each comment says

Four lines at most, at the top of the class or the relevant method:

1. What in the base game supersedes it, and from which build and date.
2. Whether it still has value under the legacy UI.
3. That branch 2 owns the removal decision.
4. Where the evidence is (`qudux-mod-docs/uat/00-setup.md` §10, or the specific `uat/NN-*.md`).

Keep it factual. No opinions about code quality, no TODOs, no `[Obsolete]` attributes — that attribute produces build warnings and changes how callers compile.

## Targets and findings

Take these from the research; do not redo it.

| Target | Superseded by | Still useful in legacy UI? |
|---|---|---|
| `Screens/QudUX_InventoryScreen.cs` | Native graphical inventory, build 207.31 (2024-05-10); the `Modern UI > Modern character sheet` toggle, build 207.69 (2024-06-14) | **Yes** — the legacy screen has no sprite tiles, no Main/Other tab split, and **neither** native screen has value-per-weight |
| The filter prompt in `Screens/QudUX_InventoryScreen.cs` (~line 634) | Vanilla's legacy inventory already has an identical filter, same prompt text; the modern UI has a better one (live box, strict/fuzzy toggle) | Marginal — QudUX only adds an 80-char cap and a Delete-to-clear shortcut |
| `Screens/QudUX_RecipeSelectionScreen.cs`, `Screens/QudUX_IngredientSelectionScreen.cs`, `Harmony Patches/Patch_XRL_World_Parts_Campfire.cs` | Vanilla already appends the same ingredient counts via `CookingRecipe.GetAnnotatedDisplayName(showQuantity: true)`, using the very call QudUX uses | Partly — the ASCII menu itself, not the counts |
| The conversation-portrait path in `Utilities/TileMaker.cs` and `Screen Extenders/ConversationUIExtender.cs` | The modern popup path draws a portrait natively | **Yes** — the legacy render path computes the icon and discards it, so the gap is real there |

Note the campfire patch's gate as part of its comment: unlike the inventory patch, `Patch_XRL_World_Parts_Campfire.cs` checks only the QudUX option and not the UI mode, so the ASCII recipe screen currently loads in both UI modes. **Do not change that here** — it is a behavior change, and branch 2 owns it. Record it.

## Non-goals

- No code changes of any kind, including the gating just mentioned.
- Do not add `[Obsolete]` attributes.
- Do not comment features that are not on this list. `../uat/00-setup.md` §10 marks the rest as still valuable.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors and **no new warnings** (a warning means you changed something you should not have).
- `git diff` shows comment lines only.
- Every comment names a build number and points at its evidence.

## Re-validates

Nothing. Confirm the build is clean and the diff is comments.
