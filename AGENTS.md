# Agent Operating Instructions

This repository maintains the portable `memgraph` memory kit and its canonical
project-memory doctrine. Keep instructions universal for Claude, Codex,
macOS/Linux, and Windows.

## Startup Discipline

Before non-trivial work:

1. Run `git status --short --branch`.
2. If `.agent/memory.db` exists, run `memgraph/memgraph session-context` on
   macOS/Linux or `powershell.exe -NoProfile -ExecutionPolicy Bypass -File
   memgraph/memgraph.ps1 session-context` on Windows.
3. Read `project-memory-structure-instruction.md`, then the relevant files under
   `memgraph/`.

If `.agent/memory.db` does not exist, the memory skill is idle for this repo.

## Source Hierarchy

1. User's latest explicit instruction.
2. `project-memory-structure-instruction.md`.
3. `memgraph/SKILL.md` and `memgraph/install/README.md`.
4. `memgraph/scripts/*`, `memgraph/queries/*`, `memgraph/references/*`.
5. This file.

## Portability Rules

- Use the wrapper only: `memgraph/memgraph` on macOS/Linux,
  `powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1`
  on Windows. `pwsh` is optional when PowerShell 7 is installed.
- Do not call `memgraph/scripts/*.py` directly except documented bootstrap via
  the venv python.
- Keep Claude integration in `.claude/settings.json`; keep Codex startup rules
  in `AGENTS.md`; keep `CLAUDE.md` as `@AGENTS.md`.
- Do not commit per-machine assets: `.env`, `memgraph/venv/`, `memgraph/vendor/`,
  archives, backups, Python caches, or local `.agent/` scratch state.

## Git Discipline

Stage only intentional source files, verify before committing, and keep generated
archives/backups out of the repository.
