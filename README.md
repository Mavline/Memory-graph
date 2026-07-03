# Memory Graph

Portable repo-local memory kit for Claude, Codex, macOS/Linux, and Windows.

The kit stores durable project memory in `.agent/memory.db` (SQLite + FTS5 +
sqlite-vec) and exposes it through the `memgraph/` wrapper. The wrapper supports
startup context, hybrid recall, and typed write helpers for decisions, claims,
workflow tasks, runs, policies, review gates, entities, and relations.

## Contents

- `project-memory-structure-instruction.md` — canonical doctrine for project
  memory, strategic documents, `AGENTS.md`, `CLAUDE.md`, git, and startup flow.
- `memgraph/` — portable skill/CLI kit with wrappers, schema, scripts, queries,
  references, and install instructions.
- `graphify-vs-memgraph-memory-analysis.md` — comparison of this repo-local
  memory model with Graphify-style inferred code graphs.

## Quick Start

Place `memgraph/` at the root of the target project.

macOS/Linux:

```bash
cd memgraph
python3 -m venv venv
./venv/bin/python -m pip install --upgrade pip
./venv/bin/python -m pip install -r requirements.txt
mkdir -p vendor
case "$(uname -s)" in
  Darwin) cp "$(./venv/bin/python -c 'import sqlite_vec; print(sqlite_vec.loadable_path())')" vendor/vec0.dylib ;;
  Linux) cp "$(./venv/bin/python -c 'import sqlite_vec; print(sqlite_vec.loadable_path())')" vendor/vec0.so ;;
  *) echo "Unsupported Unix platform"; exit 1 ;;
esac
./venv/bin/python scripts/bootstrap.py --target ..
cd ..
memgraph/memgraph session-context
```

Windows PowerShell:

```powershell
cd memgraph
python -m venv venv
venv\Scripts\python.exe -m pip install --upgrade pip
venv\Scripts\python.exe -m pip install -r requirements.txt
New-Item -ItemType Directory -Force vendor | Out-Null
$base = & venv\Scripts\python.exe -c "import sqlite_vec; print(sqlite_vec.loadable_path())"
$src = if (Test-Path "$base") { "$base" } elseif (Test-Path "$base.dll") { "$base.dll" } else { throw "vec0 not found near $base" }
Copy-Item $src vendor\vec0.dll
venv\Scripts\python.exe scripts\bootstrap.py --target ..
cd ..
pwsh memgraph/memgraph.ps1 session-context
```

For full setup, including Claude SessionStart and Codex `AGENTS.md` startup
rules, read `memgraph/install/README.md`.
