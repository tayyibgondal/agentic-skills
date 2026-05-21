---
name: audit-all
description: >-
  Orchestrator that runs every Tier 1 pre-ship audit-and-fix skill in
  a deliberate order: `secret-scan` → `dependency-audit` →
  `auth-flow-audit` → `security-audit` → `concurrency-audit` →
  `migration-safety` → `performance-audit`. Aggregates each child's
  output into ONE combined report. Hard-stops on a CRITICAL secret
  leak OR an unresolved CRITICAL auth finding; everything else is
  soft-fail and the run continues. Use when the user says "audit
  all", "run all audits", "full pre-ship audit", or "audit
  everything before I ship".
---

# Audit All (Tier 1 orchestrator)

One prompt, every pre-ship audit, one combined report. This skill
does NOT re-implement any audit logic — it literally reads the child
SKILL.md files and executes them in order. Improving any leaf skill
(e.g. adding a new regex to `secret-scan`) automatically improves
this orchestrator with zero duplication.

## When to use this skill

- Daily, before leaving your desk on a chat that touched code.
- Always before promoting `staging` → `main`.
- Anytime the user is suspicious about a chat's blast radius.

## Workflow

```
Task progress:
- [ ] Pre-flight: snapshot current HEAD SHA + working-tree state for the aggregated diff
- [ ] Step 1: secret-scan                — hard-stop checkpoint
- [ ] Step 2: dependency-audit
- [ ] Step 3: auth-flow-audit            — hard-stop checkpoint
- [ ] Step 4: security-audit
- [ ] Step 5: concurrency-audit
- [ ] Step 6: migration-safety           — skipped if no Alembic changes
- [ ] Step 7: performance-audit          — always soft-fail
- [ ] Build the aggregated report
- [ ] Print the report + final hand-off line
```

### Why this exact order

1. **`secret-scan` first** — every other audit may rewrite files, and
   we want a clean "no secrets present" baseline before any other
   edits land in the working tree. Also the cheapest skill in the
   chain, so we fail fast if there's a leak.
2. **`dependency-audit` second** — version bumps happen before the
   auto-fixes for security / perf / concurrency land, so those fixes
   apply on the upgraded codebase. (A perf-audit fix that wraps an
   LLM call in `asyncio.wait_for` is wasted if the next bump rewrites
   the LLM SDK shape.)
3. **`auth-flow-audit` third** — highest blast radius. If something
   here is CRITICAL the run stops — there is no point auditing CORS
   or N+1 queries on a codebase with an unguarded route.
4. **`security-audit` fourth** — covers everything `auth-flow-audit`
   does not (SQL injection, CORS, file uploads, dangerous builtins).
5. **`concurrency-audit` fifth** — formalizes the rules already in
   `[.cursor/skills/review-and-ship-to-staging/SKILL.md](.cursor/skills/review-and-ship-to-staging/SKILL.md)`.
6. **`migration-safety` sixth** — cheap: self-skips if no Alembic
   migrations changed in the working tree.
7. **`performance-audit` last** — soft-fail, mostly informational.
   Always runs because perf findings inform whether the user wants
   to ship now or batch up improvements.

### Execution model

For each step:

1. Read the corresponding `.cursor/skills/<name>/SKILL.md` and
   execute its workflow exactly as written.
2. Capture the "Output Template" block that the child prints at the
   end. That block becomes one section of the aggregated report.
3. Track files modified by that step (via `git diff --name-only`
   against the pre-orchestrator HEAD) so the aggregated report can
   attribute each file change to a specific audit.
4. Apply the failure policy (see below) and decide whether to
   continue.

### Failure policy (Hard-stop vs Soft-fail)

| Step | Hard-stop trigger | Continue on |
|------|-------------------|-------------|
| `secret-scan` | A real leak (post-filter) is detected and not auto-extracted | Otherwise |
| `dependency-audit` | Resolution dry-run fails (`pip install --dry-run` / `npm install --dry-run`) | Otherwise |
| `auth-flow-audit` | Any unresolved CRITICAL finding | Otherwise |
| `security-audit` | Any unresolved CRITICAL finding | Otherwise |
| `concurrency-audit` | Any unresolved CRITICAL finding | Otherwise |
| `migration-safety` | Any unresolved CRITICAL finding | Otherwise |
| `performance-audit` | NEVER hard-stops (soft-fail by design) | Always |

When a hard-stop fires, the orchestrator:

- Stops at that step (does NOT run subsequent steps).
- Builds the aggregated report covering all steps run so far.
- Prints a clear `HARD-STOP at step <N>: <reason>` banner at the top.
- Tells the user how to resolve and re-run.

### Aggregated report shape

```
audit-all summary

Pre-flight
- Pre-run HEAD:      <sha>
- Working tree:      clean | dirty (N files)

Step 1 — secret-scan          : pass | HARD-STOP (<reason>)
Step 2 — dependency-audit     : pass | <N CVEs auto-bumped, K manual> | HARD-STOP
Step 3 — auth-flow-audit      : pass | HARD-STOP (<reason>) | <N critical unresolved>
Step 4 — security-audit       : pass | <N findings>
Step 5 — concurrency-audit    : pass | <N findings>
Step 6 — migration-safety     : pass | skipped (no Alembic changes)
Step 7 — performance-audit    : <N findings>  (always soft-fail)

Files modified by this run    : <N>
  - by secret-scan:           <list>
  - by dependency-audit:      <list>
  - by auth-flow-audit:       <list>
  - ...

CRITICAL findings still requiring human review
  • <step>  <file>:<line>  <finding>
  • ...

HARD-STOP:                    yes | no  (if yes, the chain stopped at step N)

Next step:
  - If HARD-STOP: resolve the blocker above, then re-run `audit-all`.
  - If pass: run `hygiene-all`, then `review-and-ship-to-staging`.
  - Or run `ship-ready` to do hygiene + ship in one shot.
```

## Hard Rules (never break)

- **Never reorder the sequence.** Each step depends on the previous
  one having landed its auto-fixes (e.g. dependency-audit's bumps
  must precede security-audit's auto-fixes).
- **Never skip a step** unless the step's own SKILL.md says it can
  self-skip (only `migration-safety` does, when no Alembic changes
  are present).
- **Never re-implement child logic.** If a child skill is incorrect,
  fix the child SKILL.md; do not patch behavior inside this file.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging` (or use `ship-ready`).
- **Never override a child's hard-stop policy** to keep going. The
  child decided that finding is unsafe to continue past; respect it.
- **Never silently degrade an unresolved CRITICAL to a HIGH** in the
  aggregated report. Surface exactly what the child surfaced.

## Trigger Examples

- "audit all"
- "run all audits"
- "full pre-ship audit"
- "audit everything before I ship"
- "run the audit-all skill"

## Output Template

The orchestrator's output is the "Aggregated report shape" block
above, populated with real values per run. Print it at the end of
the run with the appropriate banner at the top if a hard-stop fired.
