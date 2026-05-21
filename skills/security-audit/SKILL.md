---
name: security-audit
description: >-
  Audit non-auth security risks across the backend and frontend: SQL
  injection via unparameterized queries, CORS misconfiguration, missing
  request-body validation on FastAPI routes, insecure cookie attributes,
  unsafe file-upload handlers, and dangerous Python builtins
  (`pickle.load`, `eval`, `exec`). Auto-fixes the safe ones. Use when
  the user says "audit security", "security audit and fix", or "find
  security issues".
---

# Security Audit

The non-secret, non-dependency security pass. Secret leaks live in
`secret-scan`; CVE bumps live in `dependency-audit`. This skill covers
SQL injection, CORS, missing schema validation, cookie attributes,
file-upload safety, dangerous Python builtins, path traversal, SSRF,
and unsanitized frontend HTML.

## Stack Assumptions

This skill ships with defaults tuned for a Python (FastAPI / SQLAlchemy)
+ TypeScript (Next.js / React) monorepo with backend code under
`backend/` and frontend under `frontend/`. Adapt the `rg` paths in
each sweep to your repo's layout — the patterns themselves apply to
any stack.

Common conventions the skill checks against:

- FastAPI: CORS via `CORSMiddleware`, request bodies via Pydantic
  schemas.
- SQLAlchemy: raw SQL through `db.execute(text("..."))` — every such
  call must use bound parameters, never f-string interpolation.
- File uploads: any endpoint accepting an `UploadFile` is on the audit
  list.

## When to use this skill

- Any time a new endpoint, schema, or DB query was added.
- As the third step of `audit-all` (after `secret-scan` and
  `dependency-audit`).
- Before promoting `staging` → `main` on any release touching the API
  surface.

## Workflow

```
Task progress:
- [ ] Sweep 1: SQL injection across backend/services/ and backend/routers/
- [ ] Sweep 2: CORS configuration in backend/api_server.py
- [ ] Sweep 3: Pydantic schemas on every FastAPI route that takes a body
- [ ] Sweep 4: Cookie attributes (httpOnly, Secure, SameSite)
- [ ] Sweep 5: File-upload handlers (size cap, MIME whitelist, filename sanitization)
- [ ] Sweep 6: Dangerous Python builtins (pickle.load, eval, exec, shell=True)
- [ ] Sweep 7: Path-traversal sinks (open(), os.path.join with user input)
- [ ] Sweep 8: SSRF-prone outbound HTTP (httpx, aiohttp, requests with user-supplied URLs)
- [ ] Sweep 9: HTML/Markdown rendering on the frontend (dangerouslySetInnerHTML, raw HTML)
- [ ] Apply auto-fixes; surface the rest by severity
- [ ] Print findings categorized by severity
```

### Sweep 1 — SQL injection

```bash
rg -n 'execute\(.*text\(.*\bf["\']' backend/ --type py
rg -n 'execute\(.*text\(["\'].*\+' backend/ --type py
rg -n 'execute\(.*text\(["\'].*\{' backend/ --type py
```

Any hit on `text(f"...")`, `text("..." + var)`, or `text("..." % var)`
is a CRITICAL finding. Auto-fix: rewrite to bound parameters.

```python
# Before (CRITICAL):
db.execute(text(f"SELECT * FROM users WHERE id = {user_id}"))

# After (auto-fix):
db.execute(text("SELECT * FROM users WHERE id = :user_id"), {"user_id": user_id})
```

Also flag ORM `.filter(...)` calls that interpolate strings —
`.filter(text(f"name = '{name}'"))` is the same risk in disguise.

### Sweep 2 — CORS

Read `[backend/api_server.py](backend/api_server.py)` and verify:

- `ALLOWED_ORIGINS` is a non-wildcard list (`["*"]` is CRITICAL in
  production if combined with `allow_credentials=True`).
- `allow_methods` and `allow_headers` are restrictive when credentials
  are allowed (FastAPI default is permissive; pin them).
- `allow_credentials=True` AND `allow_origins=["*"]` together: CRITICAL.
  The browser will reject the response but the misconfig signals intent
  to allow cross-site cookie exposure.

Auto-fix: replace `["*"]` with an explicit env-driven list. Surface for
human review if the env layer is unclear.

### Sweep 3 — Pydantic schemas

For each route, the body parameter must be a Pydantic model from
`[backend/schemas/](backend/schemas/)` — never a raw `dict` or `Any`.

```bash
rg -n 'async def \w+\(.*: dict' backend/routers/ --type py
rg -n 'async def \w+\(.*: Any' backend/routers/ --type py
rg -n 'async def \w+\(.*= Body\(\.\.\.\)' backend/routers/ --type py
```

Any hit on `: dict`, `: Any`, or bare `Body(...)` is HIGH severity.
Auto-fix: cannot synthesize a schema (the agent doesn't know intended
shape), so this is flagged with the recommendation `propose Pydantic
model for <route>`.

### Sweep 4 — Cookie attributes

```bash
rg -n 'set_cookie\(' backend/ --type py
rg -n 'response\.set_cookie' backend/ --type py
```

For each cookie set:

- `httponly=True` MUST be set for any cookie holding auth state
  (session, CSRF token, JWT). CRITICAL if missing.
- `secure=True` MUST be set for any cookie used in production. The
  skill is conservative — flags MEDIUM if not set, since local dev
  needs `secure=False`. Recommend toggling on env.
- `samesite="lax"` (or `"strict"`) MUST be set. Default browser
  behavior varies. HIGH if missing.

### Sweep 5 — File uploads

```bash
rg -n 'UploadFile' backend/ --type py
```

For each handler accepting an `UploadFile`:

- A size cap MUST be enforced — either via `Content-Length` rejection
  or by streaming with a max-bytes check. Missing: HIGH.
- A MIME whitelist (or magic-byte check via `python-magic` /
  `filetype`) MUST be enforced — trusting the client-provided
  `content_type` is insufficient. Missing: HIGH.
- The filename MUST be sanitized before any disk write — use
  `secure_filename` (werkzeug) or `Path(name).name` + a UUID prefix.
  Missing: CRITICAL (path traversal risk).

Auto-fix only the filename-sanitization case (low-risk wrapper). The
size cap and MIME whitelist need the agent to know acceptable values
for the specific endpoint, so they're flagged with proposed patches.

### Sweep 6 — Dangerous builtins

```bash
rg -n '\bpickle\.load' backend/ --type py
rg -n '\beval\(' backend/ --type py
rg -n '\bexec\(' backend/ --type py
rg -n 'subprocess\.(?:run|call|Popen).*shell=True' backend/ --type py
rg -n 'yaml\.load\(' backend/ --type py  # without Loader= is CVE-2017-18342
rg -n 'os\.system\(' backend/ --type py
```

Each hit is reviewed:

- `pickle.load` on attacker-controlled bytes: CRITICAL (RCE). On a
  trusted-only path (e.g. an in-process cache file the agent itself
  wrote): MEDIUM with a comment requested.
- `eval` / `exec`: CRITICAL unless behind a comment explaining the
  trust boundary. Auto-fix: never. Always human review.
- `subprocess(... shell=True)` with any interpolated variable:
  CRITICAL. Auto-fix: switch to list-arg form `subprocess.run(["cmd",
  arg])` if the variable is bound to a simple name.
- `yaml.load(x)` without `Loader=yaml.SafeLoader`: HIGH. Auto-fix:
  add `Loader=yaml.SafeLoader` (or switch to `yaml.safe_load`).
- `os.system(x)`: HIGH — replace with `subprocess.run([...])`.

### Sweep 7 — Path traversal

```bash
rg -n 'open\(.*request' backend/ --type py
rg -n 'os\.path\.join\(.*request' backend/ --type py
rg -n 'Path\(.*request' backend/ --type py
```

Any `open` / `Path` / `os.path.join` that incorporates a value coming
from the request (path param, query string, body field) without a
`.resolve().relative_to(SAFE_ROOT)` check is HIGH severity. The
classic exploit is `../../etc/passwd` in a `download/{file}` route.

### Sweep 8 — SSRF

```bash
rg -n '(httpx|aiohttp|requests)\.(get|post|put|delete|head|patch)\(' backend/ --type py
```

For each outbound HTTP call, check whether the URL is constructed from
a request-supplied value. If so, MUST be:

- Constrained to an allow-list of hosts, OR
- Resolved through DNS and the resolved IP must be in a public range
  (block `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`,
  `192.168.0.0/16`, `169.254.0.0/16`, `0.0.0.0/8`, `::1`).

Missing: HIGH. Auto-fix: cannot. Flag with the recommended
allow-list approach.

### Sweep 9 — Frontend HTML rendering

```bash
rg -n 'dangerouslySetInnerHTML' frontend/ --type tsx
rg -n '\bunsafe\b' frontend/components/ --type tsx
```

Each `dangerouslySetInnerHTML` usage must source from sanitized HTML
(e.g. via `DOMPurify`, `rehype-sanitize`, or trusted server-rendered
markdown). Unsanitized: HIGH (XSS). Auto-fix: cannot — needs the agent
to know which renderer the project uses.

The MarkdownRenderer in `[frontend/components/research/markdown-renderer.tsx](frontend/components/research/markdown-renderer.tsx)`
should use a sanitizing pipeline; flag if it doesn't.

## Severity Classes

| Severity | Examples |
|----------|----------|
| CRITICAL | SQL injection (interpolated `text()`); CORS `*` with credentials; `pickle.load` on attacker-controlled bytes; `eval` / `exec` on request data; `subprocess(shell=True)` with request data; missing filename sanitization on upload |
| HIGH | Missing Pydantic schema on a body route; missing `samesite` on auth cookie; missing upload size cap or MIME whitelist; path traversal sink; unconstrained outbound HTTP (SSRF); unsanitized `dangerouslySetInnerHTML` |
| MEDIUM | Missing `secure=True` on cookie (dev only acceptable, prod must set); `pickle.load` on trusted bytes without a comment; `yaml.load` without `SafeLoader` |
| LOW | Permissive `allow_methods`/`allow_headers` in dev-only paths |

The skill hard-stops on ANY unresolved CRITICAL finding.

## Hard Rules (never break)

- **Never auto-fix `eval` / `exec` / `pickle.load`** — the trust
  boundary is human-reviewed.
- **Never auto-fix a size cap or MIME whitelist** — the agent doesn't
  know the intended caps without product context.
- **Never widen CORS** — only narrow it. If the audit cannot determine
  the right allow-list, flag and stop.
- **Never auto-add `import os` for env-var swaps** without confirming
  the project already uses `os.getenv` for similar config — keep the
  style consistent with neighboring files.
- **Never push, commit, or PR.** Audit and rewrite the working tree
  only. Hand off to `review-and-ship-to-staging`.

## Trigger Examples

- "audit security"
- "security audit and fix"
- "find security issues"
- "run the security-audit skill"

## Output Template

End the run with:

```
Security-audit summary

Sweep 1 — SQL injection                : <N> hits  (auto-fixed: <K>)
Sweep 2 — CORS                         : <pass|fail>  (details)
Sweep 3 — Pydantic schemas             : <N> raw-dict / Any / Body routes
Sweep 4 — Cookie attributes            : <N> missing httpOnly/secure/samesite
Sweep 5 — File uploads                 : <N> missing size cap / MIME / sanitize
Sweep 6 — Dangerous builtins           : <N> pickle, <N> eval/exec, <N> shell=True
Sweep 7 — Path traversal               : <N> sinks
Sweep 8 — SSRF                         : <N> unconstrained outbound calls
Sweep 9 — Frontend HTML                : <N> dangerouslySetInnerHTML usages

Findings by severity
- CRITICAL: <N>   (auto-fixed: <K>, manual: <N-K>)
  • <file>:<line>  <finding>
- HIGH:     <N>   (auto-fixed: <K>, manual: <N-K>)
- MEDIUM:   <N>
- LOW:      <N>

Hard-stop:  yes | no  (yes if any unresolved CRITICAL)

Next step: review CRITICAL findings, then run `review-and-ship-to-staging`.
```
