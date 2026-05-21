---
name: rollback-release
description: >-
  Safely roll back `main` to a previous release tag by creating a
  `revert/<from-tag>-to-<to-tag>` branch, running `git revert`
  (preserves history — NEVER `git reset`), opening a PR titled
  `revert: rollback to <to-tag>` against `main`, and **stopping** for
  human approval before merging. The user merges it once they've
  confirmed the rollback is correct. Use when the user says "rollback
  to v1.2.0", "revert main to last tag", or "production rollback".
---

# Rollback Release

The "things broke, take us back" skill. Designed for the situation
where `main` is at `vX.Y.Z` and production is misbehaving, and you
want to land a commit on `main` that undoes everything since
`v(X-1).Y.Z` — without rewriting history.

## Why we never `git reset` for production rollback

- `git reset --hard origin/main~N` followed by `git push --force`
  destroys the commit history. Any other deployment, CI artifact, or
  collaborator that referenced those SHAs breaks.
- A revert-PR preserves the history. The rollback shows up as new
  commits ON TOP of the bad ones, so audit logs, deploy traceability,
  and rollback-of-the-rollback all work cleanly.

## When to use this skill

- Production is failing and we need `main` to look like the previous
  release.
- A migration / config change in the latest tag is causing issues
  that can't be patched faster than reverted.
- NOT for reverting an in-flight `staging` change — for that, just
  revert the PR on staging the normal way.

## Workflow

```
Task progress:
- [ ] Fetch latest origin (main + tags)
- [ ] Resolve target tag (the to-tag — where we want main to end up)
- [ ] Resolve current tag (the from-tag — where main is now)
- [ ] Identify the exact commit range to revert
- [ ] Sanity check: no merges between to-tag and from-tag that would conflict
- [ ] Create revert branch from origin/main
- [ ] git revert the range (one commit per reverted commit, in reverse order)
- [ ] Push branch
- [ ] Open PR against main via MCP
- [ ] STOP — do NOT auto-merge; print the PR URL and wait for human review
```

### Step 1. Fetch

```bash
git fetch origin --tags --prune --prune-tags
git fetch origin main
```

### Step 2. Resolve `TO_TAG`

The user specifies which tag they want to roll back to:

```bash
TO_TAG="v0.4.0"  # from the user's prompt
git rev-parse --verify "refs/tags/${TO_TAG}^{commit}"
```

If the tag doesn't exist, stop and list the recent tags
(`git tag --sort=-creatordate -l 'v*' | head -10`).

### Step 3. Resolve `FROM_TAG`

```bash
FROM_TAG=$(git describe --tags --abbrev=0 --match 'v*' origin/main)
```

If `FROM_TAG == TO_TAG`, there's nothing to roll back. Stop with a
friendly note.

### Step 4. Identify the revert range

```bash
git log --pretty=format:'%H' "${TO_TAG}..origin/main" > /tmp/revert-shas.txt
COUNT=$(wc -l < /tmp/revert-shas.txt)
```

If `COUNT` is large (> 50), warn the user — reverting that many
commits in one PR will conflict frequently. Suggest reverting tag by
tag instead (e.g. roll back to the second-most-recent tag first, ship
that, then roll back further).

### Step 5. Conflict prediction (sanity check)

```bash
git checkout -b "revert/${FROM_TAG}-to-${TO_TAG}" origin/main
```

For each SHA in `/tmp/revert-shas.txt` (in reverse — newest first),
run `git revert --no-commit --no-edit <sha>`. If any revert conflicts,
STOP IMMEDIATELY:

- Run `git revert --abort` to clean up.
- Delete the branch.
- Report the conflicting SHA and the files involved.
- Tell the user that rollback requires manual conflict resolution and
  cannot be auto-shipped.

Conflicts are a hard stop because an auto-resolved revert in this
context is more likely to be wrong than right (the conflict means the
intervening commits depend on the change we're reverting).

### Step 6. Commit and push

If all reverts apply cleanly, commit them as a single commit per
revert (use `git revert <sha>` instead of `--no-commit` — keep one
revert per commit so the PR diff is reviewable):

```bash
git checkout -b "revert/${FROM_TAG}-to-${TO_TAG}" origin/main

# Apply reverts oldest-first within the range (git revert wants the
# range in either order, but committing oldest-first keeps the log
# readable):
git log --reverse --pretty=format:'%H' "${TO_TAG}..origin/main" \
  | xargs -I {} git revert --no-edit {}

git push -u origin "revert/${FROM_TAG}-to-${TO_TAG}"
```

### Step 7. Open the PR (via MCP)

```
CallMcpTool user-github create_pull_request {
  owner: "<owner>",
  repo: "<repo>",
  title: "revert: rollback to <TO_TAG>",
  head: "revert/<FROM_TAG>-to-<TO_TAG>",
  base: "main",
  body: "## Rollback contents\n\n
         Reverts every commit between <FROM_TAG> and <to-tag>.\n\n
         ## Commits reverted (newest first)\n
         <list of N commits with SHAs and subjects>\n\n
         ## Why\n
         <copied from the user's prompt if given, otherwise empty>\n\n
         ## After merging\n
         - The `release-tag` skill is OK to re-run; it will compute the next
           patch bump on top of the revert.\n
         - Run `staging-smoke-test` once the rollback PR is merged and the
           production deploy has cycled.\n
         - Do NOT delete the original tags. They stay as historical record."
}
```

If MCP returns 403 (PAT lacks `pull_requests:write`), print the
compare URL with `base=main`:

```
https://github.com/<owner>/<repo>/compare/main...revert/<FROM_TAG>-to-<TO_TAG>?expand=1
```

…ask the user to open it and click Create PR, and stop.

### Step 8. STOP — never auto-merge

This skill DOES NOT call `merge_pull_request`. The user reads the PR,
confirms the revert is what they want, and merges it themselves. This
is the single non-autonomous step in the whole skill system and it
exists for a reason: rolling back production is a high-stakes action
and the human has to consciously approve.

Print the PR URL and the next-step recommendations:

1. Review the PR.
2. Merge via merge commit (preserves the revert history).
3. Run `staging-smoke-test` against production.
4. If satisfied, optionally run `release-tag` to label the rollback
   (e.g. as `v0.4.0+rollback.1` — but only with explicit user
   request; the `release-tag` skill's normal bump rules won't pick
   this pattern).

## Hard Rules (never break)

- **Never `git reset` or `git push --force` to `main`.** Always
  revert via a PR.
- **Never auto-merge** the rollback PR. Print the URL and stop.
- **Never delete a tag** as part of rollback. `v0.4.1` stays a tag
  forever — it's part of the deploy history.
- **Never modify a previous tag** to point at a different SHA. Same
  reason.
- **Never auto-resolve a revert conflict.** Stop and report.
- **Never roll back more than one tag at a time without explicit
  user opt-in.** If the user says "rollback to v0.3.0" while we're
  on `v0.5.0`, confirm before walking the range.
- **Never roll back if `main` is behind `origin/main`.** Pull first;
  refuse on non-fast-forward.

## Trigger Examples

- "rollback to v1.2.0"
- "revert main to last tag"
- "production rollback"
- "undo the last release"
- "run the rollback-release skill"

## Output Template

End the run with:

```
Rollback-release summary

From tag (current main):     <FROM_TAG>     (sha: <short>)
To tag (target):             <TO_TAG>       (sha: <short>)
Commits to revert:           <N>
  • <sha>  <subject>
  • ...

Revert simulation:           clean | CONFLICT (aborted, see /tmp/revert-shas.txt for the offender)

Branch:                      revert/<FROM_TAG>-to-<TO_TAG>
Pushed:                      yes | no
PR:                          <url>
Auto-merged:                 no (intentional — human approval required)

Next steps (manual):
  1. Review the PR above.
  2. Merge with a merge commit (NOT squash — preserves the revert history).
  3. Run `staging-smoke-test` against production.
```
