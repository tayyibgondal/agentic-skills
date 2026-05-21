---
name: review-and-ship-to-staging
description: >-
  Production-quality review of every file the agent edited in the current chat,
  then create a branch from `staging`, commit only those edits, open a pull
  request against `staging`, and merge it. Use when the user says
  "review and ship", "review and merge to staging", "execute the review-and-ship
  skill", or anything that asks the agent to ship the current chat's changes to
  the `staging` branch with a clean review pass first.
---

# Review and Ship to Staging

A two-phase project skill: critical code review of the agent's own edits in
this chat, then a git/GitHub flow that lands them on `staging` via a PR.

## Quality Bar (the review phase)

Apply these checks to **every** file the agent has modified in the current
chat. Pre-existing issues in untouched files are out of scope.

1. **File size cap**: no file the agent edited may exceed **700 lines** because
   of this chat's changes. If an edit pushes a file past 700, either tighten
   the new lines so the net delta is ≤0 or split the file. Pre-existing
   files already over 700 are allowed to stay if the agent's net delta is
   small and the file already carries an explicit refactor TODO; flag this in
   the review summary either way.
2. **No hardcoded values outside configs**: secrets, environment-specific
   URLs, magic numbers that should be admin-tunable, etc. must live in the
   project's config / settings layer (`backend/config/`, env vars, admin
   settings DB rows, `frontend/lib/constants.ts`), NOT inline in node /
   component code. Stage→percent display hints, stage name string literals,
   and similar local-scope constants that follow an existing pattern in the
   same dict literal are acceptable.
3. **No redundancy**: dead branches the new code makes unreachable must be
   removed (don't leave defensive `|| "old_stage"` checks once the new stage
   name is the only one that can flow through). Don't introduce duplicated
   logic if a single source of truth already exists in the codebase.
4. **Concurrency safety** — verify both axes:
   - **One user, many concurrent chats**: any mutable state the new code
     touches must NOT live on a module-level or class-level singleton that
     two flow runs could share. PocketFlow nodes are allocated fresh per
     `create_*_flow()` call, so per-instance `self._foo` is safe; per-module
     globals are not.
   - **Many users, many concurrent chats**: all shared backend state
     (`ProgressTracker.progress_data`, `active_sessions`,
     `websocket_connections`) must be mutated only under the existing lock
     (`tracker._lock` etc.) — never read-modify-write outside it. Frontend
     state must be per-component / per-message, never on a module singleton.
5. **Architecture**: changes must be additive and follow existing patterns
   already in the codebase. If the project has design-system rules or
   architectural conventions documented elsewhere (e.g. an AGENTS.md, a
   rules file, or a design system doc), honor them.
6. **Lint**: run `ReadLints` on every modified file; fix every error the
   agent introduced. Pre-existing lints in untouched files are out of scope.

If any check fails, fix it before moving to the ship phase. Summarize each
check's pass/fail outcome in the chat reply.

## Ship Phase Workflow

After the review passes:

```
Task progress:
- [ ] Identify the exact set of files this chat modified
- [ ] Confirm staging branch is reachable
- [ ] Diff those files between origin/main and origin/staging — if they differ, stop and ask the user
- [ ] Stash only the chat's files (leave unrelated working-tree changes alone)
- [ ] Create branch fix/<short-kebab-summary> from origin/staging
- [ ] Pop the stash, stage only the chat's files, commit, push
- [ ] Open the PR against staging (MCP first, fallback to URL if token lacks write scope)
- [ ] Merge the PR (MCP merge_pull_request, merge commit by default)
- [ ] Report PR URL + merge SHA back to the user
```

### Step-by-step

**1. Collect the file list.** Build it from the chat history of agent edits;
never include files the agent only read. Save it as a shell variable for the
subsequent `git stash` / `git add` calls.

**2. Verify `staging` exists and is up to date.**

```bash
git fetch origin staging
git branch -a | grep -E '(^|/)staging$'
```

**3. Confirm the files apply cleanly to staging.**

```bash
git diff --stat origin/staging..origin/main -- <chat-files>
```

If the diff is empty, my edits will apply cleanly. If it is non-empty, stop
and ask the user how to proceed (they may have intentionally diverged
`main` from `staging`).

**4. Stash only the chat's files** so unrelated working-tree changes stay put:

```bash
git stash push -m "<short-kebab-summary>" -- <chat-files>
```

**5. Branch from `origin/staging` and restore the stash:**

```bash
git checkout -b fix/<short-kebab-summary> origin/staging
git stash pop
```

**6. Stage exactly the chat's files and commit.** Commit message style
matches recent history (`git log --oneline -10 origin/staging`): lowercase
`type: short subject` (`fix:`, `feat:`, `refactor:`). Use a HEREDOC for the
multi-line body so formatting is preserved:

```bash
git add <chat-files>
git commit -m "$(cat <<'EOF'
fix: <subject line>

<one-paragraph why; what the bug was / what the change does.
Reference the rules followed: concurrency-safety, sticky tag pattern,
single-source-of-truth stage names, etc.>
EOF
)"
```

**7. Push and capture the create-PR URL:**

```bash
git push -u origin fix/<short-kebab-summary>
```

**8. Open the PR.** Prefer the `user-github` MCP tool:

```
CallMcpTool user-github create_pull_request {
  owner: "<owner>",
  repo: "<repo>",
  title: "<same as commit subject>",
  head: "fix/<short-kebab-summary>",
  base: "staging",
  body: "<summary + concurrency review + test plan>"
}
```

If the MCP returns `403 Resource not accessible by personal access token`,
the configured PAT lacks `pull_requests:write` on this repo. Fallback:
print the COMPARE URL with the base pre-filled as `staging` — NOT the
`/pull/new/<branch>` URL that `git push` prints, because that URL
defaults the base to the repository's default branch (usually `main`)
which would target the wrong branch:

```
https://github.com/<owner>/<repo>/compare/staging...<branch>?expand=1
```

Tell the user to open exactly that URL, click "Create pull request", and
**stop the ship phase**. Note the PAT gap so the user can re-scope it once.

**9. Merge the PR.** Before merging, READ the PR via the MCP to confirm
`base.ref == "staging"`:

```
CallMcpTool user-github pull_request_read {
  owner: "<owner>",
  repo: "<repo>",
  pullNumber: <number>
}
```

If `base.ref` is anything other than `"staging"`, refuse to merge and tell
the user to retarget the PR (or close + reopen against the correct base).

Once the base is confirmed `staging`, merge via the MCP. Default to
**merge** (true merge commit, `merge_method: "merge"`). This preserves
the feature-branch lineage so the git graph cleanly shows the branch
line joining back into `staging` — matching the visual the user gets
from the IDE's "Merge into Current" workflow. A squash merge severs
that link (one parent only), which leaves the feature branch as a
visually orphaned line in the graph forever.

Title: commit subject. No extra commit message.

```
CallMcpTool user-github merge_pull_request {
  owner: "<owner>",
  repo: "<repo>",
  pullNumber: <number>,
  merge_method: "merge",
  commit_title: "<commit subject>"
}
```

If the user explicitly asks for a squash merge on a particular PR
(e.g. "squash this one, the feature-branch commits are noisy"), pass
`merge_method: "squash"` for that single PR. Do not change the default.

**10. Report back.** Print the PR URL, the merge SHA, and the branch name
the user can clean up. Do NOT delete the local branch automatically.

## Hard Rules (never break)

- **Never include unrelated working-tree changes** in the commit. Always
  stage only the files this chat edited.
- **Never branch off `main`** for this skill. The branch is always created
  from `origin/staging`.
- **The PR base is ALWAYS `staging`** — never `main`. The repo's default
  branch is `main`, so any URL that doesn't explicitly say `staging` will
  silently target the wrong branch. Always pass `base: "staging"` on the
  MCP call AND, in the fallback URL, use `/compare/staging...<branch>` so
  the base is pre-filled. Never use `/pull/new/<branch>` as the fallback
  URL — that URL defaults the base to `main`.
- **Before merging, verify the PR's base is `staging`**. If the PR was
  created manually (fallback path), explicitly ask the user to confirm the
  base shows `staging` in the PR header. If the MCP `pull_request_read`
  reports `base.ref != "staging"`, refuse to merge and report the
  mismatch.
- **Never force-push.** Never `git config` changes. Never skip pre-commit hooks.
- **Never amend** a pushed commit. If the review pass turns up more work
  after pushing, push a follow-up commit on the same branch.
- **Never commit** secrets / `.env` / credentials files even if they appear
  staged.

## Trigger Examples

- "review all the files you edited and ship to staging"
- "run the review-and-ship-to-staging skill"
- "review for prod, then PR to staging and merge"

## Output Template

End the run with:

```
Review summary
- File size ≤ 700: <pass|fail with details>
- Hardcoded values: <pass|fail>
- Redundancy / dead code: <pass|fail>
- Concurrency (single-user, multi-chat): <pass|fail>
- Concurrency (multi-user, multi-chat): <pass|fail>
- Architecture / patterns: <pass|fail>
- Lint: <pass|fail>

Ship summary
- Branch: fix/<short-kebab-summary>
- Commit: <sha>  <subject>
- PR: <url>
- Merge SHA: <sha>
```
