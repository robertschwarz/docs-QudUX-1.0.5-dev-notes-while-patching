# Error 10 — `Popup.ShowYesNo` signature change

**File:** `Parts and Effects/QudUX_AutogetHelper.cs:91`

**Errors:**
```
error CS1503: Argument 2: cannot convert from 'bool' to 'string'
error CS1503: Argument 3: cannot convert from 'XRL.UI.DialogResult' to 'bool'
```

## Status: CORRECTED ✅

New signature from DLL reflection:
```
DialogResult ShowYesNo(
    string Message,
    string Sound,
    bool AllowEscape,
    DialogResult defaultResult,
    Action<DialogResult> callback
)
```

Old arg 2 was `bool allowCancel`. New arg 2 is `string Sound`. Old `DialogResult default` moved to arg 4.

```diff
- choice = Popup.ShowYesNo("...", false, DialogResult.Cancel);
+ choice = Popup.ShowYesNo("...", null, false, DialogResult.Cancel);
```

## Agent review (1.0.5)

**Verdict: CORRECTED**

Baseline compiled fine, but `Sound: null` was a behavior regression, not a neutral fix.

**Verified against decompiled 1.0.5 `XRL.UI.Popup`:**
```
public static DialogResult ShowYesNo(string Message, string Sound = "Sounds/UI/ui_notification",
    bool AllowEscape = true, DialogResult defaultResult = DialogResult.Yes,
    Action<DialogResult> callback = null)
```
- No `[Obsolete]`, fully synchronous, safe to call as before (there's a separate `ShowYesNoAsync`, unused here — not applicable).
- `defaultResult` default is `DialogResult.Yes`, but call site already passes `DialogResult.Cancel` explicitly, so no change needed there.
- Old pre-1.0.5 call was `Popup.ShowYesNo(msg, false, DialogResult.Cancel)` — 2-arg-shifted version with no `Sound` param, meaning it always played whatever the hardcoded UI sound was. The new default (`"Sounds/UI/ui_notification"`) is the direct successor of that behavior.
- Baseline's `Sound: null` doesn't crash (`SoundManager.PlayUISound` guards with `!Clip.IsNullOrEmpty()`, decompiled at `SoundManager.PlayUISound` line ~248) but it silently mutes the popup sound — a behavior change from original, not just a compile fix.
- `DialogResult.Yes`/`.No`/`.Cancel` enum values and comparisons in the surrounding `while`/`if` are unaffected — same enum, same members.

**Fix applied** (named args, keeps `Sound` at its default instead of forcing null):
```diff
-                            + "If you proceed, this message will not be shown again.\n\nProceed?", null, false, DialogResult.Cancel);
+                            + "If you proceed, this message will not be shown again.\n\nProceed?", AllowEscape: false, defaultResult: DialogResult.Cancel);
```

**Build:** `dotnet build Mods.csproj` — 0 errors touching this file/issue. Only remaining error is unrelated issue 11 (`QudUX_LegendaryInteractionListener.cs` / `Brain.Factions`).

**Risks/open questions:** None outstanding. Restores original notification-sound behavior; no other pitfalls found.
