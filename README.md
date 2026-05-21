# Agent Skills

A curated, tool-agnostic library of high-leverage **agent skills** —
focused, autonomous workflows your coding agent can invoke to audit,
clean, and ship a codebase safely.

Every skill is a single `SKILL.md` file with a frontmatter header
(`name`, `description`) followed by a deterministic, opinionated
workflow. Drop them into your agent of choice — **Cursor**, **Claude
Code**, or any tool that understands skill files — and your agent gets
the same disciplined pre-ship pipeline a senior engineer would run by
hand.

---

## Why this exists

Most coding agents are great at writing code. Far fewer are great at
the boring, high-value work _around_ the code: scanning for leaked
secrets, bumping CVEs, fixing lint debt, splitting oversized files,
opening clean PRs against the right branch, cutting SemVer tags,
generating release notes, rolling back when things go wrong.

This library encodes those workflows once, so every agent invocation
behaves the same way — with the same hard rules, the same severity
gates, and the same output shape.

---

## The 15 skills

### Audits (catch bugs before they ship)

| Skill | What it does |
|---|---|
| [`secret-scan`](skills/secret-scan/SKILL.md) | Sweep the repo for leaked API keys, tokens, and credentials. Auto-extracts hits into `os.getenv` / `process.env` lookups. |
| [`dependency-audit`](skills/dependency-audit/SKILL.md) | `pip-audit` + `npm audit`. Auto-bumps CRITICAL / HIGH CVEs. Verifies the lockfile still resolves before declaring done. |
| [`security-audit`](skills/security-audit/SKILL.md) | SQL injection, CORS, missing schema validation, cookie attributes, file uploads, dangerous Python builtins, path traversal, SSRF, unsanitized HTML. |
| [`audit-all`](skills/audit-all/SKILL.md) | Orchestrator. Runs the three above in the right order, with hard-stop gates between steps. |

### Hygiene (keep the codebase healthy)

| Skill | What it does |
|---|---|
| [`dead-code-sweep`](skills/dead-code-sweep/SKILL.md) | `ruff` + `eslint` + `vulture` + `ts-prune`. Auto-removes unused imports / vars; reports unreferenced files for human review. |
| [`lint-and-typecheck-fix`](skills/lint-and-typecheck-fix/SKILL.md) | `ruff --fix`, `mypy`, `eslint --fix`, `tsc --noEmit`. Auto-fixes what tools can fix; categorizes the rest. |
| [`file-size-enforcer`](skills/file-size-enforcer/SKILL.md) | Enforces a 700-line cap per file. Auto-splits 701–1000-line files; writes a proposed-split TODO for anything bigger. |
| [`hygiene-all`](skills/hygiene-all/SKILL.md) | Orchestrator. Runs the three above in the right order. |

### Git & Ship (land changes safely)

| Skill | What it does |
|---|---|
| [`commit-chat-to-branch`](skills/commit-chat-to-branch/SKILL.md) | Smallest skill. Takes exactly the files this chat edited, creates a fresh branch from `origin/staging`, commits — then stops. Perfect mid-task checkpoint. |
| [`review-and-ship-to-staging`](skills/review-and-ship-to-staging/SKILL.md) | Full review-and-ship: critical code review of every edited file, branch from `staging`, commit, push, PR, merge. |
| [`promote-staging-to-main`](skills/promote-staging-to-main/SKILL.md) | Open a PR from `staging` → `main` and merge it as a true merge commit. No code changes — pure release event. |
| [`rollback-release`](skills/rollback-release/SKILL.md) | Roll back `main` to a previous tag via `git revert` on a PR (never `git reset`). Stops for human approval before merging. |

### Releases (tag, notes, publish)

| Skill | What it does |
|---|---|
| [`release-tag`](skills/release-tag/SKILL.md) | Decide the next SemVer (walks conventional commits), create an annotated tag, append to `release-notes.txt`. |
| [`release-notes-gen`](skills/release-notes-gen/SKILL.md) | Render a grouped Markdown changelog between two refs, publish as a GitHub Release. |

### Top-level orchestrator

| Skill | What it does |
|---|---|
| [`ship-ready`](skills/ship-ready/SKILL.md) | The full pipeline: `audit-all` → `hygiene-all` → (asks once) → `review-and-ship-to-staging`. The one and only interactive prompt in the whole system. |

---

## Recommended pipelines

**Daily** (small fix):

```
review-and-ship-to-staging
```

**Pre-ship** (anything sensitive):

```
ship-ready
  └── audit-all          ── secret-scan → dependency-audit → security-audit
  └── hygiene-all        ── dead-code-sweep → lint-and-typecheck-fix → file-size-enforcer
  └── (asks the user)
  └── review-and-ship-to-staging
```

**Release** (after staging is stable):

```
promote-staging-to-main → release-tag → release-notes-gen
```

**Emergency**:

```
rollback-release
```

---

## Wire it up

The `skills/` folder at the root is tool-neutral. Point your agent at
it however your tool prefers.

### Cursor

```bash
mkdir -p .cursor
ln -s ../skills .cursor/skills
```

### Claude Code

```bash
mkdir -p .claude
ln -s ../skills .claude/skills
```

### Anything else

Copy the `skills/` folder into your project, or symlink it to wherever
your tool discovers skill files. Each skill is fully self-contained —
no shared imports, no runtime, just Markdown.

---

## How a skill is structured

Every skill is a single file:

```
skills/<name>/SKILL.md
```

With this shape:

```markdown
---
name: <skill-id>
description: >-
  One paragraph the agent reads first. Includes when to use the skill
  ("use when the user says X, Y, Z") so it can be triggered by natural
  language.
---

# <Title>

## When to use this skill
- bullet
- bullet

## Workflow
\`\`\`
Task progress:
- [ ] Step 1
- [ ] Step 2
...
\`\`\`

(detailed step-by-step)

## Hard Rules (never break)
- ...

## Trigger Examples
- "..."

## Output Template
\`\`\`
<the exact report shape the agent prints at the end>
\`\`\`
```

That's it. No dependencies, no runtime, no DSL. Just disciplined prose
your agent can follow.

---

## Customizing for your stack

These skills ship with sensible defaults for a Python (FastAPI /
SQLAlchemy) + TypeScript (Next.js / React) monorepo with `backend/`
and `frontend/` directories. Most are stack-agnostic; the few that
mention specific paths flag this explicitly under a **Stack
Assumptions** section.

To adapt: edit the `rg` paths, the lint commands, and the exclusion
globs in each skill. The hard rules, severity classes, and workflow
shape transfer to any stack.

---

## License

MIT. Use them, fork them, improve them, ship them.
