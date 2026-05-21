---
name: concurrency-audit
description: >-
  Audit the whole codebase (not just files this chat edited) for the two
  concurrency axes already codified in `review-and-ship-to-staging`:
  one user with many concurrent chats, and many users with many
  concurrent chats. Catches unguarded mutation of shared singletons,
  PocketFlow nodes that smuggle state via class-level vars,
  `asyncio.create_task` calls without cancellation, and frontend
  module-level mutable caches. Use when the user says "audit
  concurrency", "check race conditions", "make sure nothing breaks
  under concurrent use", or "concurrency audit".
---

# Concurrency Audit

A codebase-wide expansion of the concurrency rules at
`[.cursor/skills/review-and-ship-to-staging/SKILL.md](.cursor/skills/review-and-ship-to-staging/SKILL.md)`
lines 39–50. Where the ship skill only audits chat-edited files, this
one walks every Python and TypeScript file and applies safe auto-fixes.

## Stack-Specific Context

- The shared backend state lives in
  `[backend/state.py](backend/state.py)` and is fronted by proxy classes
  (`ActiveSessionsProxy`, `WebSocketConnectionsProxy`, plus several
  others) that internally hold module-level dicts protected by
  module-level `threading.Lock` instances:
  - `_sessions_lock`, `_websocket_lock`, `_conversations_lock`,
    `_guest_credits_lock`, `_guest_event_rates_lock`.
  - Every mutation of those dicts MUST happen inside `with <lock>:`.
- PocketFlow nodes (in `[backend/services/nodes/](backend/services/nodes/)`)
  are allocated fresh per `create_*_flow()` call. Per-instance `self._foo`
  is safe; per-class or per-module mutable state is NOT — two flow runs
  would share it.
- The frontend uses React Query and hook-local state. Anything stored at
  module scope in `[frontend/lib/](frontend/lib/)` that isn't immutable
  config is a multi-tenant bug waiting to happen.
- Long-running async tasks: see the rule "Long-running backend LLM
  calls MUST have timeouts" in `[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)`.
  Aggregation timeout = 180s, enrichment = 90s. Concurrency audit
  enforces both the timeout AND `asyncio.create_task` cancellation
  hygiene.

## When to use this skill

- After any change to `backend/state.py`, `backend/services/nodes/`,
  `backend/services/research_service.py`, or any new singleton/cache.
- As the fifth step of `audit-all`.
- Periodically (e.g. monthly) as a hygiene sweep — concurrency bugs
  accumulate slowly and only manifest under load.

## Workflow

```
Task progress:
- [ ] Axis A: enumerate every module-level mutable in backend/ (top-level dict, list, set, class with class-level mutables)
- [ ] Axis A: confirm each one is either immutable, behind a lock, or wrapped by a proxy in backend/state.py
- [ ] Axis B: enumerate PocketFlow Node subclasses; flag any class-level (not self.*) mutable attribute
- [ ] Axis C: enumerate every asyncio.create_task call; verify cancellation guard or task tracking
- [ ] Axis D: enumerate every LLM call site in backend/services/nodes/; verify asyncio.wait_for wrap
- [ ] Axis E: enumerate frontend module-level state (let / const arrays / sets / maps in frontend/lib/)
- [ ] Apply auto-fixes for low-risk findings (wrap mutation in existing lock; hoist module mutable into class instance)
- [ ] Flag higher-risk findings for human review (e.g. a singleton intentionally shared — needs design decision)
- [ ] Print per-axis report
```

### Axis A — module-level mutable state in backend/

Identify with:

```bash
rg -n '^(_?[a-z_][a-z0-9_]*)\s*[:=]\s*(\{|\[|set\(|dict\(|list\()' backend/ --type py
```

For each hit, classify:

1. **Immutable** (frozen const, e.g. `STAGES = ("a", "b", "c")` or a
   tuple of strings) — pass.
2. **Behind a lock in `backend/state.py`** — pass (the proxy classes
   guarantee every mutation goes through `with _<name>_lock:`).
3. **Bare mutable not in `backend/state.py`** — flag.

For #3, the auto-fix path is: hoist the mutable into a new entry on the
appropriate proxy in `[backend/state.py](backend/state.py)`, or — if
the data is per-request — convert it to a request-scoped FastAPI
`Depends(...)` or a per-flow `self.*` attribute.

If the agent can't decide where the data belongs, the finding stays in
the report with the recommended refactor — do NOT silently rewrite.

### Axis B — PocketFlow Node class-level state

PocketFlow nodes use `prep / exec / post`. State that needs to live
across those methods belongs on `self`, NOT on the class.

```bash
rg -nB1 '^class \w+\((?:Node|AsyncNode|BatchNode|AsyncBatchNode|AsyncParallelBatchNode)\)' backend/services/nodes/ --type py
```

For each Node subclass, read the class body and flag any:

- Mutable class attribute (`results = []` at class scope, not inside
  `__init__`).
- Reference to a module-level mutable that the node writes to (use
  `rg <module>.append|<module>\[` inside the node file).
- Use of `functools.lru_cache` or `@cache` on an `exec` method (the
  cache is process-wide and bleeds across concurrent runs).

Auto-fix: move `class_attr = []` → `self.class_attr = []` inside an
`__init__` that calls `super().__init__()`.

### Axis C — `asyncio.create_task` hygiene

```bash
rg -n 'asyncio\.create_task\(' backend/ --type py
```

For each call:

- The returned task MUST be kept (assigned, awaited, or stored on a
  per-request set) — fire-and-forget tasks with no reference are GC'd
  unpredictably and silently swallow exceptions.
- The surrounding context (request handler, WebSocket loop, flow run)
  MUST cancel the task on shutdown / disconnect — look for a paired
  `task.cancel()` in a `finally:` / on-disconnect handler.

Auto-fix only when the pattern is unambiguous: a bare
`asyncio.create_task(fn())` inside a function whose enclosing class has
a `_background_tasks: set[asyncio.Task]` attribute. In that case rewrite
to `task = asyncio.create_task(fn()); self._background_tasks.add(task);
task.add_done_callback(self._background_tasks.discard)`. Everything
else: flag.

### Axis D — LLM call timeouts

For each LLM call in `[backend/services/nodes/](backend/services/nodes/)`
(grep `call_llm_async|client\..*\.create|generate_content`), verify it
is wrapped in `asyncio.wait_for(..., timeout=...)` per the
"Long-running backend LLM calls MUST have timeouts" rule. The expected
default timeouts:

- Aggregation node: 180s
- Citation enrichment: 90s
- Pruning / classification / quick decisions: 30s

Unwrapped calls are HIGH severity (a stuck LLM provider hangs the
entire research pipeline silently). Auto-fix is allowed only when an
existing wait_for value is documented for that node type.

### Axis E — frontend module-level mutable state

```bash
rg -n '^(let|const) \w+\s*[:=]\s*(\{|\[|new Set|new Map)' frontend/lib/ --type ts --type tsx
```

For each hit:

- `const FOO = { ... } as const` or a string-literal tuple — pass.
- `let cache = new Map()` at module scope holding per-user data — HIGH
  finding. Per-user state belongs in a hook (`useRef` / React Query
  cache keyed by user id), NEVER on a module singleton — every user in
  every tab in the same Next.js process would share it under SSR.

Pub/sub event channels (e.g. `lib/events/sessionTopics.ts` which holds
a module-level subscriber set keyed by session id) are an explicit
exception when the keying is per-session and entries are reaped on
unsubscribe — flag for review with that rationale rather than
auto-fixing.

## Severity Classes

| Severity | Examples |
|----------|----------|
| CRITICAL | A bare module-level dict in `backend/services/` that two concurrent flow runs would corrupt; PocketFlow Node with a class-level mutable list/dict; LLM call with no timeout in a research pipeline |
| HIGH | `asyncio.create_task` with no reference / no cancellation; frontend `let cache = new Map()` holding per-user data at module scope |
| MEDIUM | `@lru_cache` on a node method (process-wide cache); module-level pub/sub keyed by something other than per-session |
| LOW | Missing inline comment explaining why a particular module-level value is safe |

The skill hard-stops on ANY unresolved CRITICAL finding.

## Hard Rules (never break)

- **Never auto-add a new lock** without checking whether an existing
  one in `[backend/state.py](backend/state.py)` already protects that
  state. Adding a redundant lock is a deadlock waiting to happen.
- **Never auto-fix an LLM-timeout finding** unless the canonical
  timeout for that node category is documented in
  `[.cursor/rules/frontend.mdc](.cursor/rules/frontend.mdc)` or in
  another existing node in the same family. Otherwise the agent
  guesses the timeout and either swallows legitimate long calls or
  papers over a hung provider.
- **Never auto-hoist module-level state across files.** The fix can
  rewrite within a file or move to `[backend/state.py](backend/state.py)`
  — moving to a third file requires human review of the call sites.
- **Never auto-modify pub/sub event channels** keyed by per-session
  identifiers (e.g. `[frontend/lib/events/sessionTopics.ts](frontend/lib/events/)`)
  — these are deliberately module-singleton and safe.
- **Never push, commit, or PR.** Audit and rewrite the working tree
  only. Hand off to `review-and-ship-to-staging`.

## Trigger Examples

- "audit concurrency"
- "check race conditions"
- "make sure nothing breaks under concurrent use"
- "concurrency audit"
- "run the concurrency-audit skill"

## Output Template

End the run with:

```
Concurrency-audit summary

Axis A — backend module-level mutable state
- Files scanned:                  <N>
- Mutables found:                 <N>
  - Immutable / safe:             <N>
  - Behind a lock in state.py:    <N>
  - UNSAFE:                       <N>  (CRITICAL: <K> — auto-fixed: <X>)

Axis B — PocketFlow Node class-level state
- Node subclasses scanned:        <N>
- Class-level mutables found:     <N>  (CRITICAL: <K> — auto-fixed: <X>)
- lru_cache on exec methods:      <N>  (MEDIUM)

Axis C — asyncio.create_task hygiene
- Call sites:                     <N>
- Untracked / uncancellable:      <N>  (HIGH — auto-fixed: <X>)

Axis D — LLM call timeouts
- LLM call sites:                 <N>
- Missing asyncio.wait_for wrap:  <N>  (CRITICAL — auto-fixed: <X>)

Axis E — frontend module-level state
- Mutables found:                 <N>
- Holds per-user data unsafely:   <N>  (HIGH)

Hard-stop:                        yes | no  (yes if any unresolved CRITICAL)

Next step: review the items the audit refused to auto-fix, then run `review-and-ship-to-staging`.
```
