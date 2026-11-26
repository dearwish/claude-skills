---
name: pull-main
description: Switch every project in the current VS Code workspace to its default branch (main) and pull the latest from origin. Use when the user asks to "pull main", "update all repos", "sync workspace", or similar.
model: sonnet
allowed-tools: Bash(git *)
---

# Pull Main Across Workspace

Switch every project in the current VS Code workspace to its default branch and fast-forward it to the latest `origin`.

## Workspace projects

The workspace projects are the **primary working directory** plus every **additional working directory** listed in the environment section of the system prompt.

Before processing, filter the list down to git repositories — for each path, check `git -C <path> rev-parse --git-dir` succeeds. Silently drop non-git paths (skill folders, config dirs, etc.) without listing them in the final report.

## Steps — for each project

Run these checks/commands per repo. Use `git -C <path>` rather than `cd`. Do the independent per-repo steps in parallel Bash calls where possible.

### 1. Detect the default branch

```sh
git -C <path> symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null | sed 's|^origin/||'
```

Fallback order if the above is empty: `main` if it exists, else `master`. If neither exists, skip this repo and report it.

### 2. Capture current state

- Current branch: `git -C <path> branch --show-current`
- Dirty tree: `git -C <path> status --porcelain` (non-empty = dirty)

### 3. Handle dirty trees safely

If the tree is dirty, **do not discard work**. Stash with a clear label before switching:

```sh
git -C <path> stash push --include-untracked -m "pull-main skill auto-stash $(date +%s)"
```

Remember you stashed so you can restore after the pull.

### 4. Switch to the default branch (if not already on it)

```sh
git -C <path> switch <default-branch>
```

### 5. Fast-forward pull

```sh
git -C <path> pull --ff-only
```

If `--ff-only` fails (local `main` has diverged from origin), **stop for that repo** and report it — do not force-pull or reset. The user needs to resolve it.

### 6. Restore stashed work

If you stashed in step 3, pop it back:

```sh
git -C <path> stash pop
```

If `stash pop` conflicts, leave the stash in place and tell the user — they can resolve manually via `git stash list`.

## Reporting

After processing every repo, print a short per-repo summary:

- `<repo>`: `<prev-branch>` → `<default-branch>`, fast-forwarded `<old-sha>..<new-sha>` (N commits) | already up-to-date | skipped: `<reason>`
- Note any stashes restored or left pending.

Keep the final report to one line per repo plus a brief trailer if any action is required from the user.

## Safety

- Never run `git reset --hard`, `git checkout -- .`, `git clean -f`, or `git push --force`.
- Never drop stashes automatically — if `stash pop` fails, leave it for the user.
- If the current branch has unpushed commits and is not the default branch, switching is fine (commits stay on that branch). Just report it so the user knows where their WIP lives.
