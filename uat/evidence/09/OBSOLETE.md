# UAT-09 / UAT-02: Feature obsolete — skipped

**Date:** 2026-09-19  
**Verdict:** SKIP — feature superseded by native game UI

## Finding

The QudUX revamped inventory screen (`QudUX_InventoryScreen`) and its sprite renderer (`TileMaker`, inventory caller) are obsolete. The native game UI added a full graphical inventory/equipment screen in **build 207.31 (Spring Molting beta, May 10 2024)** — after the mod's last update (~2022–2023). The `ModernUI > Modern character sheet` split option was added in **build 207.69 (June 14 2024)**, which is exactly the option the UAT-09 patch checks.

**Source:** https://wiki.cavesofqud.com/wiki/Version_history/2024

Key changelog entries:
- **207.31 (2024-05-10):** "an entire new UI" including redesigned Inventory & Equipment screen with full keyboard/mouse/controller support
- **207.69 (2024-06-14):** added `Modern UI > Modern character sheet` toggle — the same option pair `Patch_XRL_UI_InventoryScreen.cs` checks to suppress the QudUX screen

## Evidence observed in-game (UAT test #2, 2026-09-19)

- Native equipment screen already shows item sprites and category icons without the mod
- Items dropped on the ground render with correct sprites natively
- No visual gap that QudUX inventory was filling

## Impact

| UAT | Status | Reason |
|-----|--------|--------|
| 09 (inventory UI mode matrix) | SKIP | QudUX inventory screen replaced by native modern UI |
| 02 (TileMaker — inventory sprites) | SKIP (inventory steps) | TileMaker inventory caller never runs when native UI active |
| 02 (TileMaker — conversation portrait) | still valid | Conversation portrait is a separate TileMaker caller, unaffected |

## Recommendation

Remove `QudUX_InventoryScreen`, `Patch_XRL_UI_InventoryScreen`, `QudUX_OptionUseInventoryMenu`, and the inventory caller in `TileMaker`. Retain TileMaker only for the conversation portrait path.
