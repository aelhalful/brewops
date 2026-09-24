---
name: python-setup
description: How Python dependencies, the virtualenv, and running commands work in BrewOps — this project uses uv, not pip/venv/poetry directly. Use this whenever you're about to install a package, run a script or the app, set up a fresh environment, troubleshoot an import/module-not-found error, or the user mentions pip, requirements.txt, virtualenv, poetry, or "install X" in this repo. Also use it before suggesting `pip install`, `python -m venv`, or activating a venv by hand — this project has a faster, more consistent way.
---

# Python setup for BrewOps

This project is managed end-to-end by [uv](https://docs.astral.sh/uv/). uv replaces
pip, venv, and the manual "activate the virtualenv" step with one tool that reads
`pyproject.toml` and `uv.lock`. Prefer `uv run ...` over activating `.venv` yourself —
it resolves the right environment automatically, so there's no risk of running a
command against the wrong Python or a stale venv.

## Installing / syncing dependencies

```
uv sync
```

Creates `.venv/` if missing and installs exactly what `uv.lock` pins. Run this after
pulling changes that touched `pyproject.toml` or `uv.lock`, or when setting up the repo
for the first time.

## Running things

Always prefix commands with `uv run` instead of activating the venv first:

```
uv run start        # starts the FastAPI app on http://localhost:8123
uv run seed         # creates the DB and ingests sample CSVs
uv run ingest <path>
uv run pytest       # test suite
uv run scripts/setup_check.py
```

`uv run` picks up `.venv` automatically and installs anything missing before
running — no separate `pip install -r requirements.txt` step exists in this repo.

Manually activating `.venv` (`.venv\Scripts\Activate.ps1` on Windows,
`source .venv/bin/activate` elsewhere) still works and is fine for an interactive
shell session, but don't rely on it inside scripts or one-off commands — `uv run`
is the reliable default because it doesn't depend on which shell state is currently
active.

## Adding or removing a dependency

Don't hand-edit the `dependencies` list in `pyproject.toml` and don't `pip install`
directly — that leaves `uv.lock` out of sync with what's actually declared.

```
uv add <package>            # runtime dependency
uv add --dev <package>      # dev-only dependency (goes in [dependency-groups].dev)
uv remove <package>
```

Each of these updates `pyproject.toml` and `uv.lock` together and re-syncs `.venv`.
Commit both files in the same change — the lock file is what makes installs
reproducible for the next person (or CI), so a `pyproject.toml` edit without a
matching `uv.lock` update is a red flag in review.

## Python version

`pyproject.toml` pins `requires-python = ">=3.11"`. `uv run`/`uv sync` will fetch and
manage a matching interpreter for you if one isn't already on the system — no manual
`pyenv`/system Python juggling needed.
