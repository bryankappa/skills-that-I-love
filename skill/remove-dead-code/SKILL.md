---
name: remove-dead-code
description: Find and remove dead Python code using ruff and vulture. Use when cleaning up unused imports, variables, functions, classes, or unreachable code in a Python project.
---

# Remove Dead Code

Clean up dead Python code with two complementary tools:

- **ruff** — fast linter that catches unused imports, unused variables, unreachable code, and redundant syntax. Can auto-fix most of it.
- **vulture** — static analyzer that finds unused functions, classes, methods, attributes, and imports that ruff misses because it only looks within a single module.

Use both. Ruff is precise but narrow; vulture is broader but noisier (it guesses, so it produces false positives).

## Step 1 — Orient

Before touching anything, confirm:

1. The project has a clean working tree (`git status`). If not, stop and ask the user.
2. Tests exist and currently pass. Dead-code removal is only safe if you can verify you did not break anything.
3. You know the Python source roots (commonly `src/`, `app/`, or the package directory).

If any of these fail, surface the problem to the user before proceeding.

## Step 2 — Run ruff

Install if needed (`pip install ruff` or `uv pip install ruff`), then:

```bash
# Show what would change without writing.
ruff check --select F,E,W,UP,SIM --preview <path>

# Auto-fix the safe subset.
ruff check --select F401,F841,F811,F601,F602 --fix <path>
```

Key rule codes:

- `F401` — unused import
- `F811` — redefinition of unused name
- `F841` — unused local variable
- `E501` — line too long (not dead, skip unless asked)
- `SIM` — simplifications (often reveal dead branches)

Review the diff after each auto-fix pass. Do not blanket-apply `--unsafe-fixes`.

## Step 3 — Run vulture

Install (`pip install vulture`), then:

```bash
# Default confidence, quick scan.
vulture <path>

# High-confidence only — fewer false positives, good first pass.
vulture --min-confidence 80 <path>

# Include test files in the scan so tests that reference "unused" symbols keep them alive.
vulture <path> tests/
```

Vulture output lists candidates with a confidence score. Treat everything under 80% as a suggestion, not a verdict.

### Handling false positives

Vulture cannot see:

- Dynamic dispatch (`getattr`, `globals()[...]`, string-based routing).
- Framework-registered symbols (FastAPI routes, Django views, pytest fixtures, Click commands, Pydantic validators).
- Public API surface exported from `__init__.py`.

For these, add a whitelist file rather than deleting:

```bash
vulture <path> --make-whitelist > vulture_whitelist.py
```

Then commit the whitelist and re-run vulture with it included:

```bash
vulture <path> vulture_whitelist.py
```

## Step 4 — Delete carefully

For each candidate:

1. `rg` for the symbol across the whole repo, including strings and comments.
2. Check if it is re-exported from any `__init__.py`.
3. Check if it is referenced in config files, templates, or YAML.
4. If clean, delete. If ambiguous, leave it and note it in the whitelist.

Never delete in bulk. Work in small commits so `git bisect` remains useful if something breaks later.

## Step 5 — Verify

After each deletion pass:

1. `ruff check <path>` — no new errors.
2. Run the project's test suite.
3. Run the project's type checker if one is configured (mypy, pyright).
4. Commit.

If tests fail, `git reset` the deletion, figure out why the symbol was reachable, and update your understanding before trying again.

## What not to do

- Do not delete code just because vulture flagged it — vulture is a hint, not a proof.
- Do not run `--unsafe-fixes` without reading each change.
- Do not remove public API symbols without confirming no downstream consumer exists.
- Do not combine dead-code removal with refactoring in the same commit — keep the diff reviewable.
