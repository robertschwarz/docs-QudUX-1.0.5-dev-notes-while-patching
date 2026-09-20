# Error 12 — `Popup.AskString` signature change (args 3 and 4)

**File:** `Screens/QudUX_InventoryScreen.cs:634`

**Errors:**
```
error CS1503: Argument 3: cannot convert from 'int' to 'string'
error CS1503: Argument 4: cannot convert from 'int' to 'string'
```

## Status: FIXED ✅

Same root cause as Error 6. New signature from DLL reflection:
```
string AskString(
    string Message, string Default,
    string Sound, string RestrictChars, string WantsSpecificPrompt,
    int MaxLength, int MinLength,
    bool ReturnNullForEscape, bool EscapeNonMarkupFormatting,
    bool? AllowColorize
)
```

Old positional args 3 and 4 were `int maxLength, int minLength`. Now use named params to skip the new string args.

```diff
- FilterString = Popup.AskString("Enter text to filter inventory by item name.", FilterString, 80, 0);
+ FilterString = Popup.AskString("Enter text to filter inventory by item name.", FilterString, MaxLength: 80, MinLength: 0);
```

## Agent review (1.0.5)

**Verdict: CONFIRMED FIXED** — no changes made, `git diff` is empty.

**Verified:**
- Decompiled `XRL.UI.Popup.AskString` (1.0.5 DLL) full signature:
  `AskString(string Message, string Default = "", string Sound = "Sounds/UI/ui_notification", string RestrictChars = null, string WantsSpecificPrompt = null, int MaxLength = 80, int MinLength = 0, bool ReturnNullForEscape = false, bool EscapeNonMarkupFormatting = true, bool? AllowColorize = null)`
  — matches what's documented above exactly. Baseline call binds unambiguously (two positional args matching `Message`/`Default`, then named `MaxLength`/`MinLength`; no `Obsolete` attribute on the method).
- **Null-safety check (the critical risk called out):** `ReturnNullForEscape` defaults to `false` and the baseline call doesn't override it. Traced both code paths in decompiled `Popup.AskString`:
  - Legacy (non-`UseNewPopups`) path, on Escape/RightClick with `MinLength <= 0`:
    ```
    if (!ReturnNullForEscape) { return ""; }
    return null;
    ```
  - `AskStringAsync` path (used when `UIManager.UseNewPopups`), on Cancel:
    ```
    return ReturnNullForEscape ? null : "";
    ```
  Both paths return `""` (not `null`) when `ReturnNullForEscape` is `false` (the default, and what the baseline fix uses) — this reproduces old pre-1.0.5 escape behavior. `FilterString` can never become `null` from this call with the baseline args.
- Grepped all `FilterString` usages in `QudUX_InventoryScreen.cs`: only `!= ""` comparisons, string concatenation, and passing it as the `needle` to `.Contains(FilterString, ...)` on other strings — nothing calls `.Length`/`.ToLower()`/etc. directly on `FilterString` that would NPE even in a hypothetical null case.
- Sync `AskString` is not `Obsolete`; calling it synchronously from this screen is unchanged from pre-1.0.5 usage pattern elsewhere in the codebase.

**Build:** `dotnet build Mods.csproj` — zero errors touching `QudUX_InventoryScreen.cs` or `AskString`. Only remaining error is the pre-existing baseline error in `QudUX_LegendaryInteractionListener.cs` (`Brain.Factions`, issue 11, not in scope here).

**Risks / open questions:** None identified. No runtime pitfalls; behavior matches pre-1.0.5 semantics.
