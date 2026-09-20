# Error 3 — `Popup.PickSeveral` signature change

**File:** `Parts and Effects/QudUX_QuickPickupPart.cs:173`

**Error:**
```
error CS1503: Argument 2: cannot convert from 'string[]' to 'string'
```

## Status: CORRECTED ⚠️→✅

`PickSeveral` was completely redesigned in 1.0.5. New signature (from DLL reflection):

```
List<(int, int)> PickSeveral(
    string Title, string Intro, string SpacingText, string Sound,
    IReadOnlyList<string> Options,
    IReadOnlyList<char> Hotkeys, IReadOnlyList<int> Stacks,
    IReadOnlyList<IRenderable> Icons,
    GameObject Context, IRenderable IntroIcon, Action<int> OnResult,
    int Amount, int Spacing, int MaxWidth, int DefaultSelected, int IconPosition,
    bool RespectOptionNewlines, bool AllowEscape,
    bool CenterIntro, bool CenterIntroIcon, bool ForceNewPopup
)
```

Key changes:
- `Options` moved to arg 5 (was arg 2)
- Return type changed from `List<int>` to `List<(int index, int stack)>`
- `Icons` is now `IReadOnlyList<IRenderable>` — `List<IRenderable>` works directly

Applied fix:

```diff
- List<int> results = Popup.PickSeveral(
-     "Which item do you want to get ?",
-     options.ToArray(),
-     AllowEscape: true,
-     Icons: icons.ToArray()
- );
- if (results == null || results.Count == 0) return;
- foreach (int index in results)
-     selectedItems.Add(selection[index]);

+ var results = Popup.PickSeveral(
+     "Which item do you want to get ?",
+     null, null, null,
+     options,
+     null, null,
+     icons,
+     null, null, null,
+     0, 0, 0, 0, 0,
+     false, true, false, false, false
+ );
+ if (results == null || results.Count == 0) return;
+ foreach (var result in results)
+     selectedItems.Add(selection[result.Item1]);
```

## Agent review (1.0.5)

**Verdict: CORRECTED** — baseline compiled but was semantically broken (would have made the picker unusable).

### What was verified

Decompiled `XRL.UI.Popup.PickSeveral` (1.0.5) declared signature with defaults:

```
List<(int Selected, int Amount)> PickSeveral(
    string Title = "", string Intro = null, string SpacingText = "", string Sound = "Sounds/UI/ui_notification",
    IReadOnlyList<string> Options = null, IReadOnlyList<char> Hotkeys = null, IReadOnlyList<int> Stacks = null,
    IReadOnlyList<IRenderable> Icons = null, GameObject Context = null, IRenderable IntroIcon = null,
    Action<int> OnResult = null, int Amount = -1, int Spacing = 0, int MaxWidth = 60, int DefaultSelected = 0,
    int IconPosition = -1, bool RespectOptionNewlines = false, bool AllowEscape = false, bool CenterIntro = false,
    bool CenterIntroIcon = true, bool ForceNewPopup = false)
```

Traced the method body:
- `Amount`: gating check on Accept is `if (Amount >= 0 && list.Count > Amount) { Show("You cannot select more than " + Grammar.Cardinal(Amount) + " options!"); continue; }`. Baseline passed literal `0` for Amount. Since `Amount >= 0` is true and any non-empty selection has `list.Count > 0`, **the Accept button would reject every non-empty selection, every time** — the picker becomes impossible to complete with more than zero items selected. Default is `-1` (unlimited), which is what the old pre-1.0.5 call implicitly meant (no cap was ever specified).
- `MaxWidth`: baseline passed literal `0`. Used as `popupTextBuilder.MaxWidth = MaxWidth - 4` (→ -4) and `int num4 = Math.Min(MaxWidth, ...)` (→ popup width clamped to 0). Would break the popup layout/rendering. Default is `60`.
- `IconPosition`: baseline passed literal `0`. Body only adds icon padding when `Icons != null && IconPosition == -1` (`num3 += 2`). Passing `0` means icons are rendered without the padding reserved for them → cramped/overlapping icon column. Default is `-1`.
- `CenterIntroIcon`: baseline passed literal `false`; default is `true`. Not observably relevant to this call (no `IntroIcon`/`Context` supplied here) but there's no reason to deviate from default.
- `Title` vs `Intro`: confirmed via `PickOption` (which `PickSeveral` delegates to) that `Title` renders as the popup window's header/title bar text, matching the original pre-1.0.5 call's intent (`"Which item do you want to get ?"` as the prompt/header). Baseline's choice to map it to `Title` was correct.
- `Options`/`Icons` are `IReadOnlyList<T>` params — `List<string>`/`List<IRenderable>` implement that directly, so no `.ToArray()` needed (baseline had already dropped the `.ToArray()` calls correctly).
- Return type `List<(int Selected, int Amount)>`: `result.Item1` is the selected index into `Options`. Confirmed `Stacks` stays `null` in this call (not used by QuickPickupPart), so each accepted option always adds `(index, 1)` — the `Amount`/stack-quantity part of the tuple is always `1` here and irrelevant; using only `result.Item1` to index `selection` (as baseline did) is correct and unchanged.

### Fix applied

Replaced the fragile positional call (17 literal placeholder args) with named arguments that only set what actually needs to differ from the declared defaults, letting the compiler-safe defaults do the rest:

```diff
 var results = Popup.PickSeveral(
-    "Which item do you want to get ?",
-    null, null, null,
-    options,
-    null, null,
-    icons,
-    null, null, null,
-    0, 0, 0, 0, 0,
-    false, true, false, false, false
+    Title: "Which item do you want to get ?",
+    Options: options,
+    Icons: icons,
+    AllowEscape: true
 );
```

### Build result

`dotnet build Mods.csproj` → zero errors related to this file. Only remaining error is the pre-existing, unrelated `Brain.Factions` issue in `QudUX_LegendaryInteractionListener.cs` (issue 11, owned by another agent).

### Runtime risks / open questions

- None expected from this call site now — all deviations from the declared defaults are gone except the three intentionally-needed ones (`Title`, `Options`, `Icons`, `AllowEscape`).
- Not runtime-tested in-game (no game harness in this task); verified via decompiled source logic tracing only.
