# implement-in-session: Claude desktop app

Read `SKILL.md` first. This file says how to start and talk to the session from the Claude desktop app.

## Start the session
Call `start_session` with `initiation: "user_asked"`, `context: "fresh"`, `use_worktree: false` (the session creates its own worktree, see the first brief addition). Put the facts and decisions in `background`, and the brief in `prompt`.

## Brief additions
- Worktree (brief step 1): create a git worktree in the sibling folder `<repo-path>-worktrees/` from a freshly fetched `origin/main`, on a branch named after the ticket.
- Watch the PR (brief step 15): bind the PR with the `ccd_pr` tools and turn on the app's PR monitor/Auto-fix. It replaces any "check every N minutes" polling.
  Fallback: if turning on the monitor fails or asks the user for permission, do not wait for an answer. Poll the PR every 5 minutes yourself instead. To wait, run `sleep 300` as a background command; it wakes you when it exits. On each poll, read the CI state (`gh pr checks <n>`) and the unresolved review threads, and handle new comments. Stop when CI has finished and no unresolved threads remain.
- Talk to the starting session: send questions to it with `SendMessage` (find it with `ListAgents`). Also put a question at the start of your turn's final report: the starting session sees only the first 400 characters of it.

## After starting
Tell the user in one line that the session started (link it) and how you will report back. The session's notes arrive in this conversation when it ends a turn. Send it answers with `SendMessage`, using its `local_…` session id.
