---
name: ship-ready
description: >-
  Top-level orchestrator: runs `audit-all` end-to-end, then
  `hygiene-all` end-to-end, prints ONE combined report, then asks
  the user (in chat) "All checks pass. Hand off to
  review-and-ship-to-staging now? (yes / no)". On yes, invokes
  `review-and-ship-to-staging` to land ALL the auto-fixes as a
  single PR titled `chore: ship-ready autofixes (<summary>)`. This
  is the ONLY interactive prompt in the entire skill system — every
  other skill is fully autonomous. Use when the user says "ship
  ready", "full ship-ready check", "audit and hygiene then ship",
  or "get this ready to ship".
---

# Ship Ready (top-level orchestrator)

The heavyweight option. Run this when you're about to ship a
non-trivial change to production and want every audit + every hygiene
pass + the actual ship — gated by one human confirmation at the
exact moment the working tree becomes a remote-affecting action.

## When to use this skill

- "I'm shipping a non-trivial change to production this hour."
- Before any release that touches auth, billing, or other sensitive
  surfaces.
- When you'd otherwise run `audit-all` + `hygiene-all` +
  `review-and-ship-to-staging` back to back manually.

## When NOT to use this skill

- Daily small fixes that don't touch sensitive surfaces — go straight
  to `review-and-ship-to-staging`. The full audit + hygiene takes
  several minutes; daily fixes don't earn that overhead.
- WIP checkpoints — use `commit-chat-to-branch` instead (no ship).

## Workflow

```
Task progress:
- [ ] Pre-flight: snapshot HEAD SHA so we can scope the eventual PR diff
- [ ] Step 1: invoke audit-all end-to-end
- [ ] If audit-all hard-stops, ship-ready stops too
- [ ] Step 2: invoke hygiene-all end-to-end
- [ ] Step 3: build the combined audits + hygiene aggregated report
- [ ] Step 4: print the report
- [ ] Step 5: ASK THE USER in chat — "Hand off to review-and-ship-to-staging? (yes / no)"
- [ ] If yes: invoke review-and-ship-to-staging with a chore: ship-ready autofixes PR
- [ ] If no: end with the report, remind the user that the working tree has uncommitted changes
```

### Step 1. `audit-all`

Read `skills/audit-all/SKILL.md` and execute its full workflow. If
`audit-all` reports a HARD-STOP, THIS orchestrator stops too — print
the audit-all report verbatim, do NOT proceed to hygiene-all, do NOT
ask the ship-confirmation question. Reason: the project isn't ready
to ship until the hard-stop is resolved.

### Step 2. `hygiene-all`

Read `skills/hygiene-all/SKILL.md` and execute its full workflow.
Soft-fail by design — always completes.

### Step 3. Combined aggregated report

Concatenate the audit-all and hygiene-all "Output Template" blocks
into one report:

```
ship-ready summary

Pre-flight
- Pre-run HEAD:           <sha>
- Working tree at start:  clean | dirty

=================================================================
Audits   (from audit-all)
=================================================================
<full audit-all output here>

=================================================================
Hygiene  (from hygiene-all)
=================================================================
<full hygiene-all output here>

=================================================================
Combined deltas
=================================================================
- Files modified by this run:    <N>
  - by audits:                   <list>
  - by hygiene:                  <list>
  - overlap (touched by both):   <list>

Overall outcome:
  - Hard-stops:                  <none | LIST>
  - CRITICAL unresolved:         <N>
  - HIGH unresolved:             <N>
  - MEDIUM unresolved:           <N>

Recommendation:
  - HARD-STOPS exist → DO NOT SHIP. Fix the blockers and re-run `ship-ready`.
  - No hard-stops + CRITICAL = 0 → SAFE TO SHIP.
  - No hard-stops + CRITICAL > 0 (audit-all let them through somehow) → SHIP AT YOUR OWN RISK.
```

### Step 4. Print the report

Print the combined report in chat so the user has full visibility.

### Step 5. ASK THE USER — the only interactive prompt in the system

Print the question literally, plain text, no surrounding ceremony:

```
All checks complete. Working tree has uncommitted changes from this
run. Hand off to `review-and-ship-to-staging` now? (yes / no)
```

Wait for the user's reply.

This question exists because the next step actually mutates the
remote (branch + PR + merge), and the user must consciously approve
that boundary — even in autonomous mode, even in `ship-ready`. NEVER
skip this confirmation under any circumstance.

If the user says `yes` (or any clear affirmative — "ship it", "go",
"land it"), proceed to Step 6.

If the user says `no` (or any clear negative — "wait", "let me
review", "stop"), end the skill with a short note:

```
Stopping. Working tree has uncommitted changes from ship-ready.
Inspect with `git status` and `git diff`. When ready, run
`review-and-ship-to-staging`.
```

If the user response is ambiguous, re-ask once with explicit yes/no
options. After a second ambiguous response, default to NO (refusing
to ship without clear consent is always the safer choice).

### Step 6. Invoke `review-and-ship-to-staging`

Read `skills/review-and-ship-to-staging/SKILL.md` and execute its
workflow with two adjustments:

1. **Scope the file list** via `git diff --name-only <pre-flight-sha>
   HEAD` instead of building it from the chat's edited files. The
   chat may have touched files manually too — only the files
   modified by `audit-all` + `hygiene-all` belong in the ship-ready
   PR.
2. **PR title** is fixed:

   ```
   chore: ship-ready autofixes (<short summary of fix categories>)
   ```

   The summary is a 1-line list of the audits / hygiene that
   contributed changes — e.g. `(secret-scan, security-audit,
   lint-and-typecheck-fix, file-size-enforcer)`.

3. **PR body** quotes the combined aggregated report so the reviewer
   has full context.

Everything else in the ship workflow (branch from `origin/staging`,
stash, commit, push, MCP `create_pull_request`, MCP merge) runs
unchanged.

### Step 7. Report

Print the PR URL, merge SHA, and the recommendation chain:

```
ship-ready complete

PR:                       <url>
Merge SHA:                <sha>
Branch:                   fix/ship-ready-<timestamp>

Next step (optional):
  - Run `promote-staging-to-main` to ship staging → production.
  - Then `release-tag` to label the release.
  - Then `release-notes-gen` to publish the GitHub Release page.
```

## Hard Rules (never break)

- **NEVER skip the user-confirmation prompt** (Step 5). The boundary
  between "modify working tree" (every other skill) and "open a PR
  on the remote" (this step) is the one place the system requires
  human consent.
- **NEVER batch unrelated edits into the ship PR.** Use the
  pre-flight HEAD diff to scope what to add — anything the user
  hand-edited before invoking ship-ready stays in their working tree.
- **NEVER proceed to hygiene-all if audit-all hard-stops.** A
  hard-stop means the project isn't shippable; running more passes
  on top would just be noise.
- **NEVER override a "no" answer.** If the user declines the ship
  prompt, end the skill — do not retry, do not "are you sure".
- **NEVER ask any other interactive question.** This skill has
  exactly one: Step 5's ship/don't-ship confirmation. Everything
  else is autonomous.
- **NEVER tag the release** as part of ship-ready. Tagging happens
  AFTER `promote-staging-to-main` lands the changes on `main`.

## Trigger Examples

- "ship ready"
- "full ship-ready check"
- "audit and hygiene then ship"
- "get this ready to ship"
- "run the ship-ready skill"

## Output Template

The skill prints the combined report at Step 4, then (if the user
says yes) prints the PR URL + merge SHA at Step 7 in the format
shown above.
