# Step 03 — Give every confirmation prompt an explicit default

**Lane A · Depends on: 02 · Owns: the `Popup.ShowYesNo` call sites listed below**

Read `00-README.md` first.

## Goal

Make sure no confirmation prompt can silently answer itself `Yes`.

## Why

`UIManager.UseNewPopups` is `Options.ModernUI`, and under the modern UI `Popup.ShowYesNo` **returns its `defaultResult` immediately, before the popup renders**. That parameter's own default is `Yes`. So on a modern-UI setup, a prompt that omits `defaultResult` answers itself `Yes` without the player ever seeing it.

That is how "Remove ALL of your auto-pickup exclusions?" can wipe the player's settings from a single keypress. UAT confirmed the modal never appears.

## Call sites

| File | Line | Prompt | Destructive? |
|---|---|---|---|
| `Screens/QudUX_AutogetManagementScreen.cs` | 143 | `Remove auto-pickup exclusion for {item}?` | yes |
| `Screens/QudUX_AutogetManagementScreen.cs` | 158 | `Remove ALL of your auto-pickup exclusions?` | yes, worst case |
| `Screens/QudUX_RecipeSelectionScreen.cs` | 250 | `Are you sure you want to forget your recipe for …?` | yes |
| `Screens/QudUX_RecipeSelectionScreen.cs` | 288 | `Cook …?` | no |
| `Screens/QudUX_InventoryScreen.cs` | 781 | move category Main → Other | no |
| `Screens/QudUX_InventoryScreen.cs` | 790 | move category Other → Main | no |

All six are written as `Popup.ShowYesNo(message)` with no other arguments.

## Required change

Pass `defaultResult` explicitly at all six sites, using named arguments. The 1.0.5 signature is:

```csharp
DialogResult ShowYesNo(string Message, string Sound = "Sounds/UI/ui_notification",
                       bool AllowEscape = true, DialogResult defaultResult = DialogResult.Yes,
                       Action<DialogResult> callback = null)
```

- **Every destructive prompt gets `defaultResult: DialogResult.No`.** Losing a recipe or an entire exclusion list to an unrendered popup is unacceptable; doing nothing is fine.
- The two non-destructive ones also get an explicit default. Pick `No` unless the surrounding code clearly reads better with `Yes` — say which you chose and why.
- Do **not** pass `Sound: null`. That silently mutes the prompt. Leave `Sound` at its default.

## Non-goals

- Do not try to make popups render correctly under the modern UI. That is engine behavior and belongs to the modern-UI branch.
- Do not touch `QudUX_AutogetHelper.cs` — step 04 owns it.
- Do not reword any prompt.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- `grep -rn "ShowYesNo" --include=*.cs .` shows every live call passing `defaultResult`.
- No call passes `Sound`.

## Re-validates

`../uat/10` (auto-pickup exclusions) and `../uat/04` (cooking menu). Under the modern UI the prompts still will not render — the point is that the silent answer is now the harmless one.
