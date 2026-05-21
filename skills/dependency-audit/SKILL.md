---
name: dependency-audit
description: >-
  Run `pip-audit` against `requirements.txt` and `npm audit
  --audit-level=high` against `frontend/package.json`, then auto-bump
  pinned versions to fix CRITICAL and HIGH CVEs (MODERATE is reported
  for human decision). Verifies the resulting lockfiles resolve before
  declaring done. Use when the user says "audit dependencies", "check
  for vulnerable packages", or "run dependency audit".
---

# Dependency Audit

CVE sweep over both package managers. Auto-bumps HIGH+ findings only —
MODERATE and LOW are surfaced as a list because the cost/benefit on
non-exploitable advisories varies per project and shouldn't be
automated.

## When to use this skill

- As the second step of `audit-all` (after `secret-scan`) so version
  bumps land before any other auto-fixes touch the codebase.
- Weekly as a hygiene sweep.
- Anytime CI reports a Dependabot alert.

## Workflow

```
Task progress:
- [ ] Verify tooling: pip-audit available (or installable), npm available
- [ ] Snapshot current pinned versions for the rollback note
- [ ] Run pip-audit against requirements.txt; classify by severity
- [ ] Run npm audit --omit=dev --audit-level=high in frontend/; classify by severity
- [ ] Auto-bump CRITICAL + HIGH (no API/breaking-change packages, see below)
- [ ] Dry-run resolve to confirm no conflicts
- [ ] Print findings + rollback instructions for any bumped package
```

### Step 1. Tooling

```bash
command -v pip-audit || pip install --quiet pip-audit
command -v npm  # required; do not auto-install npm
```

If `npm` isn't installed, skip the frontend sweep and surface the gap
in the report — don't fail the whole skill.

### Step 2. Snapshot

Before any bump, capture the current pinned set so the user can roll
back if a bump turns out to break something:

```bash
mkdir -p .skills-cache/$(date +%s)
cp requirements.txt .skills-cache/$(date +%s)/requirements.txt.before
cp frontend/package.json .skills-cache/$(date +%s)/package.json.before
cp frontend/package-lock.json .skills-cache/$(date +%s)/package-lock.json.before
```

(Use a per-run timestamped subfolder under `.skills-cache/` so prior
snapshots aren't clobbered. Add `.skills-cache/` to `.gitignore`.)

### Step 3. pip-audit

```bash
pip-audit --requirement requirements.txt --format json --strict > /tmp/pip-audit.json
```

Parse the JSON and group by `vulns[].fix_versions` and CVSS severity.

### Step 4. npm audit

```bash
cd frontend && npm audit --omit=dev --audit-level=high --json > /tmp/npm-audit.json
```

Parse `metadata.vulnerabilities` (count by severity) and `vulnerabilities`
(per-package details with `fixAvailable`).

### Step 5. Classify and bump

For each CVE finding:

| Severity | Action |
|----------|--------|
| CRITICAL | Auto-bump to the lowest fix version. If the bump is a major-version jump on a package in `KNOWN_BREAKING_PACKAGES` (see below), DO NOT auto-bump — flag for human review with the recommended version. |
| HIGH | Same as CRITICAL. |
| MODERATE | Report only. Do not auto-bump. |
| LOW | Report only. |

`KNOWN_BREAKING_PACKAGES` — major-version bumps on these are always
human-review because they commonly require code changes in this codebase:

- **Python**: `fastapi`, `pydantic`, `sqlalchemy`, `alembic`, `passlib`,
  `python-jose`, `openai`, `anthropic`, `google-generativeai`,
  `httpx`, `uvicorn`.
- **npm**: `next`, `react`, `react-dom`, `typescript`,
  `@tanstack/react-query`, `tailwindcss`, `@radix-ui/*`,
  `framer-motion` (if used).

Minor and patch bumps on these packages ARE auto-applied for CRITICAL /
HIGH. Major bumps are flagged.

For `requirements.txt`, the bump is an in-place version replacement:

```diff
-fastapi==0.110.0
+fastapi==0.110.3
```

For npm, run:

```bash
cd frontend && npm install <package>@<fix-version>
```

so `package.json` AND `package-lock.json` update consistently.

### Step 6. Verify resolution

After all bumps:

```bash
pip install -r requirements.txt --dry-run > /tmp/pip-resolve.log 2>&1
cd frontend && npm install --dry-run --no-audit > /tmp/npm-resolve.log 2>&1
```

If either dry-run fails (conflict, peer-dep break, missing version),
revert ALL bumps from this run (`cp` back from the snapshot directory)
and surface the conflict in the report. Partial bumps risk leaving the
project in a half-upgraded state and are NEVER committed.

### Step 7. Output the rollback note

Every bumped package gets a one-line rollback instruction the user can
paste back if needed:

```
# To roll back this batch:
cp .skills-cache/<timestamp>/requirements.txt.before requirements.txt
cp .skills-cache/<timestamp>/package.json.before frontend/package.json
cp .skills-cache/<timestamp>/package-lock.json.before frontend/package-lock.json
cd frontend && npm install --no-audit
```

## Hard Rules (never break)

- **Never auto-bump a major version** on any package in
  `KNOWN_BREAKING_PACKAGES`. Surface for human review with the
  recommended version.
- **Never commit partial bumps.** If the dry-run resolve fails, revert
  the entire batch from the snapshot.
- **Never bump only `package.json`** without also updating
  `package-lock.json`. Use `npm install <pkg>@<ver>` (or
  `npm update <pkg>`), never hand-edit the lockfile.
- **Never bump packages with MODERATE or LOW severity findings**
  automatically — only surface them. Those advisories often have
  non-exploitable trigger conditions in this codebase.
- **Never push, commit, or PR.** Modifies `requirements.txt`,
  `package.json`, `package-lock.json` in the working tree only. Hand
  off to `review-and-ship-to-staging`.
- **Never run `npm audit fix --force`**. That command happily applies
  breaking-change major bumps. Use the explicit `npm install <pkg>@<ver>`
  path instead.

## Trigger Examples

- "audit dependencies"
- "check for vulnerable packages"
- "run dependency audit"
- "scan for CVEs"
- "run the dependency-audit skill"

## Output Template

End the run with:

```
Dependency-audit summary

Python (pip-audit)
- CVEs found:                <total>
  - CRITICAL:                <N>  (auto-bumped: <K>, manual: <N-K>)
  - HIGH:                    <N>  (auto-bumped: <K>, manual: <N-K>)
  - MODERATE:                <N>  (report only)
  - LOW:                     <N>  (report only)
- Packages bumped:           <list of pkg X.Y.Z -> X.Y.Z>
- Resolve dry-run:           pass | fail (reverted)

npm (frontend)
- CVEs found:                <total>
  - CRITICAL:                <N>  (auto-bumped: <K>, manual: <N-K>)
  - HIGH:                    <N>  (auto-bumped: <K>, manual: <N-K>)
  - MODERATE / LOW:          <N>  (report only)
- Packages bumped:           <list>
- Resolve dry-run:           pass | fail (reverted)

Manual-review queue
  • <package>  <current>  →  recommended <fix-ver>  (reason: major-version bump on KNOWN_BREAKING_PACKAGE / no clean fix)

Rollback
  cp .skills-cache/<timestamp>/requirements.txt.before requirements.txt
  cp .skills-cache/<timestamp>/package.json.before frontend/package.json
  cp .skills-cache/<timestamp>/package-lock.json.before frontend/package-lock.json
  (cd frontend && npm install --no-audit)

Next step: review the manual queue, then run `review-and-ship-to-staging`.
```
