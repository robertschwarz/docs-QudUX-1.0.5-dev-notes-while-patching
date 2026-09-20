# Step 12 — Stop quick pickup from discarding shields and tools

**Lane D · Depends on: nothing · Owns: `Parts and Effects/QudUX_QuickPickupPart.cs` (`BuildPopup`)**

Read `00-README.md` first.

## Goal

Make every item type the quick-pickup menu collects actually appear in it.

## Why

`BuildPopup` gathers five categories per cell:

```csharp
candidates.AddRange(currentCell.GetObjectsThatInheritFrom("Armor"));
candidates.AddRange(currentCell.GetObjectsThatInheritFrom("MeleeWeapon"));
candidates.AddRange(currentCell.GetObjectsThatInheritFrom("MissileWeapon"));
candidates.AddRange(currentCell.GetObjectsThatInheritFrom("Shield"));
candidates.AddRange(currentCell.GetObjectsThatInheritFrom("Tool"));
```

then orders only three of them and overwrites the list:

```csharp
List<GameObject> orderedGameObjects = new List<GameObject>();
foreach(GameObject obj in SelectAndOrderObjects(selection, "MeleeWeapon")) { orderedGameObjects.Add(obj); selection.Remove(obj); }
foreach(GameObject obj in SelectAndOrderObjects(selection, "MissileWeapon")) { orderedGameObjects.Add(obj); selection.Remove(obj); }
foreach(GameObject obj in SelectAndOrderObjects(selection, "Armor")) { orderedGameObjects.Add(obj); selection.Remove(obj); }

selection = orderedGameObjects;
```

Shields and tools are gathered, filtered, then silently dropped on that last line. The player sees a menu missing items they can plainly see on the ground. This predates the 1.0.5 port.

## Required change

Guarantee no candidate can be lost. Preferred shape: after the three ordering passes, append whatever remains in `selection` to `orderedGameObjects` before the reassignment. That fixes shields and tools **and** makes the bug unrepeatable if someone adds a sixth category later.

If you prefer explicit `Shield` and `Tool` passes instead, still append the remainder afterwards — the point is that the reassignment must not be able to drop anything.

Keep the existing order for the three categories that already work; weapons before armor is deliberate.

## Non-goals

- Do not change what `FilterObjectsOnCell` does or how settings toggles work.
- Do not add or remove candidate categories.
- Do not touch `AutoAct/PickupSelection.cs` — step 13 owns it.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- Drop a shield and a tool alongside a weapon and armor; all four appear in the menu.
- The existing three categories keep their relative order.

## Re-validates

`../uat/03`, which currently tells the tester to avoid shields and tools to prevent false failures. Once fixed, that warning can come out — note it in your hand-back so the script gets updated.
