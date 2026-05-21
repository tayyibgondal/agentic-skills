---
name: promote-staging-to-main
description: >-
  Open a pull request from `staging` to `main` and merge it as a true merge
  commit, promoting the current staging build to production. Use when the
  user says "promote staging to main", "ship staging to main", "merge staging
  into main", "release current staging", "make main latest", or anything that
  asks to bring `main` up to whatever is currently on `staging`. Does NOT
  modify any files — the assumption is staging is already the source of truth
  and just needs to be brought into main.
---

# Promote Staging to Main

A read-only release workflow: open a PR from `staging` → `main`, show the
user exactly what is about to land, then merge it as a true merge commit so
the git graph cleanly records the promotion event.

## When to use this skill

This skill is the "release" step that comes AFTER
`review-and-ship-to-staging` has landed individual feature PRs onto
`staging`. It promotes the current state of `staging` to `main` as a single
release event, with **no code modifications and no working-tree changes**.

Do NOT use this skill when:

- The user wants to merge a feature branch — use `review-and-ship-to-staging`
  with `base: staging` instead.
- `main` has commits that are not on `staging` (the diverged-history case).
  Stop and tell the user; they probably want to merge `main` into `staging`
  first via a normal PR.

## Workflow

```
Task progress:
- [ ] Fetch latest origin/main and origin/staging
- [ ] Confirm staging is strictly ahead of main (no main-only commits would be lost)
- [ ] Show the user the commits + file diffs about to be promoted
- [ ] Open PR from staging -> main via MCP
- [ ] Verify the PR's base.ref == "main" AND head.ref == "staging"
- [ ] Merge via MCP with merge_method: "merge" (true merge commit)
- [ ] Report PR URL and merge SHA
```

### Step-by-step

**1. Fetch and verify both branches exist.**

```bash
git fetch origin main staging
git branch -a | grep -E '(^|/)main$|(^|/)staging$'
```

**2. Confirm staging is ahead of main.**

```bash
git log --oneline origin/main..origin/staging   # commits to be promoted
git log --oneline origin/staging..origin/main   # MUST be empty
```

- If the first list is empty → there is nothing to promote. Tell the user
  and stop.
- If the second list is non-empty → `main` has diverged from `staging`. Stop
  and ask the user how to proceed. Do NOT try to "fix" the divergence from
  inside this skill.

**3. Show the user what is being promoted.** This is a production release;
the change-summary must be printed to chat before the PR is opened:

```bash
git log --oneline origin/main..origin/staging
git diff --stat origin/main..origin/staging
```

If the diff looks unexpectedly large (e.g. hundreds of files, multi-week
gap), pause and ask the user to confirm before continuing.

**4. Open the PR.** Title is always the same; body contains the change
summary so the GitHub release record is self-contained:

```
CallMcpTool user-github create_pull_request {
  owner: "<owner>",
  repo: "<repo>",
  title: "release: promote staging to main",
  head: "staging",
  base: "main",
  body: "## Release contents\n\n<numbered list of commits being promoted>\n\n## Files changed\n\n<diff stat block>"
}
```

If the MCP returns `403 Resource not accessible by personal access token`,
the configured PAT lacks `pull_requests:write` on this repo. Fallback:
print the COMPARE URL with the base pre-filled as `main` — NOT the
`/pull/new/staging` URL, which would default the base to the repository's
default branch (which IS already `main`, but be explicit):

```
https://github.com/<owner>/<repo>/compare/main...staging?expand=1
```

Tell the user to open exactly that URL, click "Create pull request", and
**stop the workflow**. Note the PAT gap.

**5. Verify the PR refs.** Before merging, read the PR via the MCP and
confirm BOTH refs are exactly what we expect:

```
CallMcpTool user-github pull_request_read {
  method: "get",
  owner: "<owner>",
  repo: "<repo>",
  pullNumber: <number>
}
```

If `base.ref != "main"` OR `head.ref != "staging"`, refuse to merge and
report the mismatch.

**6. Merge with a true merge commit** (`merge_method: "merge"`). This
matches the `review-and-ship-to-staging` convention and preserves the
feature-branch lineage in the graph. NEVER squash a staging→main promotion
— the staging history (every individual feature merge) is part of the
release record and must remain intact.

```
CallMcpTool user-github merge_pull_request {
  owner: "<owner>",
  repo: "<repo>",
  pullNumber: <number>,
  merge_method: "merge",
  commit_title: "Merge pull request #<number> from <owner>/staging"
}
```

**7. Report back.** Print the PR URL, merge SHA, the list of release
commits, and the file-stat summary so the user has a permanent record in
chat history.

## Hard Rules (never break)

- **The PR head is ALWAYS `staging`. The base is ALWAYS `main`.** Never any
  other combination. Reject anything else.
- **Never squash** a staging→main promotion. The release record needs the
  individual feature-merge commits preserved.
- **Never force-push** to `main`. Never `git config` changes. Never bypass
  required CI / branch protection checks.
- **Never modify the working tree.** This skill operates entirely on remote
  branches via the GitHub API; it does not run `git stash`, `git checkout`,
  `git commit`, or `git push` on the user's machine.
- **Stop if staging is behind or diverged from main.** Do not try to
  reconcile from inside this skill.
- **Never include secrets / `.env` / credentials** in the PR body even if
  they appear in a diff line.

## Trigger Examples

- "promote staging to main"
- "ship staging to main"
- "merge staging into main"
- "release current staging"
- "make main latest"
- "run the promote-staging-to-main skill"

## Output Template

End the run with:

```
Promotion summary
- Commits promoted: <N>
  <oldest>  <subject>
  ...
  <newest>  <subject>
- Files changed:    <stat>
- PR:               <url>
- Merge SHA:        <sha>
```
