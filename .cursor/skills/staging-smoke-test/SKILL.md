---
name: staging-smoke-test
description: >-
  Read-only health check against the live staging (or production)
  deploy. Hits `GET /health`, `GET /api/system/status`, optionally
  runs one canned research query against the chat endpoint with a
  test account and verifies the WebSocket emits at least `starting`
  and `completed` progress stages. Times out after 120s and reports
  per-step pass/fail. Designed to run as the final step after
  `promote-staging-to-main` — if it fails, the user can immediately
  invoke `rollback-release`. Use when the user says "smoke test
  staging", "run smoke test", or "verify staging is healthy".
---

# Staging Smoke Test

The "is the deploy healthy?" skill. Hits the live endpoint with the
minimum set of requests that prove the backend is reachable, the
auth is wired, the LLM provider is responding, and the WebSocket
progress channel is alive.

## When to use this skill

- Immediately after `promote-staging-to-main` (or after any staging
  deploy event).
- Anytime production is suspected unhealthy.
- Periodically as a synthetic health check (cron-style).

## Stack-Specific Context

- Backend health endpoint: `GET /health` returns `200 {"status": "ok"}`
  when the backend container is up.
- Backend status endpoint: `GET /api/system/status` returns deeper
  health (DB connection, vector store reachable, LLM provider keys
  loaded).
- WebSocket progress: `/ws/progress/{session_id}` emits stage events
  for a research run; the agreed minimum lifecycle is `starting` →
  ... → `completed` (see the stage list in
  `[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)`).
- Research entry: `POST /api/research` (or wherever the chat skill
  fires from `useChatResearch` — verify the exact path before
  hitting it).

## Configuration

Required (the skill stops early if any are missing):

- `STAGING_URL` (e.g. `https://staging.casescout.ai`) — read from
  env, NEVER hardcoded.
- `SMOKE_TEST_EMAIL` and `SMOKE_TEST_PASSWORD` — credentials for a
  dedicated test account (NOT a real user). Read from env.

Optional:

- `SMOKE_TEST_QUERY` (default: `"What is the test of reasonableness
  in s. 8 Charter?"` — a short, deterministic legal question that
  produces a quick research response and exercises the full
  pipeline).
- `SMOKE_TIMEOUT` (default: 120 seconds for the WebSocket smoke).

## Workflow

```
Task progress:
- [ ] Verify required env vars present (STAGING_URL + credentials)
- [ ] Step 1: GET /health  (expect 200, < 2s)
- [ ] Step 2: GET /api/system/status  (expect 200, all sub-checks pass)
- [ ] Step 3: POST /api/auth/login  (expect 200, JWT returned)
- [ ] Step 4: POST /api/research with the canned query (expect 202 + session_id)
- [ ] Step 5: open ws://STAGING_URL/ws/progress/{session_id}; wait for `starting` then `completed`
- [ ] Step 6: GET /api/conversations/{id}/messages  (verify the assistant message persisted)
- [ ] Print per-step pass/fail
```

### Step 1. `GET /health`

```bash
curl -sS --max-time 5 -w '\n%{http_code} %{time_total}\n' "${STAGING_URL}/health"
```

- Expect HTTP 200 and `{"status": "ok"}`-shaped body.
- Time-to-first-byte > 2s is a yellow flag (the container might be
  cold-starting) — surface as a warning but pass the step.

### Step 2. `GET /api/system/status`

```bash
curl -sS --max-time 10 "${STAGING_URL}/api/system/status"
```

Parse the JSON. Pass requires each sub-check present in the response
to read `ok`/`healthy`/`true`. The skill is tolerant of unknown
sub-checks (forward-compat for new ones) — if a sub-check exists, it
must be passing.

If any sub-check is failing, capture the full response in the report
and stop the smoke test (downstream steps will probably fail too
without diagnostic value).

### Step 3. Login

```bash
curl -sS --max-time 10 \
  -H 'Content-Type: application/json' \
  -d "{\"email\":\"${SMOKE_TEST_EMAIL}\",\"password\":\"${SMOKE_TEST_PASSWORD}\"}" \
  "${STAGING_URL}/api/auth/login"
```

Extract the access token. Failure here is CRITICAL — either auth is
broken or the test account is locked out / disabled.

### Step 4. Submit research query

```bash
curl -sS --max-time 15 \
  -H "Authorization: Bearer ${TOKEN}" \
  -H 'Content-Type: application/json' \
  -d "{\"query\":\"${SMOKE_TEST_QUERY}\"}" \
  "${STAGING_URL}/api/research"
```

Expect 202 (or 200) with a `session_id` in the body. Save it.

### Step 5. WebSocket progress

Use `websocat` if available; otherwise a 30-line Python script with
`websockets`:

```python
import asyncio, json, websockets, os, sys

async def smoke(url, token, session_id, timeout=120):
    seen_stages = set()
    async with websockets.connect(f"{url}/ws/progress/{session_id}",
                                  extra_headers={"Authorization": f"Bearer {token}"}) as ws:
        async def reader():
            async for msg in ws:
                data = json.loads(msg)
                stage = data.get("stage") or data.get("event")
                if stage:
                    seen_stages.add(stage)
                if stage == "completed":
                    return
        await asyncio.wait_for(reader(), timeout=timeout)
    return seen_stages

if __name__ == "__main__":
    stages = asyncio.run(smoke(*sys.argv[1:]))
    assert "starting" in stages, f"missing starting; saw {stages}"
    assert "completed" in stages, f"missing completed; saw {stages}"
    print(json.dumps(sorted(stages)))
```

Pass criteria:
- Connection opens within 5s.
- At least one event arrives within 10s (no event for 10s = LLM
  provider not responding or job not picked up).
- `completed` event arrives before `SMOKE_TIMEOUT`.

The skill does NOT enforce a specific sequence of intermediate stages
(those are tuned in `[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)`
and may change). Only the bookends are required.

### Step 6. Verify persistence

After the WebSocket reports `completed`, the assistant's response
must be readable from the conversations API:

```bash
curl -sS --max-time 10 \
  -H "Authorization: Bearer ${TOKEN}" \
  "${STAGING_URL}/api/conversations/${CONVERSATION_ID}/messages"
```

Expect at least one assistant message tied to the session. Failure
here means the WebSocket fired but the DB write didn't land — a
silent data loss bug.

## What this skill does NOT do

- Does NOT deploy or restart anything.
- Does NOT modify the database.
- Does NOT delete the smoke conversation it creates (use a
  scheduled cleanup if conversation pile-up becomes an issue, or
  add a "delete after smoke" step the user explicitly opts into).
- Does NOT exercise the admin surface (separate skill if needed).
- Does NOT load-test (one request at a time).

## Hard Rules (never break)

- **Never hardcode `STAGING_URL` or credentials.** Always read from
  env. If the env vars are missing, stop with a clear message.
- **Never use a real customer account.** A dedicated smoke-test
  account is required.
- **Never run against `localhost`** if the user said "staging" or
  "production" — the skill is for live deploys. If they want a
  local smoke, that's a different skill (not built).
- **Never silently retry** a failed step. One attempt per step,
  report the failure, stop the cascade.
- **Never log the JWT, password, or response body containing PII.**
  Log status codes, timings, and stage-name sets only.

## Trigger Examples

- "smoke test staging"
- "run smoke test"
- "verify staging is healthy"
- "is production up?"
- "run the staging-smoke-test skill"

## Output Template

End the run with:

```
Staging-smoke-test summary

Target:                     STAGING_URL=${STAGING_URL}
Smoke account:              ${SMOKE_TEST_EMAIL}
Canned query:               "${SMOKE_TEST_QUERY}"

Step 1 — /health             : pass  (HTTP 200, <Xms>)
Step 2 — /api/system/status  : pass  (all sub-checks)  | fail  (<which>)
Step 3 — /api/auth/login     : pass  (token obtained)  | fail  (HTTP <code>)
Step 4 — /api/research       : pass  (session_id obtained)
Step 5 — WebSocket progress  : pass  (starting → completed in <Xs>; stages seen: <list>)
Step 6 — Message persisted   : pass  (assistant message id <X>)

Overall:                     HEALTHY | DEGRADED | DOWN

Failure detail (if any):
  <step that failed, status code, raw error>

Next step:
  - If DOWN or DEGRADED: investigate, and if necessary run `rollback-release` to v<prev>.
  - If HEALTHY: nothing more required.
```
