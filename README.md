# Claude Code config

My personal Claude Code configuration (`~/.claude`): general-purpose skills and global instructions.

## Skills

| Skill | What it does |
|---|---|
| `situation-report` | Fast mid-session snapshot: goal, status, what's in flight, what's next, what waits on me. |
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

Skills that need org values (GitHub org, follow-up issue repo, issue key prefix) read them from `~/.claude/private/context.md`. Without that file they ask for them.
