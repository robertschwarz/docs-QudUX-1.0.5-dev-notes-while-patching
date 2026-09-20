# QudUX v2 — 1.0.5 Compatibility Notes

External review and patch documentation for bringing [QudUX v2](https://github.com/wildwinter/QudUX-v2) up to Caves of Qud 1.0.5.

**Branch under review:** `fix/add-support-for-1.0.5`  
**Review date:** 2026-09-19–20

---

## What's here

| Path | Contents |
|------|----------|
| `1.0.5-compat-errors.md` | All 12 build errors, root-cause analysis, and fixes |
| `1.0.5-agent-results.md` | Outcome summary from the multi-agent fix run; per-issue verdicts and edge-case findings |
| `errors/` | One file per error — decompiled API evidence, fix rationale, and agent review |
| `uat/` | UAT scripts (one per feature), setup guide, and human test results |
| `uat/human-uat-stage-1/` | Screenshots from the human UAT run |
| `uat/evidence/` | Obsolescence records for features superseded by the native modern UI |

## How the patch was produced

1. Static analysis of the 1.0.5 `Assembly-CSharp.dll` (decompiled with `ilspycmd`) identified 12 build errors.
2. Twelve Claude Sonnet subagents ran in parallel — one per error — each in an isolated git worktree. An orchestrator merged the deltas, rebuilt, and verified 0 compiler errors.
3. Human UAT covered all 12 features against a live 1.0.5 game install.
4. A second agent pass cross-referenced UAT findings against Caves of Qud patch notes (2023–2026) to flag features superseded by the native modern UI.

## Key findings

- **10 of 12 source files changed.** Build goes from 1 error → 0 errors.
- **3 fixes were wrong on first pass** and were corrected before UAT (details in `1.0.5-agent-results.md`).
- **4 features are partly or fully obsolete** now that the game has a native modern UI — see `uat/evidence/` and the patch-note sections in each UAT script.
- **Several latent bugs** unrelated to the 1.0.5 port were found during UAT; documented in `uat/00-setup.md` § Cross-cutting findings.
