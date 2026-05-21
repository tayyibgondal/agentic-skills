---
name: secret-scan
description: >-
  Scan the entire repo for leaked API keys, tokens, and credentials using a
  curated regex set (and gitleaks if installed). Auto-extracts any inline
  values it finds into `os.getenv(...)` / `process.env.X` lookups with a
  matching placeholder in `env.local.example`. Use when the user says
  "scan for secrets", "check for leaked keys", "secret scan", or anytime
  before a release / public push to be sure nothing slipped in.
---

# Secret Scan

A repo-wide credential leak detector and auto-extractor. Always run this
FIRST in any audit chain — every other audit-and-fix skill may modify
files, and we want a clean "no secrets present" baseline before any other
edits land in the working tree.

## When to use this skill

- Before `review-and-ship-to-staging` on any chat that touched config,
  router, or service files.
- As the first step of `audit-all`.
- Anytime the user even hints at having pasted a key into the codebase.

## Workflow

```
Task progress:
- [ ] Decide scan backend (gitleaks if available, regex fallback otherwise)
- [ ] Run the scan across all tracked files (excluding .git, node_modules, __pycache__, .next, dist, build)
- [ ] Cross-reference each hit against `env.local.example` placeholders (ignore documented dummies)
- [ ] For every real hit: replace the literal with an env-var lookup, add the placeholder to env.local.example
- [ ] Run a second scan pass to confirm zero hits
- [ ] Print the rotation checklist (provider + dashboard URL per leaked key)
```

### Step 1. Pick the scan backend

Prefer `gitleaks` if it's already installed:

```bash
command -v gitleaks && gitleaks detect --no-banner --redact --source . \
  --config .gitleaks.toml 2>/dev/null || gitleaks detect --no-banner --redact --source .
```

If gitleaks is not installed, fall back to a `ripgrep` (`rg`) sweep with
this regex set. These are the patterns this codebase actually risks
leaking (LLM providers + payment + GitHub + AWS), plus a generic
high-entropy fallback:

| Provider | Regex |
|----------|-------|
| OpenAI | `sk-[A-Za-z0-9]{32,}` and `sk-proj-[A-Za-z0-9_-]{40,}` |
| Anthropic | `sk-ant-[A-Za-z0-9_-]{32,}` |
| xAI | `xai-[A-Za-z0-9]{32,}` |
| Google AI / Gemini | `AIza[0-9A-Za-z_-]{35}` |
| GCP service account JSON | `"private_key":\s*"-----BEGIN PRIVATE KEY-----` |
| AWS access key | `AKIA[0-9A-Z]{16}` |
| AWS secret | `(?i)aws_secret_access_key\s*[:=]\s*["']?[A-Za-z0-9/+=]{40}` |
| Stripe live secret | `sk_live_[A-Za-z0-9]{24,}` |
| Stripe restricted | `rk_live_[A-Za-z0-9]{24,}` |
| GitHub personal access token | `ghp_[A-Za-z0-9]{36}` |
| GitHub fine-grained PAT | `github_pat_[A-Za-z0-9_]{82}` |
| Slack bot token | `xoxb-[A-Za-z0-9-]{50,}` |
| Slack user token | `xoxp-[A-Za-z0-9-]{50,}` |
| Pinecone | `pcsk_[A-Za-z0-9_]{40,}` |
| JWT secret leak | `JWT_SECRET\s*=\s*["'][^"'$\s]{16,}["']` |
| Inline bearer | `(?i)authorization\s*[:=]\s*["']?bearer\s+[A-Za-z0-9._-]{20,}` |
| Postgres URL with password | `postgres(?:ql)?://[^:\s]+:[^@\s]+@` |
| MongoDB URL with password | `mongodb(?:\+srv)?://[^:\s]+:[^@\s]+@` |
| Private RSA key block | `-----BEGIN (?:RSA |EC |OPENSSH |)PRIVATE KEY-----` |

Run as:

```bash
rg -n --hidden \
  --glob '!.git' --glob '!node_modules' --glob '!__pycache__' \
  --glob '!.next' --glob '!dist' --glob '!build' --glob '!legenv' \
  --glob '!*.lock' --glob '!package-lock.json' \
  -e 'sk-[A-Za-z0-9]{32,}' \
  -e 'sk-ant-[A-Za-z0-9_-]{32,}' \
  -e 'AIza[0-9A-Za-z_-]{35}' \
  -e 'AKIA[0-9A-Z]{16}' \
  -e 'sk_live_[A-Za-z0-9]{24,}' \
  -e 'ghp_[A-Za-z0-9]{36}' \
  -e 'xoxb-[A-Za-z0-9-]{50,}' \
  -e 'pcsk_[A-Za-z0-9_]{40,}' \
  .
```

### Step 2. Filter documented placeholders

Read `[env.local.example](env.local.example)` and build a set of known
placeholder values (e.g. `sk-your-openai-api-key-here`,
`your-gemini-api-key-here`, `your-anthropic-api-key-here`). Any hit whose
literal value matches one of these strings is documentation, not a leak —
exclude it from the actionable list.

Likewise exclude hits in:
- `.git/`, `legenv/` (the Python venv), `node_modules/`, `__pycache__/`,
  `.next/`, `dist/`, `build/`, `frontend/.next/`, `frontend/node_modules/`.
- This skill's own SKILL.md file (regex patterns will self-match).
- `[env.local.example](env.local.example)` itself (it's all placeholders by design).
- Any file matching `*.test.{py,ts,tsx}` or `*_test.py` — test fixtures
  often need static dummy keys.

### Step 3. Extract inline values into env vars (auto-fix)

For each remaining hit, the agent rewrites the file:

**Python (`backend/**/*.py`)**: replace
```python
openai_api_key = "sk-abc123..."
```
with
```python
openai_api_key = os.getenv("OPENAI_API_KEY")
```
and add `import os` at the top if missing.

**TypeScript (`frontend/**/*.{ts,tsx}`)**: replace
```ts
const apiKey = "sk-abc123..."
```
with
```ts
const apiKey = process.env.NEXT_PUBLIC_OPENAI_API_KEY  // server-only secrets must use a non-NEXT_PUBLIC_ prefix
```
Use a `NEXT_PUBLIC_` prefix ONLY for values the frontend genuinely needs
at runtime (e.g. publishable Stripe key, Posthog key). Never use it for
server-only secrets — those move to a backend route.

**Docker compose / shell**: replace the literal with `${KEY_NAME}` and
add the corresponding `KEY_NAME=` placeholder line to
`[env.local.example](env.local.example)` so devs onboarding don't get
surprised.

### Step 4. Update `env.local.example`

For every new env var introduced by Step 3, append a section under the
matching header (LLM API Keys, Search, Database, etc. — preserve the
existing section layout) with the placeholder format already in use:

```
KEY_NAME=your-key-name-placeholder
```

Do NOT write the real value into `env.local.example`. The file is
committed to git.

### Step 5. Re-scan to confirm clean

Run the scan command from Step 1 again. The actionable hit count must be
zero. If a hit remains, the auto-fix didn't take — stop, report the
file/line, and ask the user to handle manually.

### Step 6. Print the rotation checklist

The skill cannot rotate keys for the user. It MUST end with a clear
checklist of every provider that had a leaked key, with the rotation URL:

| Provider | Rotate at |
|----------|-----------|
| OpenAI | https://platform.openai.com/api-keys |
| Anthropic | https://console.anthropic.com/settings/keys |
| Gemini / Google AI Studio | https://aistudio.google.com/app/apikey |
| xAI | https://console.x.ai/ |
| Stripe | https://dashboard.stripe.com/apikeys |
| GitHub PAT | https://github.com/settings/tokens |
| AWS | https://console.aws.amazon.com/iam/home#/security_credentials |
| Pinecone | https://app.pinecone.io/ |

Include in the checklist: which file the key was in, what env var it now
lives behind, and the rotation URL. Tell the user that anything that was
already pushed to a public remote MUST be rotated even after this skill
fixes the file — git history retains the leak.

## Hard Rules (never break)

- **Never print the full leaked value** in chat output or in any log.
  Always redact to the first 4 + last 4 characters (e.g. `sk-a…XyZ9`).
- **Never commit** an env file that contains real values, even if the
  user asks. The skill writes only to `[env.local.example](env.local.example)`,
  which holds placeholders.
- **Never auto-rotate** a key — the skill has no provider credentials. It
  only moves the literal out of source and tells the user to rotate.
- **Never silently exclude** a hit. If the agent decides a hit is a false
  positive (e.g. matches a known test fixture), it MUST list it under
  "Excluded hits" in the output so the user can override.
- **Never modify `.env`** — the working `.env` is git-ignored and is the
  source of truth for local secrets. Only `env.local.example` is edited.
- **Never push, commit, or PR**. This skill audits and rewrites the
  working tree only. Hand off to `review-and-ship-to-staging` if the user
  wants to land the fixes.

## Trigger Examples

- "scan for secrets"
- "check for leaked keys"
- "secret scan"
- "is there a key in the codebase?"
- "run the secret-scan skill"

## Output Template

End the run with:

```
Secret-scan summary
- Scan backend:        gitleaks | regex-fallback
- Files scanned:       <N>
- Hits found:          <total>
  - Documented dummies: <N> (excluded)
  - Test fixtures:      <N> (excluded)
  - Real leaks:         <N>
- Auto-fixes applied:  <N> (file → env var)
- Re-scan result:      clean | <N> remaining (LIST)
- Working tree dirty:  yes | no

Rotation checklist (do this NOW, even after the file fix):
  1. <provider>  →  <rotate URL>     was in <file>:<line>, now os.getenv("<KEY>")
  2. ...

Next step: run `review-and-ship-to-staging` to land the extraction, then rotate the keys above.
```
