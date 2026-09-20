# UAT 02: TileMaker rendering — OBSOLETE, SKIPPED

**Status:** SKIPPED — feature fully superseded by native game UI. See `evidence/02/OBSOLETE.md`.

**What was fixed:** creature sprite rendering (`Utilities/TileMaker.cs`) broke on game 1.0.5 because the game's rendering API changed.

**Why skipped:** TileMaker had two callers. Both are obsolete:

| Caller | Reason obsolete |
|--------|----------------|
| `QudUX_InventoryScreen.cs` — inventory sprites | QudUX inventory screen superseded by native graphical inventory (build 207.31, May 2024) |
| `ConversationUIExtender.cs` — NPC portrait | Native game renders full graphical NPC portraits in interaction menu and look screen |

No remaining callers. Nothing to test.

See also:
- `evidence/02/OBSOLETE.md` — full obsolescence record with in-game evidence
- `evidence/09/OBSOLETE.md` — inventory screen obsolescence
- `../errors/02-render-property-not-method.md` — original issue

## Patch-note correlation (2023–2026)

### 1. Relevant patch entries

**Cause — inventory sprite caller:**

> `== 207.31: Spring Molting beta ==` [https://freeholdgames.itch.io/cavesofqud/devlog/729335/spring-molting-beta-out-now Released May 10, 2024.]
> "We added an entire new UI. (ie, we completed work on the modern UI, for folks who've been following our progress). More polish will be coming over the next several weeks, but all the functionality is there."
> "There are too many changes to fully document, but here's a partial list of new screens: trade, quests, message log, character sheet, skills, **equipment & inventory**, tinkering, game summary, every journal tab, status effects, reputation, world generation, interact nearby, books, in-game terminals."
— `2024.wiki` lines 754–760. This is the entry `evidence/09/OBSOLETE.md` cites for the native graphical inventory; **the build/date claim (207.31, May 10 2024) is correct**, verified verbatim in the local wiki dump.

> `== 207.69 ==` [Released June 14, 2024.]
> "Added an option to disable the modern character sheet while keeping other elements of the new UI (Modern UI > Modern character sheet)."
— `2024.wiki` line 592. **Cause** — this is the literal option QudUX's own gate checks (see §3). `evidence/09/OBSOLETE.md`'s claim that 207.69 added this toggle is also correct.

**Related, not a direct cause:**
> `2024.wiki` line 588: "Added an additional scale option for the character sheet (Modern UI > Character sheet additional scale percentage)." — same build, cosmetic option, not the on/off toggle QudUX depends on.

**Gap — no patch note names the conversation/dialogue screen:**
I grepped `2023.wiki`, `2024.wiki`, `2025.wiki`, and `index.wiki` (2026, builds 211.33–212.17) for `conversation`, `dialogue`, `speaker`, and `portrait`. **No entry in any year explicitly documents adding NPC portraits/icons to the conversation UI or the "interact nearby" interaction menu.** The 207.31 "entire new UI" bullet lists "interact nearby" (matches the interaction-menu screenshot in `evidence/02/OBSOLETE.md`) but does **not** list a conversation/dialogue screen by name. So: unlike the inventory claim, the conversation-portrait obsolescence claim in `evidence/02/OBSOLETE.md` is **not directly backed by a quoted patch note** — it rests on the screenshot evidence alone. I closed this gap with decompiled code instead (below).

### 2. Obsolescence verdict: **PARTLY OBSOLETE** (splits by caller — do not treat TileMaker as one unit)

**Inventory caller (`Screens/QudUX_InventoryScreen.cs`, `TileMaker.cs` line 475):** Verified obsolete under one specific condition, not universally. `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs` (current source, unmodified on this branch) already reads:
```csharp
if (QudUXOptions.UI.UseQudUXInventory
    && !(GameOptions.ModernUI && GameOptions.ModernCharacterSheet)
    && XRLCore.Core.Game.Player.Body.GetConfusion() <= 0)
{
    __result = new XRL.UI.QudUX_InventoryScreen().Show(GO);
    return false;
}
```
The gate requires **both** `ModernUI` and `ModernCharacterSheet` true to skip QudUX's screen. That means if a player runs Modern UI with "Modern character sheet" turned **off** (the exact toggle 207.69 added), QudUX's inventory screen — and therefore `TileMaker`'s inventory caller — still runs. So this caller is obsolete only in the default modern-UI configuration, not universally obsolete.

**Conversation portrait caller (`Screen Extenders/ConversationUIExtender.cs`, `Harmony Patches/Patch_XRL_UI_ConversationUI.cs`):** I read the patch file in full — **every line is commented out** (`//`), and has been since commit `85f4fc0` "fix outdated Harmony patches" (2021-09-03, `git log --oneline -- "Harmony Patches/Patch_XRL_UI_ConversationUI.cs"`), years before this 1.0.5 branch existed (`git status` confirms the file is untouched on `fix/add-support-for-1.0.5`). **This patch currently does not run at all, in either UI mode, as shipped from this source tree.** `ConversationUIExtender.DrawConversationSpeakerTile` has no other caller anywhere in the codebase (only 3 files reference it: the extender itself, the commented patch, and a reflection helper in `Concepts/Constants.cs` that's only used from the commented code).

I decompiled `XRL.UI.ConversationUI` from the installed 1.0.5 `Assembly-CSharp.dll` to check whether the native game covers this gap. Key findings:
- `InternalConversation()` **unconditionally** computes a portrait for every conversation: `Icon = (Transmitter ?? Speaker)?.RenderForUI("Conversation");` — this runs regardless of UI mode.
- The legacy/console renderer, `RenderBox()` (called from `RenderClassic()`), fetches that same `Icon`, forwards it through a `RenderNodeEvent`, but **never draws it** — it only writes the title text: `SB.WriteAt(Left, Y1, "{{y|[ " + Title + " ]}}");`. No tile/icon is rendered on screen in the legacy path.
- The modern renderer path (`UIManager.UseNewPopups` → `Render()`) calls `Popup.ShowConversation(Title, Icon, CurrentNode.GetDisplayText(...), ...)` — `Icon` **is** consumed there, i.e. the modern UI's own popup draws the portrait graphically. This matches the screenshot evidence in `evidence/02/OBSOLETE.md`.

So: **modern UI genuinely does this natively now** (confirmed by decompiled code, not just screenshots) — FULLY OBSOLETE there. But **the legacy/console renderer still drops the icon on the floor** — the gap QudUX's feature targeted in 2021 still exists in 1.0.5. The feature concept is not obsolete for legacy UI; only the current implementation is dead.

Additionally, even if someone uncommented the old patch, it likely wouldn't reattach correctly in 1.0.5: its `TargetMethod()` still finds `ConversationUI.HaveConversation(Conversation, ...)` (verified this overload still exists), but its transpiler searches for an inline IL sequence — `Ldstr " ]}}"` immediately followed by a 4-argument `Callvirt` to `ScreenBuffer.Write` — directly inside `HaveConversation`. In 1.0.5 that title-drawing code has moved into the separate `RenderBox()` method and uses a 3-argument `SB.WriteAt(int, int, string)` call, not the old 4-arg `Write`. ⚠️ unverified: I confirmed this via decompiled C#, not by literally running the Harmony transpiler against 1.0.5 IL, but the method split and signature change make a match very unlikely — the patch's own fallback branch even logs "Failed... This patch may not be compatible with the current game version" for exactly this case.

### 3. Modern-UI interaction

Confirmed, per-caller:
- **Inventory:** behaves differently by design, already gated on native flags `XRL.UI.Options.ModernUI` and `XRL.UI.Options.ModernCharacterSheet` (both must be true to suppress QudUX). This is precisely the feature-flag pattern the user's overall plan calls for, and it's already implemented correctly for this caller.
- **Conversation portrait:** would behave differently by design if revived — native `Popup.ShowConversation` already draws the icon in modern UI (redundant to add QudUX's bracket-tile there), while the legacy `RenderBox()` never draws it (real gap). But right now it behaves identically in both modes: it does nothing, because the patch is commented out.

### 4. Recommendation

- **Inventory caller — keep as is.** `Harmony Patches/Patch_XRL_UI_InventoryScreen.cs` already gates correctly on `GameOptions.ModernUI && GameOptions.ModernCharacterSheet`; `Screens/QudUX_InventoryScreen.cs` and the `TileMaker` inventory caller (`TileMaker.cs` via `QudUX_InventoryScreen.cs:475`) still serve real users (legacy UI, or modern UI with "Modern character sheet" off). `TileMaker.cs`'s render pipeline was already independently fixed for 1.0.5's `Render` API change (`ComponentRender` + `FinalRender`, see `errors/02-render-property-not-method.md`), so no further code-health work is needed here.
- **Conversation portrait caller — do not silently "remove as obsolete."** It is dead code today (safe to delete with zero behavior change, since it already does nothing), but the feature it implements is **not** obsolete in legacy UI per the decompiled evidence above. Two honest options, both consistent with "legacy-UI users keep working features":
  1. **Gate + rewrite** (recommended if this feature is wanted back): retarget the transpiler at `XRL.UI.ConversationUI.RenderBox()`'s `SB.WriteAt(Left, Y1, "{{y|[ " + Title + " ]}}")` call, gate it to legacy UI only (e.g. `!XRL.UI.Options.ModernUI`, mirroring the inventory patch's pattern), and keep the existing `QudUX_OptionTileConversationUI` option (`Options.xml` line 14, "Show sprites in conversation screen text UI", default Yes — its own name already implies text/legacy UI). This is a non-trivial rewrite, not a one-line fix.
  2. **Remove**, if the user judges this feature not worth reviving: delete `ConversationUIExtender.cs`, `Patch_XRL_UI_ConversationUI.cs`, the `QudUX_OptionTileConversationUI` option, and the `ConversationUIExtender_DrawConversationSpeakerTile` reflection field in `Concepts/Constants.cs`. This is a legitimate "remove" per the user's own rule (dead in both UI modes today), but note it forecloses the legacy-UI gap rather than fixing it.
  - "Simplify by leaning on a native API" doesn't apply cleanly here: the native API (`RenderForUI("Conversation")` / `Icon`) already exists and is what QudUX would hook into for option 1 — there's no simpler native substitute for the *legacy* renderer, since the legacy renderer itself doesn't expose the icon.

### 5. Confidence + gaps

**Verified directly:**
- 207.31 and 207.69 wiki text, quoted verbatim from local `2024.wiki`.
- `Patch_XRL_UI_InventoryScreen.cs` gating logic (read current source).
- `Patch_XRL_UI_ConversationUI.cs` is fully commented out, and since which commit (`git log`).
- No other caller of `ConversationUIExtender`/`DrawConversationSpeakerTile` exists (grepped whole mod source).
- 1.0.5 `ConversationUI` decompiled via `ilspycmd`: `HaveConversation` overloads, `InternalConversation`, `RenderClassic`, `RenderBox`, and the `Render()`/`Popup.ShowConversation` modern path — confirms native Icon computation, legacy-path drop, and modern-path use.
- `TileMaker.cs`'s 1.0.5 render-pipeline fix already applied and verified in `errors/02-render-property-not-method.md`.

**Not verified / gaps:**
- Did not launch the game myself; relied on the user's own in-game screenshots (`image-10.png`, `image-11.png`) cited in `evidence/02/OBSOLETE.md` for the modern-UI portrait/icon visuals.
- Did not run the old Harmony transpiler against 1.0.5 IL to prove it fails to match — inferred from decompiled C# structure only (tagged ⚠️ unverified above).
- 2025/2026 wiki entries (`2025.wiki`, `index.wiki` builds 211.33–212.17) contain no conversation/inventory-portrait-relevant entries at all — confirmed absence by grep, not a gap.
- Did not use any web fetches; all findings sourced from the local wiki dumps, mod source, and decompiled 1.0.5 `Assembly-CSharp.dll`.
