# Step 04 — Stop the auto-pickup confirmation from looping forever

**Lane A · Depends on: 03 · Owns: `Parts and Effects/QudUX_AutogetHelper.cs`**

Read `00-README.md` first.

## Goal

Make the first-time auto-pickup confirmation terminate even when the popup never renders.

## Why

`HandleEvent(InventoryActionEvent)` wraps the prompt in a loop that only exits on `Yes` or `No`, while asking for a default of `Cancel`:

```csharp
DialogResult choice = DialogResult.Cancel;
while (choice != DialogResult.Yes && choice != DialogResult.No)
{
    choice = Popup.ShowYesNo("Disabling auto-pickup for " + Grammar.Pluralize(E.Item.DisplayNameOnly) + ".\n\n"
        + "Changes to auto-pickup preferences will apply to ALL of your characters. "
        + "If you proceed, this message will not be shown again.\n\nProceed?",
        AllowEscape: false, defaultResult: DialogResult.Cancel);
}
```

Under the modern UI, `ShowYesNo` returns `defaultResult` immediately without rendering. `Cancel` never satisfies the exit condition, so this becomes a tight infinite loop that calls `ShowYesNo` forever — a freeze with a spinning core, not a hang on input.

On the tested setup the player did not hit this, because the `Metadata:InfoboxWasShown` flag was already `Yes` and the code took the `else` branch that skips the prompt. A fresh install walks straight into it.

## Required change

Bound the loop. Keep the intent (Escape must not dismiss the prompt) but guarantee termination:

- Cap the attempts at a small number, or exit as soon as a returned `Cancel` shows the popup is not interactive.
- Treat the bail-out as **"cancel, change nothing"**: do not set `ShouldAutoget:{blueprint}` to `No`, and do not set the `Metadata:InfoboxWasShown` flag. The player never saw the message, so it must not be marked as shown.
- Log the bail-out once so it is visible in `Player.log`.

`AllowEscape: false` and `defaultResult: DialogResult.Cancel` stay as they are — step 03's "destructive defaults to No" rule does not apply here, because `Cancel` is the signal this step keys on.

## Non-goals

- Do not make the exclusion apply anyway "to be helpful". Silently changing a global setting the player never confirmed is the bug, not the fix.
- Do not touch the six call sites in step 03.
- Do not change the settings file format or its location.

## Acceptance

- `dotnet build Mods.csproj` → 0 errors.
- The loop provably terminates when `ShowYesNo` always returns `Cancel` — trace it by hand and state the bound in your hand-back.
- Bailing out leaves both settings keys untouched.

## Re-validates

`../uat/10`. To exercise the first-run path, back up and then delete `QudUX_AutogetSettings.json` from the installed mod folder so `Metadata:InfoboxWasShown` is unset.
