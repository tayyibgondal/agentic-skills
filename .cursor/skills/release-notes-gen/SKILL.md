---
name: release-notes-gen
description: >-
  Generate a human-readable Markdown changelog between two refs
  (defaults to the previous tag → HEAD), grouped by
  conventional-commit type, and publish it as a GitHub Release on the
  tag via the `user-github` MCP. Standalone — does not require
  `release-tag` to have run first, so you can use it to backfill notes
  on an existing tag. Use when the user says "generate release notes",
  "draft changelog", "what changed since v1.2.0", or
  "publish release notes".
---

# Release Notes Generation

The "publish the changelog" skill. Reads the git log, formats it for
humans, and posts a GitHub Release.

## When to use this skill

- Right after `release-tag` (so the release page accompanies the new
  tag).
- Standalone, to backfill notes on an existing tag that was created
  manually.
- Standalone, to generate a "comparison" changelog between any two
  refs the user names (e.g. "what changed in the last week" via
  `HEAD@{1.week.ago}..HEAD`).

## Workflow

```
Task progress:
- [ ] Resolve from-ref (default: previous tag) and to-ref (default: tag or HEAD)
- [ ] Walk the commit range; classify by conventional prefix
- [ ] Group: features, fixes, refactor/perf, security/auth/docs/chore, other
- [ ] Render as Markdown with grouped headers, PR/commit links
- [ ] If the to-ref is a tag and no GitHub Release exists, create one via MCP
- [ ] If a Release already exists for this tag, refresh it (overwrite body, NOT title)
- [ ] Report the Release URL
```

### Step 1. Resolve refs

If the user gave neither, use:

```bash
TO_REF=$(git describe --tags --abbrev=0 --match 'v*' 2>/dev/null || echo HEAD)
FROM_REF=$(git describe --tags --abbrev=0 --match 'v*' "${TO_REF}^" 2>/dev/null || true)
```

- If `TO_REF` is a tag, point the release page at that tag.
- If `FROM_REF` is empty (this is the first release), walk from the
  repo's first commit.

If the user gave just one ref ("notes since v0.3.0"), use that as
`FROM_REF` and `HEAD` as `TO_REF`.

If the user gave both ("notes between v0.3.0 and v0.4.0"), use them
verbatim.

### Step 2. Walk and classify

```bash
git log --pretty=format:'%H%x00%s%x00%b%x1e' "${FROM_REF}..${TO_REF}"
```

Same classification rules as `release-tag`'s Step 3:

- `BREAKING CHANGE:` or `!:` → highlight at the top.
- `feat:` → Features.
- `fix:` → Fixes.
- `refactor:` / `perf:` → Refactor & performance.
- `security:` or `fix(security):` → Security (own section).
- `docs:` / `chore:` / `ci:` / `build:` / `test:` / `style:` → Other.
- Unclassified → Other (with `(unclassified)` suffix).

### Step 3. Render Markdown

The output goes into the GitHub Release body. Sections in order:

```markdown
## What's changed

<!-- BREAKING CHANGES (only if any) -->
### ⚠ Breaking changes
- <subject> ([commit](sha-link), [PR](pr-link))

### Features
- <subject> ([commit](...), [PR](...))

### Fixes
- <subject> ([commit](...), [PR](...))

### Refactor & performance
- <subject> ...

### Security
- <subject> ...

### Other
- <subject> ...

**Full changelog**: https://github.com/<owner>/<repo>/compare/v0.3.0...v0.4.0
```

Skip empty sections. Always include the "Full changelog" compare URL.

For each line, the link targets:

- commit link: `https://github.com/<owner>/<repo>/commit/<sha>`
- PR link: extracted from the commit subject (`(#NNN)` for squash
  merges, or `from <owner>/<branch>` for merge commits → look up via
  `gh pr list --state merged --search 'sha:<sha>'` or the MCP
  `pull_request_read` if installed).

Do NOT include the GitHub @-handles of contributors automatically.
GitHub Release UI will already attribute via the commit author.

### Step 4. Publish via MCP

Use `user-github` MCP. If the tag already has a Release, refresh the
body (keep the title); otherwise create a new Release:

```
CallMcpTool user-github releases_read {
  owner: "<owner>",
  repo: "<repo>",
  tag: "<TO_REF>"
}
```

Branch:

- 404 / not found → create:
  ```
  CallMcpTool user-github releases_create {
    owner: ...,
    repo: ...,
    tag_name: "<TO_REF>",
    name: "<TO_REF>",
    body: "<rendered markdown>",
    draft: false,
    prerelease: false
  }
  ```
- 200 / found → update body only:
  ```
  CallMcpTool user-github releases_update {
    owner: ...,
    repo: ...,
    release_id: <existing-id>,
    body: "<rendered markdown>"
  }
  ```

If the MCP call returns `403 Resource not accessible by personal
access token`, the configured PAT lacks `contents:write`. Fallback:
print the rendered markdown so the user can paste it manually at:

```
https://github.com/<owner>/<repo>/releases/new?tag=<TO_REF>
```

…and stop the skill at that point. Note the PAT gap.

### Step 5. Report

Print the Release URL, the resolved `FROM_REF`..`TO_REF`, and the
count of commits per section.

## Hard Rules (never break)

- **Never mark a Release as a draft or prerelease automatically.** If
  the user wants that, they say so in the prompt; otherwise the
  Release is published immediately.
- **Never delete an existing Release.** Only refresh the body.
- **Never edit a Release's title** without an explicit user prompt
  ("rename the v0.4.0 release to …"). Title is sticky once created
  to avoid breaking deep-links.
- **Never modify the tag pointer** — that's the `release-tag` skill's
  job, and only via `release-tag` itself.
- **Never include commit author emails** in the Release body. GitHub
  shows commit authors via its own UI; duplicating them in the body
  is noisy and may expose private addresses.

## Trigger Examples

- "generate release notes"
- "draft changelog"
- "what changed since v1.2.0"
- "publish release notes for v0.4.0"
- "run the release-notes-gen skill"

## Output Template

End the run with:

```
Release-notes-gen summary

Range walked:                <FROM_REF>..<TO_REF>
Commits:                     <N>
  - Breaking:                <N>
  - Features:                <N>
  - Fixes:                   <N>
  - Refactor / Perf:         <N>
  - Security:                <N>
  - Other:                   <N>

GitHub Release:              created | refreshed
Release URL:                 <url>
PAT fallback used:           yes | no

Next step (optional): run `staging-smoke-test` to verify the production deploy is healthy.
```
