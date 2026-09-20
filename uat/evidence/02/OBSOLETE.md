# UAT-02: Feature fully obsolete — skipped

**Date:** 2026-09-19  
**Verdict:** SKIP — both TileMaker callers superseded by native game UI

## Finding

TileMaker's conversation portrait caller (`ConversationUIExtender.DrawConversationSpeakerTile`) is obsolete. The native game already renders full graphical NPC portraits — a large sprite above the NPC's name — in both the interaction menu (look/attack/chat action picker) and the look/conversation screen. QudUX's feature added only a small `[tile]` bracket in the text title bar, which is entirely superseded.

**Evidence:** in-game observation 2026-09-19 (UAT test session):
- Interaction menu with Mehmet: full-size graphical portrait rendered natively above NPC name
- Look screen / conversation screen: full portrait + description, native rendering, no mod involvement
- Screenshots: `image-10.png`, `image-11.png` (desktop/qudux-uat)

The exact build this was introduced is not explicitly documented in the 2023–2024 wiki version history, but the visual evidence is conclusive.

## Combined obsolescence — TileMaker has no remaining active callers

| Caller | File | Status | Reason |
|--------|------|--------|--------|
| Inventory sprites | `Screens/QudUX_InventoryScreen.cs` | OBSOLETE | QudUX inventory screen superseded by native graphical inventory (build 207.31, May 2024) — see `evidence/09/OBSOLETE.md` |
| Conversation portrait | `Screen Extenders/ConversationUIExtender.cs` | OBSOLETE | Native game renders full NPC portraits natively |

TileMaker itself (`Utilities/TileMaker.cs`) has no remaining callers that serve a purpose. UAT-02 has nothing left to test.

## Impact

| UAT | Status |
|-----|--------|
| 02 (TileMaker — all steps) | SKIP — fully obsolete |
