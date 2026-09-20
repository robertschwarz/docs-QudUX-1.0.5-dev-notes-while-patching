# QudUX v2 — 1.0.5 patch companion docs

This repo is a companion to [egocarib/CavesOfQud-QudUX-v2 #32](https://github.com/egocarib/CavesOfQud-QudUX-v2/pull/32), which ports QudUX v2 to Caves of Qud 1.0.5.

Its purpose is to make the thought and implementation process transparent — for the mod's maintainer, or for any LLM picking up the work later. The PR itself contains only code; this repo contains the reasoning behind every change: how each build error was diagnosed, how the fixes were validated, what was tested in-game, and what was found along the way that isn't strictly part of the port.

The work was done with AI assistance (Claude Sonnet subagents, one per error, each in an isolated worktree), followed by a human UAT pass against a live 1.0.5 install.
