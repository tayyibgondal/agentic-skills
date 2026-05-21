---
name: release-tag
description: >-
  Cut a SemVer release tag on `main`. Decides the next version
  (`vX.Y.Z`) by walking conventional-commit prefixes (`feat:` → minor,
  `fix:`/`refactor:`/`perf:` → patch, `BREAKING CHANGE:` footer →
  major) between the previous tag and HEAD. Creates an annotated tag,
  pushes it, and appends a new section to `release-notes.txt`. Run
  AFTER `promote-staging-to-main` so the tag points at the freshly
  promoted commit. Use when the user says "tag this release", "cut a
  release tag", or "add release tag".
---

# Release Tag

The "what version am I shipping?" skill. Reads the commit log between
the previous tag and `main`, infers a SemVer bump, and stamps the tag.

## When to use this skill

- IMMEDIATELY after `promote-staging-to-main` completes — that skill
  brings `main` to the latest state, this one labels it.
- NEVER on `staging` or a feature branch.

## Workflow

```
Task progress:
- [ ] Fetch origin (tags + main)
- [ ] Verify we are tagging the HEAD of origin/main (not a stale local main)
- [ ] Identify the previous tag (or detect "no prior tag" mode)
- [ ] Walk commits between previous tag and HEAD; classify each
- [ ] Compute next version
- [ ] Refuse if HEAD is already tagged
- [ ] Generate release notes section
- [ ] Append to release-notes.txt
- [ ] Create annotated tag with the notes body
- [ ] Push tag
- [ ] Report
```

### Step 1. Fetch

```bash
git fetch origin --tags --prune --prune-tags
git rev-parse origin/main  # capture the SHA we're tagging
```

### Step 2. Identify previous tag

```bash
PREV_TAG=$(git describe --tags --abbrev=0 --match 'v*' origin/main 2>/dev/null || true)
```

- If `PREV_TAG` is non-empty: walk commits between it and HEAD.
- If empty (first release): walk every commit reachable from HEAD,
  default starting version to `v0.1.0`.

### Step 3. Walk commits and classify

```bash
git log --pretty=format:'%H%x00%s%x00%b%x1e' "${PREV_TAG:+${PREV_TAG}..}origin/main"
```

For each commit, split on `\x00` then process each record (records are
`\x1e`-separated). Classify by:

- Body or footer contains `BREAKING CHANGE:` or subject contains `!:`
  (e.g. `feat!: …`) → **major**.
- Subject starts with `feat:` or `feat(scope):` → **minor**.
- Subject starts with `fix:`, `refactor:`, `perf:`, `revert:`,
  `chore:`, `docs:`, `style:`, `test:`, `build:`, `ci:` → **patch**.
- Anything else → **patch** with a `(unclassified)` note in the
  release log.

Skill computes the bump as `max(per-commit classifications)`.

If the bump is **major**, the skill REFUSES to auto-bump unless the
user has explicitly typed "major bump" or "breaking release" in the
prompt. Otherwise it stops and asks — major bumps on a public product
should be a deliberate decision.

### Step 4. Compute next version

Parse `PREV_TAG` as `vMAJOR.MINOR.PATCH`. Apply:

- patch bump: `vMAJOR.MINOR.(PATCH+1)`
- minor bump: `vMAJOR.(MINOR+1).0`
- major bump: `v(MAJOR+1).0.0`

For first release (no prior tag): use `v0.1.0` regardless of commit
classification. The first tag is always `v0.1.0` so users can rely on
"missing tag means zero-state".

### Step 5. Refuse if HEAD is already tagged

```bash
EXISTING=$(git tag --points-at origin/main 'v*')
if [ -n "$EXISTING" ]; then
  echo "HEAD already tagged: $EXISTING — refusing to re-tag"
  exit
fi
```

This guards against running the skill twice on the same `main`.

### Step 6. Generate release notes section

The notes section appended to `release-notes.txt` has this shape:

```
================================================================
v0.4.1  —  2026-05-20  —  SHA <short>
================================================================

Features
  - <subject>  (#<pr>)
  - ...

Fixes
  - ...

Refactor / Performance
  - ...

Other
  - ...

Full changelog: https://github.com/<owner>/<repo>/compare/v0.4.0...v0.4.1

```

Order sections by importance: Features → Fixes → Refactor/Perf → Other.
Skip empty sections (don't print "Features" if there are zero feat
commits).

For each commit subject, strip the conventional prefix (`feat: foo`
→ `foo`), and append `(#NNN)` if a PR number is recoverable from
`git log --format=%s` (commit messages from squash merges include the
PR number; merge commits include it via `from <owner>/<branch>`).

### Step 7. Append to `release-notes.txt`

Read the existing file. Insert the new section at the TOP (newest
first). Keep the rest of the file unchanged. If the file doesn't
exist, create it with just the new section.

```bash
TMP=$(mktemp)
{ printf "%s\n" "$NEW_SECTION"; cat release-notes.txt; } > "$TMP"
mv "$TMP" release-notes.txt
```

The release-notes-update happens BEFORE the tag is created so the
update is included in the tag's annotation (Step 8 reads the file
content for the annotation body).

Commit the release-notes.txt update on `main` directly (this is the
ONE write to `main` this skill makes — it's a release-record change,
not a code change):

```bash
git checkout main
git pull --ff-only origin main
git add release-notes.txt
git commit -m "release: $NEW_TAG notes"
git push origin main
```

If `release-notes.txt` was already updated for this version in a
prior aborted run, the skill refreshes the section in place (not
duplicates it).

### Step 8. Create the annotated tag

```bash
git tag -a "$NEW_TAG" -m "$(cat <<EOF
Release $NEW_TAG

$NEW_SECTION
EOF
)" origin/main
git push origin "$NEW_TAG"
```

Annotated (`-a`) — not lightweight — so `git describe` works and so
GitHub renders the release page with a body.

### Step 9. Report

Print the tag, the SHA it points at, the version-bump rationale, and
the URL of the new tag on GitHub.

## Hard Rules (never break)

- **Tags are ONLY on `main`.** The skill refuses to tag `staging` or
  any feature branch. Hardcoded `origin/main` everywhere.
- **NEVER re-tag an existing SHA.** Refuses if `git tag --points-at
  origin/main` returns non-empty.
- **NEVER skip a patch number.** A tag advances from the previous
  immediate version — never `v0.4.0` → `v0.6.0` because the agent
  miscounted.
- **NEVER force-push a tag.** `git push origin <tag>` only — no
  `--force`, no `:refs/tags/<tag>` deletes.
- **NEVER bump major without explicit user opt-in.** "feat: foo!"
  alone triggers a stop-and-ask, not an auto-bump.
- **NEVER tag a `main` that is behind `origin/main`.** Always
  `git pull --ff-only` first; refuse on non-fast-forward.
- **NEVER touch a tag that has already been pushed.** Stop and ask.

## Trigger Examples

- "tag this release"
- "cut a release tag"
- "add release tag"
- "tag v0.4.0"  (explicit version overrides auto-bump)
- "run the release-tag skill"

## Output Template

End the run with:

```
Release-tag summary

Previous tag:           v0.4.0          (sha: <short>, dated: <iso>)
New tag:                v0.4.1          (sha: <short>, dated: <iso>)
Bump type:              patch | minor | major (rationale: <N feat, M fix, K perf>)

Commits walked:         <N>
  - feat:               <N>
  - fix:                <N>
  - refactor / perf:    <N>
  - chore / docs / ci:  <N>
  - unclassified:       <N>

release-notes.txt:      appended section (was: yes | no) — pushed: <main SHA>
GitHub tag URL:         https://github.com/<owner>/<repo>/releases/tag/<tag>

Next step (optional): run `release-notes-gen` to also publish a GitHub Release page from this tag.
```
