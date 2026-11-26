---
name: is-done
description: Audit the current session and answer "are we finished with everything?" — list every item I asked for, every promise or follow-up you made, and every loose end (background agents/commands, worktrees, pending merges/deploys/checks, unanswered offers), verify each against live state rather than memory, and report what is done, what is still yours, and what is waiting on me. Read-only. Use it whenever I ask "is-done", "are we done?", "anything left?", "what's left here?", "did we finish everything?", or similar end-of-session questions.
allowed-tools: Bash, Read, Grep, Glob, TaskOutput, CronList
argument-hint: (none)
---

# Is Done

Answer one question honestly: **is everything from this session finished?** The answer comes from live state, not from what the conversation says happened.

This skill only reads. It never merges, pushes, posts, deploys, deletes or edits anything. If something is unfinished, report it and offer to do it.

## 1. Build the inventory

Go through the **whole** session, not just the last few turns. If the context was compacted, the summary names the transcript file. Read it (grep for user messages and for "I'll" / "after … I'll" / "once … merges") so items from before the compaction aren't lost.

Collect one row per item, in these kinds:

- **Asks:** everything I requested, including the small ones inside a longer message ("2. check progress, 4. open an issue").
- **Promises:** everything you said you would do later, e.g. "I'll deploy once it's merged", "I'll check the next cycle", "I'll report when the agent finishes".
- **Open questions:** offers or questions you asked that I never answered, and decisions I deferred.
- **Reminders for me:** things only I can do, e.g. a credential renewal, a console click, or a merge.
- **Loose ends you created:**
  - background shells and agents still running;
  - crons or wakeups;
  - scratch worktrees and branches;
  - temp files left on remote hosts;
  - labels or state you set temporarily.

## 2. Verify each row against live state

Never mark a row done because the conversation says so. Check it:

- **PRs and issues:** `gh pr view` / `gh issue view` for state, merged or not, CI (`gh pr checks`), review decision, unresolved threads, and labels.
- **Deploys:** is the target actually at the expected commit? Compare the remote `git rev-parse HEAD` with `origin/main`. Did the post-deploy check really show the expected line?
- **Background work:** has every background shell or agent sent its completion notification? A launched agent with no notification yet is **still running**, not done. Don't guess its result.
- **Worktrees and branches:** `git worktree list` in every repo you touched. Scratch worktrees should be gone. Remote branches you pushed are merged or intentionally left.
- **Local checkouts:** each checkout you touched is clean and on the branch it started on (`git status --short`, `git branch --show-current`), and not behind in a way that matters (for example, a deploy recipe read from a stale `Makefile`).
- **Scheduled items:** list crons and wakeups you created. Note any that are still armed.
- **External posts:** comments, reviews and issues you created exist at the URL you reported.

Run the checks in parallel where they are independent. If a check can't be run (no access, the host is down), say so and mark the row **unverified**. Never mark it done.

## 3. Report

Start with a one-line verdict:

- `✅ All done — nothing left.`
- `🟡 Done on my side — N items wait on you.`
- `🔴 Not done — N items are still mine.`

Then a table with one row per item, grouped in this order:

1. **Still mine:** unfinished asks, unkept promises, loose ends to clean up.
2. **Waiting on you:** merges, approvals, answers to my open questions, manual steps, reminders with their deadline.
3. **Waiting on time or others:** running background work, a cycle or CI that hasn't run yet, a human reviewer.
4. **Done:** each with the evidence that proves it, such as a URL, a sha or a log line. Keep this group short; collapse obvious ones into a single line.

Each row: the item, its status, the evidence or what's blocking it, and the next action (who does what).

End by offering to finish the "still mine" items. Ask first, because finishing may involve actions that need confirmation. Don't start them unasked.

## Rules

- **Live state wins.** If the transcript says "merged" but `gh` says OPEN, it's open.
- **No silent drops.** An item you decided not to do is still listed, with the reason. So is an offer I ignored.
- **Don't inflate.** Separate "done" from "done but unverified" and from "started".
- **Keep it short.** The table is the answer; no narrative recap of the session.
