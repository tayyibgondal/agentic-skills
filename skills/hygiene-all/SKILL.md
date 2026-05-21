---
name: hygiene-all
description: >-
  Orchestrator that runs every repo-hygiene skill in order:
  `dead-code-sweep` → `lint-and-typecheck-fix` → `file-size-enforcer`.
  Aggregates each child's output into ONE combined report. The order
  matters: dead code is removed before line counts are taken;
  lint/typecheck runs after dead-code removal so deleted imports
  don't trigger spurious diagnostics; file-size enforcement runs last
  because the previous two steps may shrink files below the cap on
  their own. Use when the user says "run all hygiene", "hygiene
  all", "full repo hygiene pass", or "clean up the repo".
---

# Hygiene All (orchestrator)

The Friday-cleanup orchestrator. Daily lint / format work, plus the
periodic file-size pass, in one prompt.

## When to use this skill

- Weekly cleanup.
- After a chat that deleted code and may have left orphan imports
  + oversized files.
- After running `audit-all` (the audit pass may have moved code
  around; hygiene smooths the resulting style).

## Workflow

```
Task progress:
- [ ] Pre-flight: snapshot HEAD SHA + working-tree state
- [ ] Step 1: dead-code-sweep        — remove unused before measuring
- [ ] Step 2: lint-and-typecheck-fix — auto-fix what tools can fix
- [ ] Step 3: file-size-enforcer     — measure / propose / split
- [ ] Build the aggregated report
- [ ] Print the report + final hand-off line
```

### Why this exact order

1. **`dead-code-sweep` first** — removes unused imports and unused
   variables. This matters because:
   - file-size measurements at Step 3 will be wrong if you measure
     before deleting dead code (you'd needlessly split files that
     would shrink under the cap on their own).
   - Step 2's lint pass would otherwise flag dead imports as
     diagnostics; removing them first cleans the diagnostic noise.
2. **`lint-and-typecheck-fix` second** — auto-fixes everything ruff
   and eslint can auto-fix (format, import-order, prettier). Runs
   AFTER dead-code removal so the lint pass doesn't operate on lines
   that just got deleted.
3. **`file-size-enforcer` last** — the previous two steps may have
   shrunk some files below the 700-line cap on their own. Running
   this last ensures we don't propose splits for files that no
   longer need them.

### Execution model

Same as `audit-all`: read each child SKILL.md, execute its workflow,
capture its Output Template, build the aggregated report.

### Failure policy

`hygiene-all` is **fully soft-fail**. Each step independently reports
issues and continues. None of the children hard-stop this orchestrator
because hygiene findings are by definition non-blocking — they
represent debt, not bugs.

Specifically:

- If `dead-code-sweep`'s smoke-import fails (Phase 1's ruff fix broke
  something), it reverts its own batch and reports — the orchestrator
  continues to Step 2 on the unreverted baseline.
- If `lint-and-typecheck-fix`'s mypy or tsc report shows residue, the
  residue is included in the aggregated report; Step 3 still runs.
- If `file-size-enforcer` writes TODO comment blocks for files > 1000
  lines, those are aggregated as items — they aren't blockers.

### Aggregated report shape

```
hygiene-all summary

Pre-flight
- Pre-run HEAD:      <sha>
- Working tree:      clean | dirty (N files)

Step 1 — dead-code-sweep
  - Python  F401/F811/F841 auto-fixed:  <N>
  - TS unused-vars auto-fixed:          <N>
  - Vulture findings (report only):     <N>
  - ts-prune findings (report only):    <N>
  - Smoke import after:                 pass | fail (reverted)

Step 2 — lint-and-typecheck-fix
  - ruff lint findings auto-fixed:      <N>  (residue: <K>)
  - ruff format files touched:          <N>
  - eslint findings auto-fixed:         <N>  (residue: <K>)
  - mypy residue:                       <N>  top codes: <list>
  - tsc residue:                        <N>  top codes: <list>

Step 3 — file-size-enforcer
  - Files > 700 lines:                  <N>
    - 701–1000 (auto-split):            <K>
    - 1001+ (TODO comment written):     <M>

Files modified by this run              : <N>
  - by dead-code-sweep:                 <list>
  - by lint-and-typecheck-fix:          <list>
  - by file-size-enforcer:              <list>

Next step:
  - Review the mypy / tsc residue from Step 2 (those can't be auto-fixed).
  - Run `review-and-ship-to-staging` to land the hygiene diff, OR
  - Run `audit-all` first if you also want a security pass before shipping.
  - Or run `ship-ready` to do audit + ship in one shot.
```

## Hard Rules (never break)

- **Never reorder the sequence.** Each step's correctness depends on
  the previous one's output.
- **Never re-implement child logic.** If a child is wrong, fix the
  child SKILL.md; do not patch behavior here.
- **Soft-fail by design.** Children's hard-stops are reported but
  this orchestrator does NOT halt on them — the user wants a full
  hygiene snapshot.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging` (or use `ship-ready`).
- **Idempotence required.** Running `hygiene-all` twice in a row with
  no other intervening changes should produce zero diff on the
  second run (each child is idempotent; the orchestrator inherits
  that property).

## Trigger Examples

- "run all hygiene"
- "hygiene all"
- "full repo hygiene pass"
- "clean up the repo"
- "run the hygiene-all skill"

## Output Template

The orchestrator's output is the "Aggregated report shape" block
above, populated with real values per run.
