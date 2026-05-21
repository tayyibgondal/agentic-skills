---
name: file-size-enforcer
description: >-
  Walk every source file in the repo (Python, TypeScript, TSX) and
  enforce the 700-line cap from
  `[.cursor/skills/review-and-ship-to-staging/SKILL.md](.cursor/skills/review-and-ship-to-staging/SKILL.md)`
  lines 24-27. For each violator, propose a concrete split plan
  (which functions / components / route handlers belong in which new
  file). Auto-apply the split for files <= 1000 lines; files over 1000
  get a human-review TODO with the proposed split written into the
  file as a comment block. Use when the user says "enforce file size",
  "split oversized files", or "run file size enforcer".
---

# File-Size Enforcer

Codebase-wide application of the 700-line rule from the ship skill.
Where the ship skill only enforces it on chat-edited files, this skill
enforces it everywhere on demand.

## Why the cap exists

Files over 700 lines historically:
- Resist refactoring (every change touches every reader's mental model).
- Hide accumulating responsibilities that should be separate modules.
- Make `ReadLints` / `mypy` errors hard to scope.

700 is the trigger; 1000 is the "the agent must not autonomously rewrite
something this large" hard cap.

## When to use this skill

- Periodically (e.g. monthly).
- After any chat that pushed several files over the cap.
- As the third (final) step of `hygiene-all`.

## Workflow

```
Task progress:
- [ ] Enumerate all source files: rg --files -t py -t ts -t tsx
- [ ] Filter out vendor / generated paths (node_modules, .next, __pycache__, dist, build, legenv, migrations/versions)
- [ ] Sort by line count, descending
- [ ] For each violator (>700): classify size band, propose split plan
- [ ] Auto-apply splits for files in 701-1000 band
- [ ] Write TODO comment for files >1000 (do NOT auto-split)
- [ ] Re-measure after auto-splits; verify each new file is under the cap
- [ ] Print report
```

### Step 1. Enumerate

```bash
rg --files \
  --type py --type ts --type tsx \
  --glob '!node_modules' --glob '!.next' --glob '!__pycache__' \
  --glob '!dist' --glob '!build' --glob '!legenv' \
  --glob '!backend/db/migrations/versions/*' \
  --glob '!frontend/components/ui/*'  \
  | xargs wc -l \
  | sort -rn \
  | head -100
```

Excluded paths and the rationale:

- `node_modules`, `.next`, `__pycache__`, `dist`, `build`, `legenv` —
  vendor / generated, not source.
- `backend/db/migrations/versions/*` — Alembic migrations can be long
  for legitimate reasons (large data backfills); splitting them is
  rarely safe.
- `frontend/components/ui/*` — shadcn primitives are vendored from
  upstream; do not modify.

### Step 2. Classify per-file

| Size band | Action |
|-----------|--------|
| 0–700 | Pass. Skip. |
| 701–1000 | Auto-apply split. |
| 1001+ | Human review only. Write proposed split as a comment block at the top of the file. |
| 1500+ | HIGH severity flag in addition to the comment block. |

### Step 3. Propose the split (both bands)

The split plan is the SAME format regardless of whether it's applied or
written as a comment. For each violator:

1. Read the file end-to-end.
2. Group functions / classes / React components by responsibility. Heuristics:
   - All Pydantic schemas → `<base>_schemas.py`.
   - All admin-only handlers → `<base>_admin.py`.
   - All helper / pure functions used only inside the file → `<base>_helpers.py`.
   - All React subcomponents > 60 lines that aren't exported → their
     own file under `<dir>/<base>/<subcomponent>.tsx`.
   - All `useEffect` orchestration logic that exceeds 30 lines → a
     custom hook under `frontend/lib/hooks/`.
3. Compute the resulting line counts; if any proposed file is still
   > 700, recurse on it before finalizing the plan.

### Step 4. Auto-apply (band 701–1000 only)

For each accepted split:

1. Create the new file with the appropriate imports.
2. MOVE (not copy) the targeted code blocks.
3. Update the original file to `from .<new_module> import <names>` so
   downstream callers keep working.
4. Re-run `rg` to find any other module that imported from the
   original by symbol name — those still work because the original
   re-exports.

If a split would require changing a public API (file path used by
callers outside the package), the skill DOWNGRADES that split to
"human review" and emits the comment block instead. Better to leave
a 900-line file than to break callers.

### Step 5. Comment block (band 1001+)

For files this large, the skill writes a comment at the top:

```python
# TODO(file-size-enforcer): This file is <N> lines. Proposed split:
#   - <new_path>:   <list of names>
#   - <new_path>:   <list of names>
#   - keep here:    <list of names>
# Reason: <one line>
# When ready, apply manually OR delete this comment to re-evaluate.
```

The skill is idempotent — if the comment already exists, it is
refreshed in place (not duplicated) and the report notes "stale TODO
refreshed".

### Step 6. Re-measure

After all auto-applies, re-run Step 1's `wc -l` pipeline. Every file
auto-split must be under 700 in the new state. If any isn't, revert
the split for that file (the heuristic was wrong) and emit a comment
block instead.

## Hard Rules (never break)

- **Never auto-split a file above 1000 lines.** The risk of getting
  the responsibility boundaries wrong is too high. Always write the
  comment block instead.
- **Never split** files in `[frontend/components/ui/](frontend/components/ui/)`
  — these are shadcn primitives.
- **Never split** files in `[backend/db/migrations/versions/](backend/db/migrations/)`
  — migrations are atomic units.
- **Never split** files that re-export from a published `index.ts` /
  `__init__.py` barrel unless the barrel is also updated.
- **Never modify imports in another package** to point at the new
  split file. Always update via re-export from the original.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging`.
- **Idempotence is required.** Running this skill twice in a row with
  no other changes must produce zero new edits on the second run.

## Trigger Examples

- "enforce file size"
- "split oversized files"
- "run file size enforcer"
- "any files too big?"
- "run the file-size-enforcer skill"

## Output Template

End the run with:

```
File-size-enforcer summary

Files scanned (filtered):        <N>
Over 700 lines:                  <N>
  - 701–1000 (auto-split):       <K>
  - 1001–1500 (comment block):   <M>
  - 1501+ (HIGH flag):           <L>

Auto-splits applied
  • <original-path>  (was <X> lines)
    → <new-path-1>    (<a> lines)
    → <new-path-2>    (<b> lines)
    → original now    <c> lines
  • ...

Comment blocks written (manual review)
  • <path>:1  (<X> lines)  proposed: <new-file-1>, <new-file-2>
  • ...

Files refreshed (stale TODO updated): <N>

Re-measure
  • Every auto-split file now under 700:  pass | fail (reverted)

Next step: review the manual queue, then run `review-and-ship-to-staging`.
```
