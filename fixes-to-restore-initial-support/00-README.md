# Branch 1 — restore working QudUX v2 on Caves of Qud 1.0.5

**Read this file first.** It is the shared context for every step in this folder. Each `NN-*.md` is one self-contained step for one agent.

## What this is

QudUX v2 is a C# quality-of-life mod for Caves of Qud. The repo is at [egocarib/CavesOfQud-QudUX-v2](https://github.com/egocarib/CavesOfQud-QudUX-v2), on branch `fix/add-support-for-1.0.5`.

Twelve commits already made the mod **compile** against game 1.0.5. Then in-game UAT found that several features are broken at runtime and one bug hard-locks the game. These steps fix that. They do not redesign anything.

## Where this sits in the PR stack

| Branch | Scope |
|---|---|
| **1 — `fix/add-support-for-1.0.5` (you are here)** | Fix what exists. Bugs only. |
| 2 — stacked on 1 | Remove features the base game has made obsolete. |
| 3+ — stacked on 2 | Modern-UI support. Not designed yet — **do not anticipate it.** |

Several QudUX features are obsolete under the game's modern UI but still work under the legacy UI. In branch 1 you **fix them if broken and leave them in place**. Removal is branch 2's job. Steps 15–17 handle dead code and documentation.

## Golden rules

1. **Never commit or push.** Make your edits and leave them in the working tree. The orchestrator collects and stages them.
2. **Minimal change.** Fix the named bug. Do not refactor surrounding code, rename things, restyle, or "improve" anything you pass on the way.
3. **Never assert game behavior you have not verified** against the decompiled 1.0.5 assembly. Every wrong conclusion in this project so far came from assuming. If you cannot verify something, say so and mark it `⚠️ unverified`.
4. **Preserve file encoding.** Most files are UTF-8 with BOM and CRLF line endings. Do not let an editor or `sed` rewrite them — a whole-file line-ending flip buries your real change in the diff.
5. **Stay in your lane.** Your step names the files you own. Another agent owns the rest; touching their files creates merge conflicts.

## Build

```bash
cd "<mod-repo-root>"
dotnet build Mods.csproj -nologo -v q 2>&1 | grep -E "error|Error\(s\)"
```

The branch currently builds with **0 errors** and 49 pre-existing warnings. It must still build with 0 errors when you are done. Do not try to fix the pre-existing warnings.

`Mods.csproj` is gitignored. If you work in a git worktree, copy it in from the main checkout first, or the build will not run.

## Decompiling the game (your source of truth)

Game assemblies: `<steam-library>/steamapps/common/Caves of Qud/CoQ_Data/Managed/`

```bash
ilspycmd -t <Full.Type.Name> \
  "<steam-library>/steamapps/common/Caves of Qud/CoQ_Data/Managed/Assembly-CSharp.dll" \
  -r "<steam-library>/steamapps/common/Caves of Qud/CoQ_Data/Managed"
```

Use `-t` for one type at a time; decompiling the whole assembly is slow. `-l c` lists class names — pipe it through `grep` to find a type. If `ilspycmd` is missing, install it to a local tool path; the current version pins to `9.1.0.7988` because newer builds fail on .NET SDK 9:

```bash
dotnet tool install ilspycmd --version 9.1.0.7988 --tool-path <dir> --add-source https://api.nuget.org/v3/index.json
```

## Confirmed root causes

Established already. Do not re-derive these.

| # | Finding |
|---|---|
| 1 | **Hard freeze.** Eight legacy screens block on `Keyboard.getvk(...)`, which waits on a key queue that `GameManager` stops feeding whenever a modern-UI window is visible (`UIManager.instance.PassthroughOnTop()` returns false). There is no timeout, so the screen hangs until the game is killed. Vanilla's own legacy screens share this defect. **`Keyboard.getvk(map, pump, wait: false)` returns `Keys.None` instead of blocking** — that is what makes a mod-side fix possible. |
| 2 | **Silent confirmations.** Under the modern UI (`UIManager.UseNewPopups => Options.ModernUI`), `Popup.ShowYesNo` returns its `defaultResult` immediately without rendering. Six call sites omit `defaultResult`, so it falls back to `Yes` — including "Remove ALL of your auto-pickup exclusions?". |
| 3 | **Quick pickup drops shields and tools.** `BuildPopup` collects five categories but drains only three into `orderedGameObjects`, then overwrites `selection` with it. |
| 4 | **Legendary batch marking finds nothing.** It filters `GetPointsOfInterestEvent.GetFor(The.Player)`, whose `StandardChecks` exclude anything hostile to the player. Most legendary creatures are hostile. |

**Not a confirmed cause:** the dead quest-giver locator. Static analysis eliminated every candidate — the Harmony patch does bind (one `Check` overload, parameters matched by name), `ConversationUI` still calls it, `ApplyRegistrar` still runs the registration, and the node ID and property the mod looks for both still exist. Step 07 diagnoses it at runtime instead of guessing.

## Lanes

Lanes run in parallel. Steps inside a lane share files, so they run in order.

| Lane | Owns | Steps |
|---|---|---|
| A | `Screens/`, `Utilities/InputsUtilities.cs`, `Parts and Effects/QudUX_AutogetHelper.cs` | 01 → 02 → 03 → 04, and 05, 06 after 02 |
| B | `Parts and Effects/QudUX_ConversationHelper.cs`, `Harmony Patches/Patch_XRL_World_BeginConversationEvent.cs` | 07 → 08 |
| C | `Parts and Effects/QudUX_LegendaryInteractionListener.cs` | 09 → 10, and 11 |
| D | `Parts and Effects/QudUX_QuickPickupPart.cs`, `AutoAct/PickupSelection.cs` | 12, 13, then 14 |
| E | dead files, `Options.xml`, `Concepts/Options.cs`, docs | 15, 17 anytime; 16 only after A–D are merged |

## Step index

| Step | File | Lane | Depends on |
|---|---|---|---|
| 01 | `01-input-helper.md` | A | — |
| 02 | `02-screens-adopt-helper.md` | A | 01 |
| 03 | `03-popup-defaults.md` | A | 02 |
| 04 | `04-autoget-confirm-loop.md` | A | 03 |
| 05 | `05-gamestats-empty-list.md` | A | 02 |
| 06 | `06-question-key-layout.md` | A | 02 |
| 07 | `07-questgiver-diagnosis.md` | B | — |
| 08 | `08-questgiver-action-key.md` | B | 07 |
| 09 | `09-legendary-hostile-scan.md` | C | — |
| 10 | `10-legendary-idempotency.md` | C | 09 |
| 11 | `11-journal-write-investigation.md` | C | — |
| 12 | `12-quickpickup-shields-tools.md` | D | — |
| 13 | `13-quickpickup-ground.md` | D | — |
| 14 | `14-quickpickup-dead-debug.md` | D | 12, 13 |
| 15 | `15-delete-dead-code.md` | E | — |
| 16 | `16-obsolete-comments.md` | E | A–D merged |
| 17 | `17-obsolete-inventory-doc.md` | E | 15 |

Steps 07, 11 and 13 are **diagnosis** steps. Ending one with a precise, evidence-backed cause and no code change is a valid outcome. Guessing at a fix is not.

## Hand-back format

End your run with, at most, this:

- Verdict: FIXED / FIXED WITH CAVEATS / NOT FIXED (diagnosed) / BLOCKED
- Files changed, with a one-line description each
- Build result
- The evidence that convinced you — decompiled signature, log line, or code path
- Anything you deliberately left alone, and why
- Open questions or `⚠️ unverified` items

## Verifying in-game

Install: `./move-mod.ps1` copies the mod to `%USERPROFILE%/AppData/LocalLow/Freehold Games/CavesOfQud/Mods/QudUX_v2`.

**It deletes and recreates that folder**, which wipes `QudUX_AutogetSettings.json` — the player's auto-pickup exclusions and the "shown once" popup flag. Back that file up before reinstalling if your step touches auto-pickup.

Game log (mod exceptions land here): `%USERPROFILE%/AppData/LocalLow/Freehold Games/CavesOfQud/Player.log`. It is overwritten on every launch; copy it before relaunching.

Existing manual test scripts live in `../uat/`. Each step names the one that re-validates it. The research behind the obsolescence calls is in `../uat/00-setup.md` §10 and each `../uat/NN-*.md` "Patch-note correlation" section — read, don't redo.
