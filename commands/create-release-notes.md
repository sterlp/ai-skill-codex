Your goal is to create a short, informative `release-notes-<date>.md` for the changes since the last release tag.

1. Run `git tag --sort=-creatordate | head -10` to find the most recent release tag (format `vX.X.X` or `X.X.X`)
2. Run `git log <last-tag>..HEAD --oneline` to collect all commit messages since that tag
3. Run `git diff <last-tag>..HEAD --stat` to get a high-level overview of changed files
4. Analyze commits and changes — categorize each into: **New Features**, **Breaking Changes**, or **Bug Fixes**; skip merge commits and style-only changes
5. Write `release-notes-<date>.md` using this structure:
```markdown
# Release Notes — <date>

## Breaking Changes
- <short description> (`<affected class/module>`)

## New Features
- <short description> (`<affected class/module>`)

## Bug Fixes
- <short description, reference issue-xx.md if present>

## Notes
- <optional: migration hints, deprecations, known issues>
```

**Rules:**

- Run git tools in batch if you can
- Keep each bullet to one line — concise and factual
- Omit empty sections entirely
- Reference `<project>/issue-xx.md` files where relevant
- Breaking changes first — never bury them
- If no release tag exists, use the first commit as the baseline
- Write for the end user — avoid technical jargon, keep it short, precise, and scannable in under 30 seconds