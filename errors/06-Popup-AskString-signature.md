# Error 6 — `Popup.AskString` signature change (arg 3)

**File:** `Screens/QudUX_CharacterTileScreen.cs:384`

**Error:**
```
error CS1503: Argument 3: cannot convert from 'int' to 'string'
```

## Status: FIXED ✅

New signature from DLL reflection:
```
string AskString(
    string Message, string Default,
    string Sound, string RestrictChars, string WantsSpecificPrompt,
    int MaxLength, int MinLength,
    bool ReturnNullForEscape, bool EscapeNonMarkupFormatting,
    bool? AllowColorize
)
```

Args 3–5 are now string params (`Sound`, `RestrictChars`, `WantsSpecificPrompt`). The old positional `int` args are now named params further along.

Fix: use named params to skip the new string args.

```diff
- string entry = Popup.AskString("Enter text to filter by object name.", "", 30);
+ string entry = Popup.AskString("Enter text to filter by object name.", "", MaxLength: 30);
```

Also covers Error 12 (same root cause, same fix pattern — see `12-Popup-AskString-signature-arg4.md`).

## Agent review (1.0.5)

**Verdict: CONFIRMED FIXED**

Decompiled `XRL.UI.Popup` (1.0.5 `Assembly-CSharp.dll`) shows exactly one sync overload plus one async overload — no ambiguity risk:

```
public static string AskString(string Message, string Default = "", string Sound = "Sounds/UI/ui_notification",
    string RestrictChars = null, string WantsSpecificPrompt = null, int MaxLength = 80, int MinLength = 0,
    bool ReturnNullForEscape = false, bool EscapeNonMarkupFormatting = true, bool? AllowColorize = null)

public static async Task<string> AskStringAsync(string Message, string Default = "", int MaxLength = 80,
    int MinLength = 0, string RestrictChars = null, bool ReturnNullForEscape = false,
    bool EscapeNonMarkupFormatting = true, bool? AllowColorize = null, bool pushView = false,
    string WantsSpecificPrompt = null)
```

Checks performed:
- (a) Binds unambiguously: only one `AskString` (sync) exists; `AskStringAsync` is a differently-named method, so `Popup.AskString("...", "", MaxLength: 30)` resolves cleanly to the sync overload.
- (b) Skipped-param defaults preserve old behavior: `Sound` defaults to the standard UI notification sound (matches legacy call sites), `ReturnNullForEscape = false` (escape yields `""`, not `null` — fine either way since call site uses `string.IsNullOrEmpty`), `EscapeNonMarkupFormatting = true`, `AllowColorize = null`, `MinLength = 0`, `RestrictChars = null`, `WantsSpecificPrompt = null`. None of these change caller-visible semantics versus the pre-1.0.5 3-arg call.
- (c) Sync `AskString` is not `[Obsolete]` and has no deprecation comment in the decompile. Internally, if `UIManager.UseNewPopups` is true it delegates to `AskStringAsync(...).Result` (blocking on the async task) and otherwise runs the legacy synchronous input loop directly — both paths are safe to call from this screen's synchronous `UpdateSearchString` method, matching the original call site's blocking usage pattern. No async/await refactor is required or warranted here.

Call site verified (`Screens/QudUX_CharacterTileScreen.cs:384`, inside `UpdateSearchString`):
```csharp
string entry = Popup.AskString("Enter text to filter by object name.", "", MaxLength: 30);
if (!string.IsNullOrEmpty(entry)) { ... }
```

**Diff applied:** none — baseline fix already correct, no changes made in this review.

**Build result:** `dotnet build Mods.csproj` — zero errors in this file. Only remaining build error is the pre-existing, out-of-scope Issue 11 (`QudUX_LegendaryInteractionListener.cs` `Brain.Factions`).

**Runtime risks / open questions:** none identified. Behavior is equivalent to pre-1.0.5 code; no obsolete API used; no ambiguous overload resolution.
