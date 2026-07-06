---
name: rewrite-history
description: Rewrite commit history on the current branch into a coherent didactic story with step-by-step commits. Use when asked to clean up, reorganize, squash, or rewrite commits before opening a PR.
---

# Rewrite Commit History for Clarity

Reorganize the commits on the current branch to tell a clear, step-by-step story where:

- Each commit does exactly one thing
- Commit messages are detailed and explain the "why"
- Related changes are grouped together logically
- Project-specific commit rules (see Phase 2) are strictly enforced

## Workflow

### Phase 1: Analyze Current History

1. **Find the commit range** since diverging from the main branch:

   ```bash
   CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
   MAIN_BRANCH=$(git rev-parse --abbrev-ref origin/HEAD | cut -d'/' -f2)
   COMMIT_RANGE="${MAIN_BRANCH}..${CURRENT_BRANCH}"
   git log ${COMMIT_RANGE} --oneline --reverse
   ```

2. **Analyze each commit** to categorize:
   - What files changed (`git show --name-status <sha>`)
   - What kind of change (schema, migration, feature, refactor, test, docs)
   - Size of changes (line count)

3. **Identify logical groupings**: feature implementation steps, database changes, test additions, refactoring, documentation.

### Phase 2: Discover Project Conventions

Before planning, read the project's convention files — `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, or equivalents — and extract any rules about commit structure. Common examples:

- Commit message format (conventional commits, custom prefixes)
- Changes that MUST be committed together (e.g., a schema change and its generated migration)
- Changes that MUST be committed separately (e.g., database migrations never bundled with feature code)

Treat these rules as **non-negotiable** during reorganization. If the current history violates them, split, reorder, or restructure commits to fix it. If no convention file exists, fall back to the commit types listed below.

### Phase 3: Generate Reorganization Plan

For each logical unit, show:

```
□ Unit: [descriptive name]
  Commits to combine/reorder: [list]
  New commit message: [proposed message]
  Files affected: [list]
```

Then show a **before and after** visualization: the current commit list vs. the proposed reorganized list. Get the user's approval before executing.

### Phase 4: Execute the Rewrite

1. **Create a safety backup** first:

   ```bash
   git branch backup/${CURRENT_BRANCH}-pre-rewrite
   ```

2. **Rebuild the history non-interactively** (interactive `git rebase -i` doesn't work in agent sessions). The most reliable approach:

   ```bash
   git reset --soft $(git merge-base ${MAIN_BRANCH} HEAD)
   ```

   Then stage and commit each planned unit in order using `git add` (with `-p` or per-path staging to split files across commits) followed by `git commit`.

   For simple squash/reword-only plans, a scripted rebase also works:

   ```bash
   GIT_SEQUENCE_EDITOR="sed -i '' 's/^pick <sha>/squash <sha>/'" git rebase -i ${MAIN_BRANCH}
   ```

3. **Write each commit message** in this format:

   ```
   [type]: Brief one-line summary (50 chars max)

   Detailed explanation:
   - What changed
   - Why it changed
   - Any context needed to understand this commit
   ```

### Phase 5: Validate

- **Verify the final tree is identical** to the backup — this must show no output:

  ```bash
  git diff backup/${CURRENT_BRANCH}-pre-rewrite
  ```

- **Run the project's checks**: detect the typecheck/lint/test commands from `package.json` scripts, `Makefile`, `justfile`, or the convention files read in Phase 2, and run them.
- **Review the final history**: `git log --oneline -20`
- After the user confirms everything looks right, delete the backup branch.

## Commit Types

Default prefixes (project conventions override these):

- **feat**: New feature implementation
- **fix**: Bug fix
- **refactor**: Code reorganization without behavior change
- **test**: Test additions or fixes
- **docs**: Documentation
- **chore**: Maintenance, cleanup
- **db(schema)** / **db(migrate)**: Database schema and migration changes

## Notes

- Never force push — use `git push --force-with-lease` only after explicit user confirmation
- Each commit should be independently understandable and buildable
- If anything goes wrong mid-rewrite, restore with `git reset --hard backup/${CURRENT_BRANCH}-pre-rewrite`
