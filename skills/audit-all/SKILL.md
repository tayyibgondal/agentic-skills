---
name: audit-all
description: >-
  Orchestrator that runs every pre-ship audit-and-fix skill in a
  deliberate order: `secret-scan` → `dependency-audit` →
  `security-audit`. Aggregates each child's output into ONE combined
  report. Hard-stops on a CRITICAL secret leak OR any unresolved
  CRITICAL security finding; lesser findings are soft-fail and the
  run continues. Use when the user says "audit all", "run all
  audits", "full pre-ship audit", or "audit everything before I
  ship".
---

# Audit All (orchestrator)

One prompt, every pre-ship audit, one combined report. This skill
does NOT re-implement any audit logic — it literally reads the child
SKILL.md files and executes them in order. Improving any leaf skill
(e.g. adding a new regex to `secret-scan`) automatically improves
this orchestrator with zero duplication.

## When to use this skill

- Daily, before leaving your desk on a chat that touched code.
- Always before promoting `staging` → `main`.
- Anytime you're suspicious about a chat's blast radius.

## Workflow

```
Task progress:
- [ ] Pre-flight: snapshot current HEAD SHA + working-tree state for the aggregated diff
- [ ] Step 1: secret-scan         — hard-stop checkpoint
- [ ] Step 2: dependency-audit
- [ ] Step 3: security-audit      — hard-stop checkpoint
- [ ] Build the aggregated report
- [ ] Print the report + final hand-off line
```

### Why this exact order

1. **`secret-scan` first** — every other audit may rewrite files, and
   we want a clean "no secrets present" baseline before any other
   edits land in the working tree. Also the cheapest skill in the
   chain, so we fail fast if there's a leak.
2. **`dependency-audit` second** — version bumps happen before the
   auto-fixes for security land, so those fixes apply on the upgraded
   codebase.
3. **`security-audit` third** — SQL injection, CORS, schema
   validation, file uploads, dangerous builtins. Runs last because
   its auto-fixes are the most invasive and benefit from a clean
   secret + dependency baseline.

### Execution model

For each step:

1. Read the corresponding `skills/<name>/SKILL.md` and execute its
   workflow exactly as written.
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
| `security-audit` | Any unresolved CRITICAL finding | Otherwise |

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
Step 3 — security-audit       : pass | HARD-STOP (<reason>) | <N findings>

Files modified by this run    : <N>
  - by secret-scan:           <list>
  - by dependency-audit:      <list>
  - by security-audit:        <list>

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
- **Never skip a step.** If a step is irrelevant for the current
  repo (e.g. no `requirements.txt`), the child skill self-skips and
  reports — the orchestrator never decides for it.
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
