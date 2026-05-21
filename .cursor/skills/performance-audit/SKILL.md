---
name: performance-audit
description: >-
  Audit the codebase for the performance pitfalls that bite under
  scale: LLM calls without timeouts, blocking `time.sleep` in async
  paths, N+1 SQLAlchemy queries in routers, unstable `useEffect`
  dependencies on the frontend, and missing `staleTime` / `gcTime` on
  React Query hooks. Soft-fail by default — most findings are
  informational, not blockers. Use when the user says "audit
  performance", "find slow endpoints", or "performance check".
---

# Performance Audit

The lightest-touch audit in Tier 1. Performance issues rarely block a
ship — they degrade gracefully — so this skill's job is to put a list
of concrete asks on the user's desk, not to gate the pipeline.

The one exception: any unwrapped LLM call in
`[backend/services/nodes/](backend/services/nodes/)` is treated as
CRITICAL because a hung provider takes the entire research pipeline
down silently. Per `[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)`
("Long-running backend LLM calls MUST have timeouts"), every LLM call
must be wrapped in `asyncio.wait_for`.

## When to use this skill

- As the seventh and final step of `audit-all`.
- After adding a new PocketFlow node, new router endpoint, or new
  React Query hook.

## Workflow

```
Task progress:
- [ ] Sweep 1: LLM call timeouts in backend/services/nodes/
- [ ] Sweep 2: blocking time.sleep in any async path
- [ ] Sweep 3: N+1 SQLAlchemy queries in backend/routers/ and backend/services/
- [ ] Sweep 4: unstable useEffect deps in frontend/components/ and frontend/lib/hooks/
- [ ] Sweep 5: React Query hooks missing staleTime / gcTime
- [ ] Sweep 6: large in-memory accumulations (e.g. building a 100k-row list before yielding)
- [ ] Sweep 7: missing pagination on listing endpoints
- [ ] Apply auto-fixes only where canonical
- [ ] Print findings; do NOT hard-stop (this skill is soft-fail)
```

### Sweep 1 — LLM call timeouts

```bash
rg -nB2 'call_llm_async\(|client\.(chat|messages|models)\.\w+\(|generate_content_async\(' backend/services/nodes/ --type py
```

For each match, look upward (within the same `async def`) for an
`asyncio.wait_for(...` wrapper. Missing wrapper = CRITICAL.

Auto-fix only for nodes with a known timeout (see the rule entry in
`[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)`):

- `aggregation_agent.py`: 180s
- `citation_enrichment.py`: 90s
- Anything else: 60s default (flag as MEDIUM "agent picked default,
  please confirm")

Auto-fix pattern (wrap the existing await):

```python
response = await asyncio.wait_for(call_llm_async(prompt), timeout=180)
```

Add `import asyncio` if missing.

For citation enrichment specifically, the existing behavior on timeout
is to skip gracefully — the auto-fix MUST preserve that. Use a
`try/except asyncio.TimeoutError: return None` block.

### Sweep 2 — `time.sleep` in async paths

```bash
rg -n 'time\.sleep\(' backend/ --type py
```

For each hit, check whether the enclosing function is `async def`. If
yes, this is HIGH severity — blocks the whole event loop.

Auto-fix: replace `time.sleep(x)` → `await asyncio.sleep(x)` and add
`import asyncio` if missing.

Hits in sync functions are LOW (they may be intentional rate-limit
backoffs in non-async scripts under `[backend/scripts/](backend/scripts/)`).

### Sweep 3 — N+1 queries

For each `for` loop in `backend/routers/` and `backend/services/`,
check whether the body contains a `db.query(...)` or `db.execute(...)`.
If yes, it is a N+1 candidate.

```bash
rg -nB1 -A5 'for \w+ in ' backend/routers/ backend/services/ --type py | rg -B5 '(db\.query|db\.execute)\('
```

Surface as MEDIUM with the recommended `selectinload` /
`joinedload` / `.in_(ids)` rewrite. Don't auto-fix — the right
eager-load strategy is query-shape-specific.

### Sweep 4 — unstable `useEffect` deps

```bash
rg -nB1 -A3 'useEffect\(' frontend/components/ frontend/lib/hooks/ --type tsx --type ts
```

For each hook, inspect the dependency array. Flag if it contains:

- An object literal (`{...}`) directly inline — new reference every
  render → infinite re-fire.
- A function reference that isn't memoized via `useCallback` and isn't
  defined outside the component.
- An empty `[]` paired with a body that references variables from the
  component scope (stale-closure risk).

Hits are MEDIUM. Auto-fix only the simplest cases (e.g. inline empty
object literal `{}` that's obviously a default — replace with a
module-level `const EMPTY = {} as const`).

The user has explicit React rules in
`[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)` —
particularly the callback-ref pattern for conditionally-rendered DOM
elements (see the "Chat scroll listener" entry). Findings that violate
that rule are HIGH because they're already-known patterns.

### Sweep 5 — React Query staleness

```bash
rg -n 'useQuery\(' frontend/lib/hooks/ --type ts --type tsx
```

For each `useQuery` call, check the options object for `staleTime`
and `gcTime`. Hooks that hit user-mutable data (conversations,
billing, profile) should typically declare both. Missing both: LOW —
just a polish recommendation.

For hooks listed in
`[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)` (e.g.
`useChatResearch`, `useConversationHistory`, `useBillingInfo`) the
public API must remain stable, so the skill never modifies the hook
signature — only the internal options.

### Sweep 6 — Large in-memory accumulations

Look for the anti-pattern:

```python
all_rows = []
for chunk in stream:
    all_rows.extend(chunk)
return all_rows
```

When the iteration is over a paginated source. Surface as MEDIUM with
the recommended `yield from` (generator) or `AsyncIterator` rewrite.
Auto-fix: never (changes the call-site contract).

### Sweep 7 — Missing pagination

For each `GET` route that returns a list (FastAPI handler that returns
`List[Schema]`), check whether `limit` and `offset` (or `page` and
`page_size`) parameters are declared. Missing pagination on a list
endpoint is MEDIUM — fine in dev, painful in prod.

Auto-fix: cannot synthesize meaningful defaults. Surface with the
recommended `limit: int = Query(50, le=200), offset: int = Query(0,
ge=0)` shape.

## Severity Classes

| Severity | Examples |
|----------|----------|
| CRITICAL | LLM call site missing `asyncio.wait_for` wrap in a research pipeline node |
| HIGH | `time.sleep` in an `async def`; React `useEffect` pattern that the project's own rules call out as bug-prone |
| MEDIUM | N+1 query candidates; unstable `useEffect` deps; large in-memory accumulations; missing pagination |
| LOW | Missing React Query `staleTime` / `gcTime`; `time.sleep` in non-async scripts |

The skill is **soft-fail** — it reports CRITICAL findings but does not
hard-stop `audit-all`. The user decides whether to ship with known
performance debt.

## Hard Rules (never break)

- **Never modify a published hook signature.** The hooks listed in
  `[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)` as
  "keep API stable" are read-only at the signature level — internal
  options can be tuned, parameters cannot be added.
- **Never auto-add a timeout** to an LLM call without a canonical
  value (per-node policy in `.cursor/rules/frontend.mdc` or matching
  a neighboring node in the same family). Guessing a timeout is worse
  than leaving the audit finding visible.
- **Never auto-rewrite N+1 patterns.** The right eager-load strategy
  depends on the query shape — flag with a recommendation, don't
  apply.
- **Never push, commit, or PR.** Modifies the working tree only.
  Hand off to `review-and-ship-to-staging`.

## Trigger Examples

- "audit performance"
- "find slow endpoints"
- "performance check"
- "run the performance-audit skill"

## Output Template

End the run with:

```
Performance-audit summary

Sweep 1 — LLM call timeouts        : <N hits>  missing wrap → <K> auto-fixed
Sweep 2 — time.sleep in async      : <N hits>  → <K> auto-fixed
Sweep 3 — N+1 query candidates     : <N hits>  (manual review)
Sweep 4 — unstable useEffect deps  : <N hits>  → <K> auto-fixed
Sweep 5 — React Query staleness    : <N hooks missing staleTime/gcTime>
Sweep 6 — large accumulations      : <N hits>  (manual review)
Sweep 7 — missing pagination       : <N endpoints>  (manual review)

Findings by severity
- CRITICAL: <N>   (auto-fixed: <K>, manual: <N-K>)
- HIGH:     <N>   (auto-fixed: <K>, manual: <N-K>)
- MEDIUM:   <N>   (mostly manual)
- LOW:      <N>

Hard-stop:  no  (this skill is soft-fail)

Next step: triage the manual queue, then run `review-and-ship-to-staging`.
```
