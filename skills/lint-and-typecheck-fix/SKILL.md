---
name: lint-and-typecheck-fix
description: >-
  Repo-wide lint and type-check pass with auto-fix. Runs `ruff check
  backend/ --fix`, `mypy backend/`, `cd frontend && npm run lint --
  --fix`, and `npx tsc --noEmit`. Auto-fixes everything the tools can
  fix; reports the rest as a categorized list. Mirrors the per-file
  `ReadLints` check at line 53 of `review-and-ship-to-staging`, but
  expanded to the entire repo on demand. Use when the user says
  "fix all lints", "run lint and typecheck", "lint and typecheck fix".
---

# Lint and Typecheck Fix

The "make the build green" hygiene step. Auto-fixes everything ruff
and eslint can auto-fix, then surfaces what mypy and tsc still don't
like.

This skill is the second step of `hygiene-all` — it runs AFTER
`dead-code-sweep` so removed imports don't trigger spurious diagnostics
on lines that just got deleted.

## When to use this skill

- After any chat that touched many files and may have introduced
  formatting / import-order / type-narrowing regressions.
- As the second step of `hygiene-all`.
- When the user wants a "clean build" baseline.

## Workflow

```
Task progress:
- [ ] Verify tooling: ruff, mypy, npm, npx
- [ ] Phase 1: ruff (lint + format) --fix
- [ ] Phase 2: mypy (report only — never auto-fix)
- [ ] Phase 3: eslint --fix
- [ ] Phase 4: tsc --noEmit (report only — never auto-fix)
- [ ] Cross-check: does the project import after Phase 1 + 3?
- [ ] Print categorized report
```

### Phase 1 — ruff

```bash
ruff check backend/ \
  --fix \
  --exclude legenv,__pycache__,backend/db/migrations/versions \
  --output-format=json > /tmp/ruff-lint.json

ruff format backend/ \
  --exclude legenv,__pycache__,backend/db/migrations/versions
```

The project may have a `pyproject.toml` / `ruff.toml` already
configured — ruff picks it up automatically. The skill does NOT pass
`--select` (would override project config). It passes `--exclude` only
to ensure the vendor/migration paths are always skipped even if the
config drifts.

`ruff format` runs after `ruff check --fix` because:
- `check --fix` may move imports / add parentheses.
- `format` then settles whitespace consistently.

Together they replace black + isort + flake8 in a single tool.

### Phase 2 — mypy (report only)

```bash
command -v mypy || pip install --quiet mypy
mypy backend/ \
  --ignore-missing-imports \
  --no-error-summary \
  --show-error-codes \
  --pretty \
  > /tmp/mypy.txt 2>&1
```

`--ignore-missing-imports` is set because the project doesn't ship
stubs for every third-party dep (PocketFlow, internal scripts, etc.).
If the project has a `mypy.ini` / `pyproject.toml` with mypy config,
that wins.

mypy errors are NEVER auto-fixed. The skill categorizes them:

- `[no-any-return]`: function annotated `-> T` but returns `Any`.
- `[arg-type]`: argument type mismatch.
- `[union-attr]`: accessing attr on Optional without narrowing.
- `[assignment]`: assignment incompatibility.
- `[return-value]`: returning wrong type.

For each category, the report includes top-3 offending files so the
user can attack them in batches.

### Phase 3 — eslint

```bash
cd frontend && npx eslint . \
  --ext .ts,.tsx \
  --fix \
  --no-error-on-unmatched-pattern \
  --format json \
  > /tmp/eslint.json 2>&1 || true
```

`|| true` because eslint's exit code reflects unfixed errors, which is
expected — the skill parses the JSON to decide pass/fail.

Project ESLint config in `[frontend/.eslintrc](frontend/)` (or
`eslint.config.mjs`) is the source of truth. The skill does not
override rules.

If the project also has Prettier configured (`.prettierrc` or
similar), run it after eslint so prettier's formatting wins over
eslint's where they disagree:

```bash
cd frontend && [ -f .prettierrc ] && npx prettier --write . --log-level=warn || true
```

### Phase 4 — tsc --noEmit (report only)

```bash
cd frontend && npx tsc --noEmit --pretty false > /tmp/tsc.txt 2>&1 || true
```

`--pretty false` makes the output easier to grep + categorize.

Categorize by error code:

- `TS2322`: Type not assignable.
- `TS2345`: Argument not assignable.
- `TS2532`: Object possibly undefined.
- `TS18046`: Element implicitly has 'any' type.
- `TS2339`: Property does not exist on type.

Same as mypy: the skill reports top-3 offending files per category.

### Cross-check — does the project still import / build?

After Phases 1 + 3 auto-fix the working tree:

```bash
python -c "import sys; sys.path.insert(0, 'backend'); import api_server"  # backend smoke
cd frontend && npx tsc --noEmit  # frontend smoke (re-run, expected to mirror Phase 4)
```

If the backend smoke import fails (e.g. ruff's auto-fix broke an
import), revert ruff's changes (`git checkout -- backend/`) and
report. eslint's `--fix` is conservative enough that this is rarely
needed for the frontend.

## Hard Rules (never break)

- **Never auto-fix mypy or tsc errors.** Type-fix decisions
  (`# type: ignore`, narrowing with `assert`, `cast`, refactoring the
  function signature) carry real meaning. Always report only.
- **Never override project lint configuration** by passing `--rule`,
  `--select`, or `--no-config` flags. The project's `pyproject.toml`
  / `eslint.config.*` are authoritative.
- **Never touch `frontend/components/ui/`** — shadcn primitives.
- **Never touch `backend/db/migrations/versions/`** — migrations are
  intentionally formatted/imported atypically.
- **Revert ruff's batch on smoke-import failure.** Don't ship a
  working tree that doesn't import.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging`.

## Trigger Examples

- "fix all lints"
- "run lint and typecheck"
- "lint and typecheck fix"
- "clean build please"
- "run the lint-and-typecheck-fix skill"

## Output Template

End the run with:

```
Lint-and-typecheck-fix summary

Phase 1 — ruff check + format
  - Findings before:           <N>
  - Auto-fixed:                <K>
  - Unfixed (report):          <N-K>
  - Files reformatted:         <M>
  - Smoke import after:        pass | fail (reverted)

Phase 2 — mypy (report only)
  - Total errors:              <N>
  - By code:
    • no-any-return:           <c>  top: <file1>, <file2>, <file3>
    • arg-type:                <c>
    • union-attr:              <c>
    • assignment:              <c>
    • return-value:            <c>
    • other:                   <c>

Phase 3 — eslint --fix
  - Findings before:           <N>
  - Auto-fixed:                <K>
  - Unfixed (report):          <N-K>
  - Files touched:             <M>
  - Prettier run:              yes | no

Phase 4 — tsc --noEmit (report only)
  - Total errors:              <N>
  - By code:
    • TS2322 (not assignable): <c>  top: <file1>, <file2>, <file3>
    • TS2345 (arg not assign): <c>
    • TS2532 (poss undefined): <c>
    • TS18046 (implicit any):  <c>
    • TS2339 (no property):    <c>
    • other:                   <c>

Working tree dirty:            yes | no

Next step: triage the mypy / tsc residue, then run `review-and-ship-to-staging`.
```
