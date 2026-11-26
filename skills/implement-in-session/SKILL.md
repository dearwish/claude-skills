---
name: implement-in-session
description: Start a separate Claude Code session that implements an agreed change end to end — own worktree, reviewed plan, reviewed implementation, UI review when a user can see the change, PR, review-comment handling, and a summary of its decisions. Works from the Claude desktop app and from the Claude Code CLI. Use when the user says to "start a session/PR for it" after a change has been investigated and decided in the current session.
---

# Implement in a new session

**Org context:** read `~/.claude/private/context.md` first. It defines `<org>` (GitHub org), `<follow-up-repo>` (where follow-up issues go) and `<KEY>` (issue-tracker key prefix), plus any extra org rules. If the file is missing, use the values the calling skill passed. If none were passed, ask me for them.

Use after the current session has investigated a change and the user (and any approver) has decided it. This session stays with the user; a new session does the work.

**Where you run decides how you start and talk to the session.** This file holds what is the same everywhere. Read ONE of these as well:
- `desktop.md` (next to this file): you run in the Claude desktop app, so the `start_session` tool is available.
- `cli.md` (next to this file): you run in the Claude Code CLI in a terminal, so start the session with `claude --bg`.

## 1. Collect before starting
- The ticket key, the decided change in one sentence, and who approved it.
- The agreed approach: which component or layer owns the change, the data or control flow, and any interface or contract change. The plan must follow it, not derive a new design.
- Acceptance criteria: what behaves differently after the change, and the test or check that proves it. Also say whether a user can see the change, because that decides the UI review.
- What is IN scope: files, functions, file:line anchors.
- What must stay UNCHANGED, and options the user rejected (with the reason), so the new session does not re-propose them.
- What is OUT of scope but affected (other services, infra repo, alerts, dashboards): the session must list these as follow-ups, not change them.
- The measured facts that justify the change (numbers, dates, sources).
If any of these is unclear, ask the user before starting.

## 2. Start the session
Start it as `desktop.md` or `cli.md` says. Give it the facts and decisions from step 1 as background, and the brief below as its task. Put that method file's "Brief additions" into the brief where the steps below point to them.

Brief — the steps, in this order:
1. Work in your own git worktree from a freshly fetched `origin/main`, on a branch named after the ticket (how: the method's brief additions). If the brief names an existing PR to continue, work on that PR's branch instead and open no new PR. Work only in the worktree. Never edit, commit or switch branches in the main checkout.
2. Write an implementation plan. Never commit it.
3. Spawn a fresh plan-review subagent. Give it the agreed approach and the acceptance criteria, so it checks that the plan follows the one and meets the other. If it reports issues, verify each claim yourself against the code before you change the plan. Do not spawn a verifier subagent for the plan. Open the file and line the claim cites. Drop a claim only when the code shows it is wrong, and note that evidence. If the code does not settle it, keep the claim.
4. If verified issues affect the implementation and are not Low, rewrite the plan and go back to step 3.
5. Apply the remaining verified notes and start implementing.
6. Finish the implementation. Run lint, typecheck and the affected tests.
7. Split the diff into review groups only if it is large:
   - Small (about 15 changed files or fewer, or one concern): one group, no split. For a borderline size, decide yourself.
   - Large: split the changed files into groups by function, for example "DB schema and repositories", "API routes", "frontend export".
     - Each file is in exactly one group, and together the groups cover every changed file.
     - Tests go in the group of the code they test.
     - Check this with a script against `git diff --name-only origin/main...`, not by eye.
   Do not ask anyone to approve the groups.
   For each group, spawn two fresh reviewer subagents in parallel:
   - functionality/correctness;
   - style, which follows the repo's `style-review` AND `test-review` skills, plus CLAUDE.md.
   Give each reviewer the worktree, the diff and its group's files, not your conclusions. Also give the functionality reviewers the acceptance criteria. A reviewer may read outside its group to follow a call, but reports findings only in its group's files. Three groups give six reviewers.
8. Give each reviewer its own verifier subagent. A verifier starts as soon as its own reviewer finishes, and waits for no other agent. Skip it when its reviewer found nothing. A style verifier checks each claim against the `style-review` / `test-review` rules. A functionality verifier checks that the failure can happen.
9. If verified Medium/High issues exist, or this is the first Low-only round, implement the fixes that genuinely help, then go back to step 7 with fresh reviewers. Keep the groups, and add any new file the fixes touched to the group it belongs to. Record each round in the PR description.
10. If a user can see the change, run the repo's `ui-review` skill and follow it fully: its rounds, its stop rule, and its cleanup of browsers and stacks. Do not drive a browser your own way: the skill sets up the browser so that it does not use all of the machine's resources. Record its rounds in the PR description. If no user can see the change, say so in the PR description. Also do what the repo's CLAUDE.md and the org context file require for a change that users can see or do.
11. Report in ASD-STE100: issues fixed in the review rounds, and leftover issues from the last round with the reason each was not done.
12. Report in ASD-STE100: goal, architecture, and how the new code relates to it.
13. Write the decision summary (see below).
14. Open the PR against `main` with `gh` (goal, facts, what changed, what stays, the decision summary, out-of-repo follow-ups, review rounds, UI review rounds or why none was needed; attribution line at the end).
15. Watch the PR as the method's brief additions say. For each review comment: fix it in this PR. Reply "not a defect" with the reason only when the comment is wrong. The only exception is a comment that is completely unrelated to the work of this PR: a different service or feature, with no path from this diff to it. For that comment, file a follow-up issue in `<follow-up-repo>` (never in the code repo), and reply with its link. More work, a larger diff or a preference is not a reason to file a follow-up. Commit and push if code changed. Reply on every unresolved thread and resolve it. Continue until CI finishes and no unresolved threads remain. If this changes a decision, update the decision summary in the PR description.

Decision summary: a short list of every decision you made that the brief did not settle, most important first. Use ASD-STE100. For each decision, write one or two sentences:
- what you decided, and the reason;
- who decided: you, the starting session (when the owner did not answer in time), or the user (quote the user's words).
After the list, write a separate list of the things that the user approved during the run. Keep the summary short, but a reader who did not watch the run must understand each item without the code.

Blocking questions: if an architecture question stops you, send it to the starting session (how: the method's brief additions). Give the question, the options, and your recommendation. Then end your turn and wait. Do not open the PR while a question is open. The reply is either the owner's answer or the starting session's own decision. Continue with either one. If the starting session later sends a correction, it overrides that reply. Every answer goes into the decision summary.

Also include in the brief: the repo rules to respect (read CLAUDE.md; explicit `git add` paths only; never commit plans or reports), shell notes, the "ASD-STE100 rules" below word for word, and "done when": acceptance criteria met and checked, UI review done or not needed, PR open, CI finished, all threads resolved, both ASD-STE100 reports and the decision summary sent, follow-ups listed.

## 3. After starting
Tell the user in one line that the session started, as the method file says. Do not wait for it. When its final report arrives, check its result (diff/PR) before you tell the user it is done. Show the user its decision summary.

If the session asks an architecture question, it is blocked until you reply:
1. Put the question to the user in chat, with the options, its recommendation and yours. Do not use `AskUserQuestion`: it blocks your turn, so the timer below could not wake you.
2. In the same turn, start `sleep 600` as a background command; it wakes you after 10 minutes.
3. If the user answers first, send the answer to the session, quoting the user's words. When the timer later fires, do nothing.
4. If the timer fires with no answer, decide yourself. Base the decision on the decided change, the repo's CLAUDE.md and the code. Send it to the session, marked as your decision and not the owner's. Tell the user what you decided and why, so they can overrule it before the PR merges.

## Autonomous callers
Another skill can start the session and act as the starting session, for example `/drive-task-to-done`. When that skill runs without questions to the user, its rules replace these steps:
- § 1: it fills a gap in the brief from the approved plan and the code. It does not ask the user.
- § 3, architecture questions: it decides at once, without the timer. It sends the decision marked as its own and names it in its progress line, so the user can overrule it before the merge.
- § 3, denied permissions: it records them as items for the user and continues with other work.
- Org context: it passes the context values in the brief. Use them and do not ask for them.
- `cli.md`, `claude rm`: it runs it after the PR merges, without asking.
Its additions to the brief (branch name, PR rules, reviewers, test limits) override this file where the two conflict.

## ASD-STE100 rules
- Short sentences: 20 words or fewer for an instruction, 25 for a description.
- One instruction in each sentence. Active voice, simple tenses.
- No "-ing" verb forms: write "to read the file", not "reading the file".
- One word for one meaning. Use the same name for the same thing every time.
- Common words: "use", "make sure", "start", "help" (not "utilize", "ensure", "initiate", "facilitate").
- Keep the articles ("the", "a"). Do not write in telegraph style.
- Keep code names, file paths, commands and error text exactly as they are.
