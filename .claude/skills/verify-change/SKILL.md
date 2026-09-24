---
name: verify-change
description: Verify a change to BrewOps and report the evidence. Do not claim success without showing command output.
---

# Verify The Change

1. If `brewops.db` is missing, run `uv run seed`.
2. Run the tests: `uv run pytest`. All must pass.
3. If the change relates to Lab A, run `uv run scripts/lab_a_check.py` and require `[DONE]`.
4. Start the app with `uv run start`, confirm http://localhost:8123 loads and `/api/stats` responds, then stop it.
5. Report each command you ran and its result. If anything failed, say so plainly.


## Gotchas

- The app runs on port **8123**, not 8000/3000/5000.
- Run `uv run seed` before the app, it won't start without the database.
- CLI output is ASCII-only (`[OK]`/`[FAIL]`), so it works in Windows terminals, keep it that way.
- "Looks done" is not "verified." Always show the actual command output.

## Version

0.1.0