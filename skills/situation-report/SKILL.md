---
name: situation-report
description: Use when I ask for a quick mid-session status check, like "where are we?", "status?", "what's the current status?", "where does this stand?", "where it stands", "what are you doing now?", "what's running?", "catch me up" or "sitrep". It gives a fast snapshot of the current session: the goal, what's done, what's in flight, what's next and what's blocked on me. It's read-only and lighter than /is-done. Use /is-done instead for the full end-of-session "are we finished with everything?" audit.
allowed-tools: Bash, Read, TaskOutput, CronList
argument-hint: (none)
---

# Situation Report

Give me a **fast snapshot** of this session so I can get back on track in about 10 seconds of reading. Speed matters more than completeness here. The full audit is `/is-done`.

This skill only reads. It never merges, pushes, posts, edits or starts work.

## How to stay fast

- **Work from the conversation you already have.** Don't re-read the transcript, and don't grep old messages. If the context was compacted, use the summary as it is.
- **Do at most one round of live checks, all in one parallel batch.** Only check things whose state may have changed since you last saw them:
  - a background agent or shell that hasn't sent its completion notification (use `TaskOutput`, or treat it as still running);
  - CI, review state or merge state of the PR currently being worked on (`gh pr view <n> --json state,mergeStateStatus,reviewDecision,statusCheckRollup`);
  - a deploy or cycle you're waiting on.
- **Skip checks for anything already confirmed** earlier in the session (merged PRs, finished agents, verified deploys). If nothing could have changed, run no tools at all.
- **No subagents, no worktree sweeps, no per-repo `git status` loops, no inventory of every past ask.** Those belong to `/is-done`.

## Output

At most about 12 lines, in this shape. Leave out any section that would be empty.

```
**Goal:** <one line: what we're trying to get done right now>
**Status:** <one line: where it stands, e.g. "PR #321 open, CI green, waiting on review">

**Done:** <short comma-separated list of the main milestones; no evidence dump>
**In flight:** <running agents/shells/CI/deploys, with live state from the check>
**Next (me):** <the very next thing you will do>
**Waiting on you:** <decisions, approvals, answers to open questions, manual steps>
```

Rules for the content:

- **Live state wins.** If a check contradicts what the conversation says, report the checked state and say so in a few words.
- **Running isn't done.** A background agent without a completion notification is "running". Never guess its result.
- **If a check couldn't run**, say "unverified" for that line. Don't drop it.
- **If the session has several threads of work**, give one `Goal`/`Status` pair per thread (max 3) and fold the rest into one "Other:" line.
- End with one line: offer `/is-done` if it looks like we're near the end, or otherwise offer to continue with "Next (me)". Don't start anything unasked.
