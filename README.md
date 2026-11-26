# Claude Code config

My personal Claude Code configuration (`~/.claude`): general-purpose skills and global instructions.

## Skills

| Skill | What it does |
|---|---|
| `situation-report` | Fast mid-session snapshot: goal, status, what's in flight, what's next, what waits on me. |
| `drive-task-to-done` | Own one task end to end without check-ins: plan, implement in a separate session, drive the PR to a verified merge, file every follow-up, report every decision. |
| `is-done` | End-of-session audit: every ask, promise and loose end, checked against live state. |
| `address-pr-comments` | Fix, reply to and resolve every review thread on a PR, then refresh its title and description. |
| `implement-in-session` | Start a separate session that implements an agreed change end to end, up to a reviewed PR. |
| `pull-main` | Switch every repo in the VS Code workspace to its default branch and pull. |

## Layout

- `skills/`: the public skills above.
- `CLAUDE.md`: global instructions. It imports `private/CLAUDE.md`.
- `private/`: a separate private repo, not part of this one. It holds org-specific skills, agents and agent memory, `context.md` with org constants, and a snapshot of `settings.json`. Its skills and agents are symlinked into `skills/` and `agents/`.

`.gitignore` is an allowlist, so everything else in `~/.claude` (transcripts, memory, caches, `settings.json`, the private symlinks) stays out of git. A new skill stays out until it is added to the allowlist.

## Org constants

Skills that need org values (GitHub org, follow-up issue repo, issue key prefix, and for `drive-task-to-done` the PR approver, automations and extra rules) read them from `~/.claude/private/context.md`. Without that file, `drive-task-to-done` uses safe defaults (listed in its Context table) and passes them on to the skills it calls; the other skills, run on their own, ask for the values.

## Working with this setup

**Add a public skill:** create `skills/<name>/SKILL.md`, then add `!skills/<name>/` and `!skills/<name>/**` to `.gitignore`.

**Add a private skill:** create `private/skills/<name>/SKILL.md`, then run `bash private/bin/link.sh` to symlink it into `skills/`.

**Settings:** `settings.json` is not tracked here. Claude Code reads only the real file in `~/.claude`. The private repo keeps a copy: run `bash private/bin/snapshot-settings.sh` before you commit there.

**New machine:**

1. Clone this repo to `~/.claude`.
2. Clone the private repo into `~/.claude/private`.
3. Run `bash ~/.claude/private/bin/link.sh`.
4. Copy `~/.claude/private/settings.json` to `~/.claude/settings.json`.

Without the private repo, the public skills still work. They ask for the org values that `context.md` would give them.
