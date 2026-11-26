# implement-in-session: Claude Code CLI

Read `SKILL.md` first. This file says how to start and talk to the session from the Claude Code CLI in a terminal.

## Start the session
1. Call `ListAgents`. Its first line gives the name of this session ("This session is <name>"). The new session sends its messages to that name.
2. Fetch: `git -C <repo-path> fetch origin`.
3. Write the prompt to a file in your scratchpad directory. First a "Background" part with the facts and decisions, then the brief, with this session's name filled in.
4. Start the session from the repo folder:
   ```bash
   cd <repo-path> && claude --bg -w <ticket> -n <ticket> --permission-mode auto "$(cat <prompt-file>)"
   ```
   - `--bg` starts the session in the background. `-w` gives it a new git worktree; `-n` names it.
   - The command prints a short session id. Keep it.
   - If it says "Workspace not trusted", ask the user to run `claude` in that folder once and accept the trust prompt. Then retry.

## Brief additions
- Worktree (brief step 1): `claude --bg -w` already created your worktree from `origin/main`. Rename its branch after the ticket: `git branch -m <ticket-branch>`. To continue an existing PR, switch to its branch instead: `git fetch origin <pr-branch> && git switch -c <pr-branch> --track origin/<pr-branch>`.
  - The worktree is at `<repo-path>/.claude/worktrees/<ticket>`. The main checkout shows it as untracked. Never stage it.
- Watch the PR (brief step 15): poll the PR every 5 minutes. There is no PR monitor in the CLI. To wait, run `sleep 300` as a background command; it wakes you when it exits. On each poll, read the CI state (`gh pr checks <n>`) and the unresolved review threads (GraphQL `reviewThreads`), and handle new comments. Stop when CI has finished and no unresolved threads remain.
- Talk to the starting session: it is named `<name>`. It does not see your turns, only your messages. Send every message to it with `SendMessage`. If a send fails, check the name with `ListAgents`.
  - Questions: as the brief's "Blocking questions" says.
  - Denied permissions: if a tool call is denied, do not try to work around it. Send what you need and why, then end your turn and wait.
  - Progress: send one short message when the PR is open, with its link.
  - Done: when "done when" is true, send a final message with the PR link, both ASD-STE100 reports and the decision summary.

## After starting
Tell the user in one line that the session started: its name, its short id, and the commands `claude attach <id>` (to open it) and `claude logs <id>` (to see its recent output). The session's messages arrive in this conversation. Answer it with `SendMessage` to its name.

If the session reports a denied permission, tell the user what it needs and why. Do not do the action for it from this session. The user can answer in the session itself with `claude attach <id>`.

When the PR is merged, or the user stops the work, offer to run `claude rm <id>`. It deletes the session, and its worktree when that is safe. Run it only after the user agrees.
