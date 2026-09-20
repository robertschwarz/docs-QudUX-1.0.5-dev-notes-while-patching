# CLAUDE.md

This is a **public** documentation repo. No personal identifiers, local paths, or credentials belong here.

## Context

Post-mortem and UAT documentation for a QudUX v2 → Caves of Qud 1.0.5 compatibility review.
The code fixes themselves live in the mod repo on branch `fix/add-support-for-1.0.5`.

## Rules for agents working here

- **Public repo.** Never write absolute local paths, personal names, machine identifiers, or credentials into any file.
- Use `%USERPROFILE%` / `$env:USERPROFILE` for game data paths; use repo-relative paths for files within this repo.
- Do not commit game log files (`Player*.log`).

## Commit style

Use [Conventional Commits](https://www.conventionalcommits.org/) prefixes. Title only — no body, no footer.

| Prefix | Use for |
|--------|---------|
| `fix:` | correcting wrong information, broken links, bad data |
| `docs:` | new or updated documentation content |
| `refactor:` | restructuring without changing meaning |
| `chore:` | tooling, config, gitignore, CLAUDE.md itself |
| `style:` | formatting only, no content change |

Examples:
```
fix: replace absolute paths with relative equivalents
docs: add README
chore: add gitignore
```
