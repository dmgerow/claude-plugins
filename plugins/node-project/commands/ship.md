---
description: Pre-flight check, commit, push, open PR, watch CI, fix failures, squash-merge, and babysit release + deploy workflows to completion
argument-hint: "[commit message or description of changes]"
allowed-tools: Bash(git:*), Bash(gh:*), Bash(npm run:*), Bash(npm test:*), Bash(npx tsc:*), Bash(npx eslint:*), Read, Edit, Glob, Grep
---

## Context

- Current branch: !`git branch --show-current`
- Git status: !`git status --short`
- Git diff (staged + unstaged): !`git diff HEAD`
- Worktree check: !`git rev-parse --show-toplevel`

## Your task

Ship the current changes end-to-end: pre-flight → commit → PR → watch CI → fix if needed → merge → watch release-please → merge release PR → watch deploy to completion.

Arguments (description of changes or commit message hint): `$ARGUMENTS`

### Step 1 — Pre-flight checks

Run these checks locally first. Fix any failures before committing — catching issues here is faster than waiting for CI.

1. `npm run lint` — if there are fixable errors, run `npm run lint -- --fix` and re-check
2. Type check:
   - If `package.json` has `"build"` using `tsc -b` (summit): `npx tsc -b --noEmit` (or `npm run build`)
   - Otherwise: `npx tsc --noEmit`
3. `npm run build` (skip if typecheck already builds)
4. `npm test` — if tests exist and are relevant to the changed files

If any step fails and cannot be auto-fixed, stop and report the issue. Do not proceed to commit.

### Step 2 — Commit and push

1. If currently on `main`, you are inside a worktree (path contains `.claude/worktrees/`) — use `git checkout -b <branch-name>` to create a branch. Branch name should be kebab-case description of the change.
2. Stage all changes: `git add -A`
3. Commit with a Conventional Commits message:
   - Use `$ARGUMENTS` as a hint if provided
   - Format: `type: description` (e.g. `feat: add export button`, `fix: correct rounding error`)
   - Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`
4. Push: `git push -u origin HEAD`

### Step 3 — Open PR

The PR title **must** be in Conventional Commits format — this is enforced by a CI check and will fail if wrong.

```bash
gh pr create --title "<type>: <description>" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points describing the change>

## Test plan
- [ ] Pre-flight checks passed locally (lint, typecheck, build, test)
- [ ] CI passing on GitHub

🤖 Shipped with /ship
EOF
)"
```

Capture the PR number and URL from the output.

### Step 4 — Watch CI

Use `gh run watch` to monitor CI:

```bash
gh pr checks <PR_NUMBER> --watch --interval 30
```

Track fix attempts by counting commits with `[ci-fix]` in the message:
```bash
FIX_ATTEMPTS=$(git log --oneline | grep -c '\[ci-fix\]' || true)
```

**If the PR title validation check fails**, fix the title immediately and re-check:
```bash
gh pr edit <PR_NUMBER> --title "<type>: <description>"
```

**If all checks pass** → proceed to Step 5.
**If CI fails for other reasons** → proceed to fix loop (Step 4a).

### Step 4a — CI fix loop (max 3 attempts)

If `FIX_ATTEMPTS >= 3`: stop, post a comment, and report to the user:
```bash
gh pr comment <PR_NUMBER> --body "CI auto-fix reached 3 attempts — manual intervention needed."
```
Then show the user the final failure output and exit.

Otherwise:

1. Get the failure logs:
   ```bash
   RUN_ID=$(gh run list --branch <BRANCH> --limit 1 --json databaseId --jq '.[0].databaseId')
   gh run view $RUN_ID --log-failed
   ```

2. Analyze the error. Common categories:
   - **Lint error** → fix the specific lint violation
   - **Type error** → fix the TypeScript issue
   - **Build error** → fix the compilation or import issue
   - **Test failure** → fix the failing test or the code it tests
   - **Dependency issue** → check `package.json`

3. Fix the code. Re-run the failing check locally to verify the fix.

4. Commit the fix:
   ```bash
   git add -A
   git commit -m "fix: resolve CI failure [ci-fix]"
   git push
   ```

5. Return to Step 4 (watch CI again).

### Step 5 — Merge

Once CI passes:

```bash
gh pr merge <PR_NUMBER> --squash --delete-branch
```

Then proceed immediately to Step 6 — do not stop here.

### Step 6 — Watch release-please workflow

After merging, a `release-please` workflow will trigger on `main`. Find and watch it:

```bash
gh run list --limit 5
```

Identify the in-progress run on `main` triggered by the merge push (workflow name contains "release-please"). Then watch it:

```bash
gh run watch <RUN_ID> --interval 30
```

**If release-please creates a release PR** (look for a PR titled `chore(main): release ...`):

```bash
gh pr list --json number,title --jq '.[] | select(.title | test("release"; "i")) | .number'
```

Merge it immediately:
```bash
gh pr merge <RELEASE_PR_NUMBER> --squash --delete-branch
```

**If release-please does not create a release PR** (e.g. it was a `chore:` or `ci:` commit that doesn't trigger a release), skip to the end and report.

**If the release-please workflow fails**, inspect the logs:
```bash
gh run view <RUN_ID> --log-failed
```
Note: A failure in the "Assign release PR" step (exit code 127, `gh` not found) is non-fatal — the release PR is still created. Confirm the PR exists before treating this as a real failure.

### Step 7 — Watch deploy workflow

After merging the release PR, a new workflow run will trigger on `main` that builds and pushes the Docker image and triggers the Portainer redeploy. Find and watch it:

```bash
gh run list --limit 5
```

Identify the new in-progress run triggered by the release PR merge, then watch it to completion:

```bash
gh run watch <RUN_ID> --interval 30
```

If it fails, inspect logs:
```bash
gh run view <RUN_ID> --log-failed
```
Report the failure to the user — do not attempt to auto-fix deploy failures.

Once the deploy run completes successfully, report:

> Shipped and deployed! PR #N merged → release PR #M merged → vX.Y.Z deployed successfully.
