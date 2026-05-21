---
name: dead-code-sweep
description: >-
  Run `ruff` (Python) and `eslint`/`ts-prune` (TypeScript) across the
  repo to remove unused imports, unused variables, and unused
  re-declarations. Also runs `vulture` (Python) and `ts-prune`
  (TypeScript) to find unreferenced files / exports — those are
  reported but NEVER auto-deleted because of false-positive risk
  (dynamic imports, FastAPI router auto-discovery, Next.js route
  files). Use when the user says "remove dead code", "dead code
  sweep", or "find unused imports".
---

# Dead-Code Sweep

The fast part of repo hygiene: ruff + eslint take 5 seconds, fix 200
findings, and shrink the diff for every future review. The slow part
(unreferenced files) is informational only.

## When to use this skill

- After a chat that deleted code and may have left imports orphaned.
- As the first step of `hygiene-all` (removed code must come out
  before file-size measurements are accurate).
- Whenever a build warns about unused imports.

## Workflow

```
Task progress:
- [ ] Verify tooling: ruff, eslint, ts-prune, vulture
- [ ] Phase 1: Python unused-imports/vars auto-fix (ruff --fix)
- [ ] Phase 2: TypeScript unused-imports auto-fix (eslint --fix)
- [ ] Phase 3: Python unreferenced-module sweep (vulture, report only)
- [ ] Phase 4: TypeScript unreferenced-export sweep (ts-prune, report only)
- [ ] Cross-check: any file deleted by Phases 1/2 that the build still references? (revert if so)
- [ ] Print report
```

### Phase 1 — Python unused imports / vars

```bash
ruff check backend/ \
  --select F401,F811,F841 \
  --fix \
  --exclude legenv,__pycache__,backend/db/migrations/versions \
  --output-format=json > /tmp/ruff-deadcode.json
```

Rule meanings:

- `F401`: unused import.
- `F811`: redefinition of unused name.
- `F841`: unused local variable.

Excluded paths:

- `legenv` — venv, vendored.
- `__pycache__` — generated.
- `backend/db/migrations/versions` — Alembic templates sometimes leave
  imports for symmetry across migrations; don't touch.

ruff's `--fix` is conservative — it only removes imports it's certain
are safe. If a file has a `__all__` declaration, ruff respects it.

### Phase 2 — TypeScript unused imports / vars

```bash
cd frontend && npx eslint . \
  --ext .ts,.tsx \
  --rule '{"@typescript-eslint/no-unused-vars": ["error", {"argsIgnorePattern": "^_", "varsIgnorePattern": "^_"}]}' \
  --fix \
  --no-error-on-unmatched-pattern \
  --output-format=json > /tmp/eslint-deadcode.json
```

The leading-underscore exemption matches the project's convention for
intentionally-unused destructured props (e.g. `const { _, x } = ...`).

If the project has a pre-existing ESLint config that already covers
`no-unused-vars`, prefer:

```bash
cd frontend && npx eslint . --fix --no-error-on-unmatched-pattern
```

and let project config dictate severity. Surface the count either way.

### Phase 3 — Python unreferenced modules (report only)

```bash
command -v vulture || pip install --quiet vulture
vulture backend/ \
  --min-confidence 80 \
  --exclude legenv,__pycache__,backend/db/migrations/versions \
  > /tmp/vulture.txt
```

Vulture at confidence 80 still flags false positives — primarily:

- FastAPI routers imported only for their side-effect of
  `@router.post(...)` registration.
- Pydantic models used as response_model annotations.
- Migrations imported by Alembic via filename, not symbol.

The skill reports every Vulture hit categorized by likely-false-positive
class, with a recommended verification command (`rg <symbol> backend/`).
NEVER auto-deletes files based on Vulture output.

### Phase 4 — TypeScript unreferenced exports (report only)

```bash
cd frontend && npx ts-prune \
  --project tsconfig.json \
  --skip 'frontend/components/ui|.next' \
  > /tmp/ts-prune.txt
```

ts-prune flags exports that no other file imports. False positives:

- Next.js route files (`page.tsx`, `layout.tsx`, `loading.tsx`,
  `error.tsx`) — discovered by the framework via filename, not import.
  Skill excludes these from the report.
- Components used only via `dynamic()` imports.
- Storybook stories (`.stories.tsx`) — referenced by Storybook
  configuration.

The skill reports the residue. NEVER auto-deletes.

### Cross-check — did Phases 1/2 break the build?

After Phases 1 + 2 auto-fix the working tree, run:

```bash
cd frontend && npx tsc --noEmit
python -c "import backend.api_server"
```

(or whatever the project's smoke-import is for backend).

If either fails, revert the auto-fix batch (`git checkout -- .` is the
fastest path) and surface the failure — Phase 1's overconfident import
removal hit something dynamic.

For safer ruff fixing, narrow to one file at a time via a loop:

```bash
for f in $(rg -l 'F401|F811|F841' --type py); do
  ruff check "$f" --select F401,F811,F841 --fix
  python -c "import importlib.util; ..."  # smoke import f
done
```

…and revert per-file on failure. Use this slow path only after the
batch path failed.

## Hard Rules (never break)

- **Never auto-delete a file.** Even if Vulture / ts-prune is 100%
  sure no other file imports it, the runtime may. Always report only.
- **Never modify `frontend/components/ui/`** — shadcn primitives.
- **Never modify Alembic migrations** — even cleanups risk drift with
  what's deployed.
- **Never disable an existing ESLint rule** to "fix" findings. The
  skill operates within the project's configured rule set.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging`.
- **Revert the batch on smoke-import failure.** Don't ship a working
  tree that doesn't import.

## Trigger Examples

- "remove dead code"
- "dead code sweep"
- "find unused imports"
- "clean up unused stuff"
- "run the dead-code-sweep skill"

## Output Template

End the run with:

```
Dead-code-sweep summary

Phase 1 — Python unused imports/vars
  - Findings:                  <N>
  - Auto-fixed by ruff:        <K>
  - Files touched:             <M>
  - Smoke import after:        pass | fail (reverted)

Phase 2 — TypeScript unused imports/vars
  - Findings:                  <N>
  - Auto-fixed by eslint:      <K>
  - Files touched:             <M>
  - tsc --noEmit after:        pass | fail (reverted)

Phase 3 — Python unreferenced modules (report only)
  - Vulture findings (conf 80): <N>
    • <file>:<line>  <symbol>  (likely-FP: <yes|no — class>)
    • ...

Phase 4 — TypeScript unreferenced exports (report only)
  - ts-prune findings:         <N>
    • <file>  <symbol>  (likely-FP: <yes|no — class>)
    • ...

Working tree dirty:            yes | no

Next step: review Phase 3 / Phase 4 findings manually, then run `review-and-ship-to-staging`.
```
