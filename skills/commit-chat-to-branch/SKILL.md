---
name: commit-chat-to-branch
description: >-
  Take exactly the files the agent edited in the current chat, create
  a fresh branch from `origin/staging`, commit them — then STOP. No
  push, no PR, no merge. Lets the user (a) checkpoint mid-task before
  switching contexts, (b) hand the branch off to a teammate, or
  (c) defer the full ship flow to later. Structurally a subset of
  `review-and-ship-to-staging` (the first half). Use when the user
  says "commit this chat to a branch", "checkpoint this chat",
  "branch and commit my edits", "save this chat as a branch", or
  "create a branch with my changes".
---

# Commit Chat to Branch

The smallest skill in the system. Two steps: branch + commit. Stops
there. Designed to be the cheapest possible way to put chat work into
git without committing to (a) shipping it yet or (b) pushing it
anywhere.

## When to use this skill

- "I need to stop for the day but don't want to lose the chat's
  edits."
- "I'm context-switching — checkpoint this and let me come back."
- "My teammate wants to take this over — give them a branch they can
  `git fetch && git checkout`."
- After `audit-all` or `hygiene-all`, when you want to land the
  fixes into a branch but NOT yet open a PR.

## Relationship to `review-and-ship-to-staging`

This skill IS the first half of
[`review-and-ship-to-staging`](../review-and-ship-to-staging/SKILL.md)
— specifically steps 1, 4, 5, 6 from that skill's workflow. If the
user later runs `review-and-ship-to-staging` on the same chat:

1. The ship skill SHOULD detect that the branch already exists
   (`git rev-parse --verify <branch>` succeeds) and simply push + PR
   + merge from where this skill left off.
2. If the ship skill doesn't yet know about that branch, it will
   re-branch from `origin/staging` — that's OK, the second branch
   just supersedes the first. The user can delete the orphan.

A branch created by this skill uses the `chat/<short-kebab-summary>`
prefix (vs the ship skill's `fix/<short-kebab-summary>`) — that prefix
difference makes it obvious in `git branch` listings which branches
are "checkpointed but not shipped" vs "actively being shipped".

## Workflow

```
Task progress:
- [ ] Build the chat-files list from agent edits in this chat
- [ ] If empty, stop with "nothing to commit"
- [ ] Decide the base branch (default: origin/staging)
- [ ] Fetch origin/<base>
- [ ] Stash only the chat's files
- [ ] Create branch chat/<short-kebab-summary> from origin/<base>
- [ ] Pop the stash
- [ ] Stage exactly the chat's files
- [ ] Commit with a heredoc message
- [ ] Stop — print the report
```

### Step 1. Build the chat-files list

Use the chat history to enumerate files the agent EDITED (Write /
StrReplace / Delete / EditNotebook tool calls). NEVER include files
the agent only read or searched. Save as a shell variable:

```bash
FILES=( "path/one.py" "path/two.tsx" "path/three.md" )
```

### Step 2. Empty-list guard

If `FILES` is empty:

```
No files were edited in this chat — nothing to commit.
```

…and stop. Don't create an empty branch, don't ask follow-up
questions.

### Step 3. Decide the base branch

Default: `origin/staging` (matches the convention used by the rest of
the skill system).

Honor user overrides in the same prompt:

- "branch from main" → base = `origin/main`
- "branch from current" → base = current branch HEAD
- "branch from `<ref>`" → base = `<ref>`

Always `git fetch origin <base>` first so we have the latest.

### Step 4. Stash only the chat's files

Important: leave UNRELATED working-tree changes alone. The user may
have edits in flight that aren't part of this chat.

```bash
git stash push -m "<short-kebab-summary>" -- "${FILES[@]}"
```

### Step 5. Create the branch and pop the stash

Pick the kebab-case summary from the chat's top-level theme — keep
it short (≤ 6 words), all lowercase, hyphens between words.

```bash
BRANCH="chat/<short-kebab-summary>"
if git rev-parse --verify "$BRANCH" >/dev/null 2>&1; then
  echo "Branch $BRANCH already exists locally. Pick a new name (e.g. add -v2 suffix)."
  git stash pop  # restore the working tree before stopping
  exit
fi
git checkout -b "$BRANCH" "$BASE"
git stash pop
```

If `git stash pop` reports a conflict (extremely rare — would require
the base branch to have a conflicting change since the user last
pulled), STOP IMMEDIATELY:

- Report the conflicting files.
- Tell the user to resolve, then re-run.
- Do NOT auto-resolve.

### Step 6. Stage and commit

Stage EXACTLY the chat's files — never `git add -A` / `git add .` —
even though we already stashed only those files. The explicit add
double-locks the intent.

Then commit with a HEREDOC body. Match the conventional-commit style
used by recent history:

```bash
git log --oneline -10 origin/staging  # check existing tone
git add "${FILES[@]}"
git commit -m "$(cat <<'EOF'
<type>: <subject line>

<one-paragraph why; what the chat changed.>

This commit was checkpointed via `commit-chat-to-branch` — the branch
was NOT pushed and NO PR was opened. Run `review-and-ship-to-staging`
on this branch when ready to land it.
EOF
)"
```

Conventional types (pick the most accurate):

- `feat:` — new user-facing capability.
- `fix:` — user-visible bug fix.
- `refactor:` — internal restructuring with no behavior change.
- `perf:` — measurable performance improvement.
- `chore:` — tooling, docs, build, dependency bumps.
- `docs:` — documentation only.

### Step 7. Stop

Do NOT `git push`. Do NOT open a PR. Do NOT switch branches back.

Print the Output Template and end.

## Output Template

```
Checkpoint summary

Base branch:        <base>
New branch:         chat/<short-kebab-summary>
Commit:             <sha>  <subject>
Files (<N>):
  • path/one.py
  • path/two.tsx
  • path/three.md

Working tree:       clean (only chat files were touched)
Branch is local-only: yes (NOT pushed to origin)

Next step (pick one):
  - Continue working in this chat — your edits are safely on the branch.
  - Switch to another task — `git checkout staging` (or wherever) and pick this up later via `git checkout chat/<short-kebab-summary>`.
  - Hand off — tell your teammate to `git fetch && git checkout chat/<short-kebab-summary>` (the branch is local-only until someone pushes it).
  - Push without PR — `git push -u origin chat/<short-kebab-summary>` (still no PR, just makes it visible on the remote).
  - Push + PR + merge — run `review-and-ship-to-staging` and it will pick up the existing branch.
```

## Hard Rules (never break)

- **Never push** the new branch to `origin`. The user controls when
  (and whether) it leaves their machine.
- **Never open a PR** or call the GitHub MCP. This skill is
  local-only.
- **Never include unrelated working-tree changes** in the commit —
  always stage exactly the chat-files list, never `git add -A` /
  `git add .`.
- **Never branch off `main`** by default — same convention as
  `review-and-ship-to-staging`. Only branch off another base if the
  user explicitly asks.
- **Never amend a previous commit** to land follow-up chat edits —
  create a NEW commit on the same branch instead. If the user wants
  amend semantics, they can do it manually.
- **Never commit** secrets / `.env` / credentials files even if they
  appear in the chat-files list. Same rule as the ship skill.
- **Never delete or rename** the branch if it already exists — stop
  and ask the user to choose a new name. (Branch deletion would
  silently discard committed work.)
- **Never overwrite an existing commit on the branch** via
  `--amend` or `git commit --fixup` autonomously.
- **Never proceed past `git stash pop`'s conflict.** Surface the
  conflict and stop.

## Trigger Examples

- "commit this chat to a branch"
- "checkpoint this chat"
- "branch and commit my edits"
- "save this chat as a branch"
- "create a branch with my changes"
- "run the commit-chat-to-branch skill"
