---
name: migration-safety
description: >-
  Audit every Alembic migration in `backend/db/migrations/versions/`
  for the safety pitfalls that take down a production DB: missing
  `downgrade()`, destructive ops without a backfill plan, NOT NULL
  columns added without `server_default`, and orphan heads in the
  revision chain. Dry-runs `alembic upgrade head --sql` so the user
  sees the exact SQL. Never auto-fixes destructive changes — those are
  always human review. Use when the user says "check migrations",
  "audit alembic", or "migration safety check".
---

# Migration Safety

A read-mostly skill — it never auto-fixes destructive operations. The
auto-fixes are limited to filling in an empty `downgrade()` body when
the corresponding `upgrade()` is purely additive (an inverse can be
synthesized safely).

## Stack-Specific Context

- Alembic config: `[alembic.ini](alembic.ini)`.
- Migrations live in
  `[backend/db/migrations/versions/](backend/db/migrations/)`.
- Migration runner / convenience entry: `[backend/db/migrate.py](backend/db/migrate.py)`.
- Database URL: `DATABASE_URL` env var (SQLite for local dev,
  Postgres in staging/production per
  `[docker-compose.staging.yml](docker-compose.staging.yml)` /
  `[docker-compose.production.yml](docker-compose.production.yml)`).

## When to use this skill

- Any time a new file appears under
  `[backend/db/migrations/versions/](backend/db/migrations/)`.
- As the sixth step of `audit-all` (the skill self-skips if no Alembic
  changes are in the working tree, so it's cheap to leave in the
  chain).
- Before promoting `staging` → `main` on any release that touches the
  schema.

## Workflow

```
Task progress:
- [ ] List versions/ — if empty / no new files since origin/staging, exit clean
- [ ] Check 1: every migration file has both upgrade() and downgrade() (non-empty bodies)
- [ ] Check 2: alembic heads — must be exactly one
- [ ] Check 3: alembic check (Alembic 1.9+) for un-modeled changes
- [ ] Check 4: scan migration bodies for destructive ops; require explicit backfill or human-review TODO
- [ ] Check 5: scan for NOT NULL column adds without server_default
- [ ] Generate dry-run SQL via `alembic upgrade <prev>:head --sql`
- [ ] Print findings; auto-fill safe inverse for empty downgrades; everything else is manual
```

### Check 1 — `upgrade()` AND `downgrade()` populated

For each migration file under `[backend/db/migrations/versions/](backend/db/migrations/)`:

```bash
rg -n 'def downgrade\(' backend/db/migrations/versions/<file> -A 2
```

Empty bodies (`pass`, `...`, or `# TODO`) on either function are
flagged. Allowed exception: a `# baseline` migration (the very first
one) may legitimately have an empty `downgrade()` if it would otherwise
need to drop tables that predate Alembic — confirm by inspecting
`down_revision = None`.

Auto-fix for empty `downgrade()`: only if the corresponding `upgrade()`
is **purely additive** (only `op.add_column`, `op.create_table`,
`op.create_index`). In that case, synthesize the inverse
(`op.drop_column`, `op.drop_table`, `op.drop_index`) preserving names.
For anything else (alter, rename, type change, data migration), leave
empty and surface as HIGH for human review.

### Check 2 — single head

```bash
alembic heads
```

Output MUST contain exactly one line. Multiple heads = a merge
migration was forgotten and two branches will fight on `upgrade head`.

If multiple heads are detected, the skill does NOT auto-merge. It
prints both heads and the recommended `alembic merge -m "merge <a>
and <b>" <a> <b>` command, then stops with a HIGH finding.

### Check 3 — `alembic check`

```bash
alembic check
```

If the SQLAlchemy models in `[backend/db/](backend/db/)` have changed
without a corresponding migration, this command fails. Surface as a
HIGH finding with the diff — never auto-generate a migration (the
agent doesn't know intent for renames vs add+drop).

### Check 4 — destructive ops

Scan each new migration for:

```python
op.drop_table(
op.drop_column(
op.alter_column(... existing_type=..., type_=...)  # type change
op.execute("DROP "
op.execute("DELETE "
op.execute("TRUNCATE "
op.execute("UPDATE "
```

Every hit is HIGH severity. Each one must be accompanied by EITHER:

- A `# safe: <reason>` comment immediately above (e.g. `# safe:
  column was never written to in production — confirmed by query on
  YYYY-MM-DD`), OR
- A preceding backfill / data-migration block in the same `upgrade()`.

Missing both: flag as HIGH and require human review. Never auto-fix.

### Check 5 — NOT NULL adds without `server_default`

```python
op.add_column("table", sa.Column("new_col", sa.String(), nullable=False))
```

If `server_default` is not also supplied, this WILL fail on Postgres
the moment the migration runs against a non-empty table. Skill flags
as CRITICAL.

Auto-fix (allowed): add `server_default=sa.text("''")` for strings,
`sa.text("0")` for numerics, `sa.false()` for booleans, `sa.func.now()`
for timestamps — but ONLY if the agent can determine a safe default
from the column type alone. If the default is domain-specific (e.g.
"empty array of role IDs"), flag for human review.

The "right" pattern is usually:

```python
op.add_column("table", sa.Column("new_col", sa.String(), nullable=True))
op.execute("UPDATE table SET new_col = '' WHERE new_col IS NULL")
op.alter_column("table", "new_col", nullable=False)
```

Surface that recommendation in the report when auto-fixing is not
appropriate.

### Step 6 — Dry-run SQL

```bash
alembic upgrade <previous_head>:head --sql > /tmp/migration.sql
```

Attach the contents to the report so the user can see EVERY statement
that will run against production. Highlight any `DROP`, `DELETE`,
`UPDATE`, or `TRUNCATE` lines in the dump.

If the alembic command itself fails (e.g. environment not configured,
DB URL missing), report the failure and the exact command — do NOT
guess a config.

## Hard Rules (never break)

- **Never auto-generate a new migration** via `alembic revision
  --autogenerate`. The agent doesn't know which schema deltas are
  intended renames vs add+drop. Always human review.
- **Never auto-fix a destructive `upgrade()`.** Drop / delete /
  truncate / type-change blocks are always human review.
- **Never auto-merge multiple heads.** Surface the merge command and
  stop.
- **Never run `alembic upgrade head`** as part of this skill — only
  `--sql` (dry-run). The skill is read-only against the database.
- **Never edit a migration that has already been pushed to
  `origin/staging` or `origin/main`.** Pushed migrations have likely
  run somewhere; editing them in place creates state drift between
  environments. Surface as a "needs new follow-up migration" finding
  instead.
- **Never push, commit, or PR.** Modifies only the working-tree
  migrations to fill in safe inverse downgrades. Hand off to
  `review-and-ship-to-staging`.

## Trigger Examples

- "check migrations"
- "audit alembic"
- "migration safety check"
- "is this migration safe to ship?"
- "run the migration-safety skill"

## Output Template

End the run with:

```
Migration-safety summary

Working-tree migrations (new since origin/staging)
  • <file>  rev=<rev>  down_revision=<prev>
  • ...
  (none → skill exited clean)

Check 1 — upgrade()/downgrade() pair      : <pass|N empty>  (auto-filled: <K>)
Check 2 — alembic heads                   : <single|N heads>
Check 3 — alembic check (model drift)     : <pass|fail>
Check 4 — destructive ops                 : <N hits>  (all manual review)
Check 5 — NOT NULL without server_default : <N hits>  (auto-fixed: <K>, manual: <N-K>)

Dry-run SQL                : written to /tmp/migration.sql
  • DROP / DELETE / TRUNCATE statements: <N>  (REVIEW THESE BEFORE MERGE)

Findings by severity
- CRITICAL: <N>   (auto-fixed: <K>, manual: <N-K>)
- HIGH:     <N>   (all manual)
- MEDIUM:   <N>

Hard-stop:                 yes | no  (yes if any unresolved CRITICAL)

Next step: review the dry-run SQL, address manual items, then run `review-and-ship-to-staging`.
```
