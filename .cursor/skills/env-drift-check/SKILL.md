---
name: env-drift-check
description: >-
  Reconcile every environment variable referenced in code with what's
  declared in `env.local.example`, `docker-compose.local.yml`,
  `docker-compose.staging.yml`, and `docker-compose.production.yml`.
  Catches: vars referenced in code but missing from any compose /
  example (silent runtime failure waiting to happen); vars declared
  in compose but unused in code (cruft); values that differ between
  staging and production that probably shouldn't (e.g. log level).
  Auto-adds missing placeholders to `env.local.example`. Use when the
  user says "check env drift", "audit env vars", or
  "env-drift check".
---

# Env Drift Check

The "why does prod work but staging crash with KeyError: NEW_VAR_NAME"
skill. The most common pre-deploy bug in this codebase is adding a
new `os.getenv("FOO")` call without updating compose — this skill
catches that before the deploy.

## Stack-Specific Context

- Reference file: `[env.local.example](env.local.example)` — every var
  the codebase consumes MUST appear here with a placeholder.
- Local compose: `[docker-compose.local.yml](docker-compose.local.yml)`
  — usually loads from `.env`.
- Staging compose: `[docker-compose.staging.yml](docker-compose.staging.yml)`
  — explicit `environment:` block or `env_file:` directive.
- Production compose: `[docker-compose.production.yml](docker-compose.production.yml)`
  — same.
- Backend reads via `os.getenv(...)` / `os.environ[...]` / `os.environ.get(...)`.
- Frontend reads via `process.env.X` (browser only sees
  `NEXT_PUBLIC_*` — anything else is server-only at build/runtime).

## When to use this skill

- Any time a new `os.getenv` or `process.env.X` was added.
- As a Tier 4 step (deploy readiness) before
  `promote-staging-to-main`.
- Periodic hygiene (monthly).

## Workflow

```
Task progress:
- [ ] Sweep 1: every os.getenv / os.environ key in backend/
- [ ] Sweep 2: every process.env.X key in frontend/
- [ ] Parse env.local.example into a set
- [ ] Parse each docker-compose file into per-service env sets
- [ ] Cross-reference: code vs example, code vs each compose
- [ ] Diff staging vs production env blocks (flag suspicious deltas)
- [ ] Auto-fix: add missing placeholders to env.local.example
- [ ] Print categorized report
```

### Sweep 1 — backend env reads

```bash
rg -n 'os\.getenv\(\s*["\']([^"\']+)["\']' backend/ --type py -r '$1' --only-matching
rg -n 'os\.environ\[["\']([^"\']+)["\']\]' backend/ --type py -r '$1' --only-matching
rg -n 'os\.environ\.get\(\s*["\']([^"\']+)["\']' backend/ --type py -r '$1' --only-matching
```

Combine into a set `BACKEND_VARS` of unique env-var names. For each,
also record the file:line where it appears (top-3) for the report.

Exclude default-arg patterns where the key is empty or interpolated
(e.g. `os.getenv(key_name)` where `key_name` is a variable — those
are dynamic and the skill can't statically check them; surface as a
"dynamic, not checked" list).

### Sweep 2 — frontend env reads

```bash
rg -n 'process\.env\.([A-Z_][A-Z0-9_]*)' frontend/ --type ts --type tsx -r '$1' --only-matching
rg -n "process\\.env\\[['\"]([A-Z_][A-Z0-9_]*)['\"]\\]" frontend/ --type ts --type tsx -r '$1' --only-matching
```

Combine into `FRONTEND_VARS`. Note which are `NEXT_PUBLIC_*` (must be
defined at build time; safe to ship to browser) vs server-only.

### Sweep 3 — Parse declarations

For each file, build a set of declared keys:

- `[env.local.example](env.local.example)`: every `KEY=...` line that
  isn't a comment.
- `[docker-compose.local.yml](docker-compose.local.yml)`,
  `[docker-compose.staging.yml](docker-compose.staging.yml)`,
  `[docker-compose.production.yml](docker-compose.production.yml)`:
  per service, parse the `environment:` block (yaml list or dict form)
  AND any `env_file:` directives (read those files too).

Use `yq` if available (`yq '.services.*.environment' <file>`),
otherwise fall back to a tolerant YAML parse via Python (`pyyaml`).

### Sweep 4 — Cross-reference

For each var in `BACKEND_VARS ∪ FRONTEND_VARS`:

| Found in… | Action |
|-----------|--------|
| Code only | HIGH — runtime KeyError if the var isn't set. Auto-fix: add `KEY=` placeholder line to `env.local.example`. |
| Code + example, not in staging compose | HIGH — staging deploy will fail. Flag for manual addition to staging compose (the skill never edits compose files). |
| Code + example + staging, not in production | CRITICAL — production deploy will fail. Flag for manual addition. |
| Example + compose but not in code | MEDIUM — cruft. Flag with the file:line where it's defined. Don't auto-remove (vars are sometimes consumed by sidecar processes the skill doesn't see). |
| Code only AND dynamic key (variable name) | LOW — can't statically check; surface separately. |

### Sweep 5 — Staging vs production diff

For every var that appears in BOTH staging and production compose,
compare values:

- Identical: pass.
- Different (e.g. `LOG_LEVEL=DEBUG` vs `LOG_LEVEL=INFO`): pass with a
  note. Some vars are intentionally per-env.
- Suspicious: production has a more permissive value than staging —
  e.g. `DEBUG=true` in prod, `RATE_LIMIT_DISABLED=1` in prod. Flag as
  HIGH.

The suspicious-set the skill checks for:

```
DEBUG=true
DEVELOPMENT=1
RATE_LIMIT_DISABLED
ALLOW_ALL_ORIGINS
CORS_ALLOW_ALL
DISABLE_AUTH
MOCK_PAYMENTS
SKIP_EMAIL
```

Any of these set to a permissive value in production is HIGH.

### Sweep 6 — Auto-fix: add to env.local.example

For each "code only" var, append a new line under the appropriate
section header in `[env.local.example](env.local.example)`. Section
detection:

- Var name contains `API_KEY` or starts with `OPENAI_` / `ANTHROPIC_`
  / `GEMINI_` / `XAI_` → "LLM API Keys" section.
- Var name contains `DB`, `DATABASE`, or `POSTGRES` → "Database".
- Var name contains `STRIPE` or `PAYMENT` → "Payments".
- Var name contains `SMTP`, `EMAIL`, `SENDGRID`, `MAILGUN` → "Email".
- Var name contains `AWS`, `S3`, `BUCKET` → "AWS".
- Anything else → "Other".

If a section doesn't exist, append it. Use the existing comment
header format (the file already uses `# ===` separators).

Each new line is:

```
KEY_NAME=your-key-name-placeholder  # TODO: fill in
```

The `# TODO: fill in` suffix is intentional — onboarding devs grep for
it to know what's missing.

## Hard Rules (never break)

- **Never edit `docker-compose.*.yml` files.** Compose is the source
  of truth for what each environment exposes; changes there require a
  deploy. The skill REPORTS gaps and recommends additions but never
  applies them.
- **Never edit `.env`** (gitignored — local secrets).
- **Never remove a var** from `env.local.example` even if it appears
  unused. The cost of orphan removal (breaking a script the skill
  doesn't see) outweighs the benefit of cleanup. Surface as MEDIUM
  for human cleanup instead.
- **Never write real values** to `env.local.example`. Only
  placeholders / `# TODO: fill in` markers.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging`.

## Trigger Examples

- "check env drift"
- "audit env vars"
- "env-drift check"
- "are all my env vars wired up?"
- "run the env-drift-check skill"

## Output Template

End the run with:

```
Env-drift-check summary

Code references
  - Backend (os.getenv / os.environ):  <N> unique vars
  - Frontend (process.env.X):          <N> unique vars  (NEXT_PUBLIC_ count: <K>)
  - Dynamic / unparseable:             <N> (LOW)

Declarations
  - env.local.example:                 <N> vars
  - docker-compose.local.yml:          <N> vars
  - docker-compose.staging.yml:        <N> vars
  - docker-compose.production.yml:     <N> vars

Findings
- CRITICAL  Code-only var missing from production compose:  <N>
    • <var>  used at <file>:<line>
- HIGH      Code-only var missing from staging compose:     <N>
- HIGH      Suspicious permissive prod value:               <N>
    • <var>=<value>  (compare staging: <value>)
- HIGH      Code-only var missing from env.local.example:   <N>  (auto-fixed: <K>)
- MEDIUM    Declared but unused (cruft):                    <N>
- LOW       Dynamic keys (not statically verified):         <N>

Auto-fixes applied
  - env.local.example: appended <K> placeholder lines (sections: <list>)

Next step: review CRITICAL / HIGH compose gaps and apply manually, then run `review-and-ship-to-staging`.
```
