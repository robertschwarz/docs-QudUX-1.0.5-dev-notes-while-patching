# Error 4 — `CookingGamestate.GetIngredientQuantity` removed

**File:** `Screens/QudUX_RecipeSelectionScreen.cs:409, 419`

**Error:**
```
error CS0117: 'CookingGamestate' does not contain a definition for 'GetIngredientQuantity'
```

## Status: FIXED ✅

`CookingGamestate` (lowercase 's') is `[Obsolete]`. The replacement is `CookingGameState` (capital 'S', same namespace: `XRL.World.Skills.Cooking`).

`GetIngredientQuantity` exists on the new type with the same signature:
```
static int GetIngredientQuantity(ICookingRecipeComponent component)
```

The `using XRL.World.Skills.Cooking;` directive was already present in the file.

```diff
- CookingGamestate.GetIngredientQuantity(recipe.Components[i])
+ CookingGameState.GetIngredientQuantity(recipe.Components[i])
```

Applied at both line 409 and 419.

## Agent review (1.0.5)

**Verdict: CONFIRMED FIXED** — no changes needed, delta patch is empty.

Verified against decompiled 1.0.5 `Assembly-CSharp.dll` (`ilspycmd -t XRL.World.Skills.Cooking.CookingGameState`):

```csharp
public static int GetIngredientQuantity(ICookingRecipeComponent component)
{
	if (ingredientQuantity == null)
	{
		ingredientQuantity = new Dictionary<string, int>();
	}
	string ingredientId = component.getIngredientId();
	if (!ingredientQuantity.ContainsKey(ingredientId))
	{
		ingredientQuantity[ingredientId] = component.PlayerHolding();
	}
	return ingredientQuantity[ingredientId];
}
```

- Static method, same signature `(ICookingRecipeComponent)`, returns `int`. Matches.
- Backed by a `static Dictionary<string,int> ingredientQuantity` field — no dependency on the `instance` singleton, so no null-instance risk when the recipe screen is open.
- Value comes from `component.PlayerHolding()` — quantity of that ingredient the player is holding — same semantic as expected ("quantity available").
- Diffed against pre-1.0.5 original (`git show HEAD:"Screens/QudUX_RecipeSelectionScreen.cs"`): the only change between original and new game API is the type name (`CookingGamestate` → `CookingGameState`), call site and args are byte-for-byte identical. No signature drift, no param reordering.
- Grepped the whole file for `CookingGamestate`/`CookingGameState` — only the two call sites at 409/419 reference it; no other obsolete-type usages in scope.

**Build:** `dotnet build Mods.csproj` — zero errors on lines related to this issue. Only remaining error in the tree is the known, out-of-scope `Brain.Factions` issue (#11, another agent's).

**Runtime risks:** None identified. Static dictionary cache is reset via `ResetInventorySnapshot()` elsewhere in the game's own flow, unrelated to this call site — behavior matches pre-1.0.5.

**Status: FIXED ✅** (unchanged)
