---
name: auth-flow-audit
description: >-
  Audit every FastAPI route in `backend/routers/` and every Next.js route in
  `frontend/app/` for authentication and authorization gaps: unguarded
  endpoints, IDOR on user-owned resources (`conversation_id`,
  `message_id`, `query_id`, `feedback_id`), admin endpoints missing
  `is_admin` checks, guest-flow escalation, JWT misconfiguration, and
  password-flow user-enumeration leaks. Auto-fixes the safe ones. Use when
  the user says "audit auth", "audit authentication flow", "fix auth
  issues", or "auth audit".
---

# Auth Flow Audit

The highest-blast-radius audit in the system. Any miss here means a user
can read or mutate someone else's data — or, worse, escalate to admin.
Runs across the entire `backend/routers/` tree, the auth services in
`backend/services/`, and the protected pages in `frontend/app/`.

## When to use this skill

- Any time a new router or endpoint was added in this chat.
- As the third step of `audit-all` (after `secret-scan` and
  `dependency-audit`).
- Before promoting `staging` to `main` on any release that touches the
  auth, billing, or admin surfaces.

## Stack-Specific Context

Pin these facts when reading the codebase:

- The JWT helpers live in `[backend/auth.py](backend/auth.py)`
  (`create_access_token`, `decode_access_token`, `create_refresh_token`,
  `refresh_access_token`).
- The FastAPI dependency that resolves the current user from the bearer
  token is `get_current_user` in
  `[backend/services/auth_service.py](backend/services/auth_service.py)`.
- The "require an authenticated user" gate is `require_auth` (used like
  `user: User = Depends(require_auth)` — see
  `[backend/routers/auth.py](backend/routers/auth.py)`).
- Admin routes live under `[backend/routers/admin/](backend/routers/admin/)`
  (`analytics.py`, `reports.py`, `settings.py`, `users.py`) and must
  ALL gate on a `current_user.is_admin` check, NOT just `require_auth`.
- The guest flow uses anonymous IDs and has its own router/service:
  `[backend/routers/guest_conversation.py](backend/routers/guest_conversation.py)`,
  `[backend/services/guest_id.py](backend/services/guest_id.py)`,
  `[backend/services/guest_lifecycle.py](backend/services/guest_lifecycle.py)`.
- Frontend auth hooks: `useAuth` and `useAuthGuard` in
  `[frontend/lib/hooks/](frontend/lib/hooks/)`.

## Workflow

```
Task progress:
- [ ] Enumerate every backend route definition (`@router.get/post/put/delete/patch`)
- [ ] Classify each: public | requires-auth | requires-admin | requires-ownership
- [ ] Check (a) the dependency on the route matches its required class
- [ ] Check (b) IDOR — every user-owned resource id in the path/body is verified against current_user.id
- [ ] Check (c) admin routes — verify is_admin assertion
- [ ] Check (d) guest flow — verify no guest endpoint accepts an authenticated user_id
- [ ] Check (e) JWT — expiry set, refresh rotation works, secret from env (not hardcoded)
- [ ] Check (f) signup / password reset — rate limited, no user-enumeration
- [ ] Check (g) frontend — protected pages gate on useAuthGuard, no JWT in localStorage
- [ ] Apply auto-fixes (add Depends, add ownership check, add admin gate)
- [ ] Print findings categorized by severity
```

### Step 1. Enumerate routes

```bash
rg -n '^(async def|def) \w+\s*\(' backend/routers/ --type py | head -200
rg -n '@(router|app)\.(get|post|put|delete|patch)\(' backend/routers/ --type py
```

For each match, record: file, line, HTTP method, path template, function
signature.

### Step 2. Classify expected access level

Use the path/method prefix as the heuristic — the agent then verifies
the dependency matches. Default classification rules:

| Pattern | Expected class |
|---------|----------------|
| `backend/routers/admin/*` (any method) | requires-admin |
| `/conversations/{conversation_id}`, `/messages/{message_id}`, `/queries/{query_id}`, `/feedback/{feedback_id}` | requires-ownership |
| `/auth/login`, `/auth/signup`, `/auth/verify-email`, `/auth/forgot-password`, `/auth/google/callback`, `/health`, `/api/system/status` | public |
| `/guest/*` | public-guest (no JWT, validated guest cookie) |
| Everything else | requires-auth |

If a route's classification is ambiguous, the skill flags it for human
review rather than auto-fixing — getting the class wrong is the worst
outcome.

### Step 3. Check (a) — dependency matches class

For every route:

- **requires-auth**: signature MUST include `Depends(require_auth)` (or
  `Depends(get_current_user)` if the route handles unauthenticated users
  separately). A missing `Depends(...)` is a CRITICAL finding.
- **requires-admin**: signature MUST include both `Depends(require_auth)`
  AND a check `if not current_user.is_admin: raise HTTPException(403)`
  in the body — OR a `Depends(require_admin)` if a helper exists.
- **public**: signature must NOT depend on `require_auth` (otherwise
  legitimate users get blocked). Annotate intent with `# public:
  <reason>` so the next auditor knows.

Auto-fix: insert `user: User = Depends(require_auth)` (and the matching
import from `backend.routers.auth`) for any requires-auth route missing
it. Do NOT auto-fix requires-admin gaps — add a `# TODO: ADMIN-AUDIT`
comment and surface in the report. Admin gaps are reviewed by a human
because the wrong fix (e.g. checking `is_admin` after a `return`) is
worse than no fix.

### Step 4. Check (b) — IDOR on user-owned resources

For every route whose path includes `{conversation_id}`,
`{message_id}`, `{query_id}`, `{feedback_id}`, `{report_id}`, the body
of the handler MUST contain a check shaped like:

```python
resource = db.query(Conversation).filter_by(id=conversation_id).first()
if not resource:
    raise HTTPException(404)
if resource.user_id != current_user.id and not current_user.is_admin:
    raise HTTPException(404)  # 404 NOT 403 — don't leak existence
```

The skill confirms this by reading the function body and grepping for
`.user_id ==`, `.owner_id ==`, or `is_admin`. If absent, it is a CRITICAL
finding.

Auto-fix: insert the ownership-or-admin check immediately after the
resource-fetch. If the agent cannot find the resource-fetch line
unambiguously, surface for human review.

Hard rule: respond `404` on IDOR violation, not `403` — a `403` confirms
the resource exists to an attacker probing IDs.

### Step 5. Check (c) — admin gates

For every file under `[backend/routers/admin/](backend/routers/admin/)`,
EVERY route must perform an admin check. Audit by:

```bash
rg -n '@router\.(get|post|put|delete|patch)' backend/routers/admin/
```

For each hit, confirm the corresponding handler body contains an
`is_admin` check. Missing checks are CRITICAL.

### Step 6. Check (d) — guest flow

Guests have anonymous IDs (cookies / headers) and must not be able to
read or mutate authenticated-user data. Check:

- Guest endpoints (`[backend/routers/guest_conversation.py](backend/routers/guest_conversation.py)`,
  `[backend/routers/guest_events.py](backend/routers/guest_events.py)`)
  must NOT accept a `user_id` body field or path param — guests are
  scoped purely by their guest cookie.
- Authenticated endpoints must NOT accept a guest cookie as a substitute
  for a bearer token.
- Credit accounting: a guest cannot consume an authenticated user's
  credits, even by spoofing both identifiers. Look in
  `[backend/services/payment_service.py](backend/services/payment_service.py)`
  and `[backend/services/credit_pricing.py](backend/services/credit_pricing.py)`
  for the deduction logic and confirm it keys off exactly one identity.

### Step 7. Check (e) — JWT correctness

Inspect `[backend/auth.py](backend/auth.py)`:

- `SECRET_KEY` (and the refresh-token secret, if separate) MUST come from
  `os.getenv(...)` — never a literal in source.
- `create_access_token` MUST set an `exp` claim (default 30 min or less
  is healthy; over 24 h is a flag).
- `create_refresh_token` MUST use a distinct secret OR a distinct key id
  so an access token cannot be replayed as a refresh token.
- `decode_access_token` MUST call `jwt.decode(..., algorithms=["HS256"])`
  with an explicit allow-list — never `algorithms=jwt.get_unverified_header(token)['alg']`
  (the classic algo-confusion attack).
- Token revocation: if `users.password_changed_at` exists in the model,
  tokens issued before that timestamp must be rejected. If it does not
  exist, surface as a "should add" recommendation, not a fix.

### Step 8. Check (f) — signup / password reset

- `/auth/signup` and `/auth/login` MUST be rate-limited (look for
  `slowapi`, `fastapi-limiter`, or a custom decorator). Missing rate
  limiting is a HIGH finding (not CRITICAL — it's an availability
  concern, not a confidentiality breach).
- `/auth/forgot-password`: response MUST be identical whether the email
  exists or not. The skill greps for `if user:` branches that return
  different messages and flags them.

### Step 9. Check (g) — frontend

- Every page under `frontend/app/` that calls a hook from
  `[frontend/lib/hooks/](frontend/lib/hooks/)` that hits an authenticated
  endpoint MUST be wrapped in `useAuthGuard()` (or equivalent server-side
  redirect). Pages that show authenticated data without an auth guard
  briefly leak loading state to unauthenticated users.
- JWTs MUST NOT be written to `localStorage` (XSS exposure). The skill
  greps `frontend/` for `localStorage.setItem.*token` and flags. Tokens
  belong in `httpOnly` cookies; if you must use `localStorage`, the
  finding stays in the report with a rationale request.

### Step 10. Apply auto-fixes, then re-audit

After fixes, re-run Steps 3–9. Anything still flagged is for human
review — the skill never auto-fixes the same finding twice in one pass.

## Severity Classes

| Severity | Examples |
|----------|----------|
| CRITICAL | Missing `Depends(require_auth)` on a non-public route; missing IDOR check; admin route without `is_admin` gate; JWT secret hardcoded; algo-confusion (`algorithms=...` unbound) |
| HIGH | Missing rate-limit on signup/login/forgot-password; user-enumeration on password reset; JWT exp missing or > 24h; refresh token sharing access secret |
| MEDIUM | JWT in `localStorage`; protected page missing `useAuthGuard`; admin endpoint returns 403 instead of 404 on IDOR |
| LOW | Missing `# public:` annotation on a deliberately public route |

The skill hard-stops on ANY unresolved CRITICAL finding — `audit-all`
relies on that signal.

## Hard Rules (never break)

- **Never auto-fix an admin-gate finding.** Adding the wrong `is_admin`
  check (placed after the database write, or checking the wrong field)
  is worse than no check. Flag for human review.
- **Never auto-fix JWT secret logic.** Secret rotation, algo allow-list,
  revocation policy — all human review.
- **Never downgrade severity** without a written reason in the report.
- **Never push, commit, or PR.** Audit and rewrite the working tree
  only. Hand off to `review-and-ship-to-staging`.
- **IDOR fixes always respond 404, never 403.** A 403 leaks resource
  existence to a probing attacker.
- **Never modify `[backend/auth.py](backend/auth.py)` JWT logic
  autonomously.** Only flag findings and propose patches in the report.
- **Never silently exclude a route** from the audit. If a route is
  deliberately public, the skill adds a `# public: <reason>` annotation
  rather than removing it from the audit list.

## Trigger Examples

- "audit auth"
- "audit authentication flow"
- "fix auth issues"
- "auth audit"
- "run the auth-flow-audit skill"

## Output Template

End the run with:

```
Auth-flow-audit summary
- Routes enumerated:           <N>
  - Classified public:         <N>
  - Classified requires-auth:  <N>
  - Classified requires-admin: <N>
  - Classified ownership:      <N>
  - Ambiguous (human review):  <N>

Findings by severity
- CRITICAL: <N>   (auto-fixed: <K>, manual: <N-K>)
  • <file>:<line>  <route>  <finding>
  • ...
- HIGH:     <N>   (auto-fixed: <K>, manual: <N-K>)
- MEDIUM:   <N>
- LOW:      <N>

JWT review (from backend/auth.py)
- SECRET_KEY from env:         pass | fail
- exp claim set:               pass | fail
- Refresh-token isolated:      pass | fail
- Algorithm allow-list bound:  pass | fail

Frontend review
- Pages missing useAuthGuard:  <N>
- JWTs in localStorage:        <N>

Hard-stop:                     yes | no  (yes if any CRITICAL unresolved)

Next step: review the CRITICAL findings above, then run `review-and-ship-to-staging`.
```
