---
name: drive-task-to-done
description: Use when I ask to own one task (a GitHub issue, or a tracker ticket that links to one) end to end without check-ins, for example "drive-task-to-done owner/repo#123", "take this issue to merged without asking me" or "babysit #45 until it's done". It plans, implements in a separate session, drives the PR through review and CI to a verified merge, files every follow-up, and loops until /is-done is clean, then reports every decision and why. Safe to re-run, because it resumes from live state. If an org-specific drive-to-done skill is installed, use that one for that org's tasks.
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Skill, Agent, Monitor, TaskOutput, ListAgents, SendMessage
argument-hint: <issue-url | owner/repo#N | repo#N | #N>
---

# Drive Task to Done

Take one task from wherever it stands to finished, **without asking me anything mid-flight**. Decide, act, and log each decision with its reason. The only output while it runs is a one-line progress note at each state change. At the end comes the final report (§ 9).

**Autonomy is not licence.** The rules in § Rules override autonomy. When one blocks a step, do everything else, and put the blocked step in the report as an item for me. Never invent a status to get past it.

## Context

Read `~/.claude/private/context.md` first, if it exists. The values below come from it. A value that is missing uses its default.

| Value | Meaning | Default |
|---|---|---|
| `<org>` | GitHub org of the code repos | the owner of the current directory's repo (`gh repo view --json owner -q .owner.login`) |
| `<follow-up-repo>` | where follow-up issues go | the task's own repo |
| `<KEY>` | issue-tracker key prefix, if a tracker sits beside GitHub | none: GitHub issues only |
| `<checkouts>` | root of the local checkouts | `~/work` |
| `<approver>` | who approves a PR I authored, and how to ask them (a bot, a person, a channel) | none: the PR waits on a human approver and is never merged by this run |
| `<automations>` | agents that may own a task, their labels, their plan headers, and their takeover label | none: every task is local |
| `<started>` | how to show that work began: a tracker status (for example "In Progress"), a label, or a project field | none: assignment only (§ 2a) |
| extra rules | release rules, protected repos, reviewers never to request, merge rules | none |

Resolve these values once, in § 1, and pass them on: sub-skills must not ask for them.

### Autonomy preamble (pass it word for word in `args` to every skill and subagent this run starts, with the values filled in)

> AUTONOMOUS MODE (drive-task-to-done). Context values (use these, even if `~/.claude/private/context.md` is missing; do not ask for them): `<org>`=…, `<follow-up-repo>`=…, `<KEY>`=…. The owner pre-granted every confirmation this skill would ask for, including `claude rm` of a session after its PR merges. Do not use AskUserQuestion and do not stop to show a draft. Where the skill says "show me / confirm / ask", do the step as written and return it with its URL. Where it says "needs a decision", choose your recommendation and return the choice with a one-line reason. The pre-grant covers only the owner's confirmations: a question for someone else still goes to them, and you wait for their answer. Never merge, release or deploy. Never `git add -A`; stage explicit paths. Never switch branches in a checkout under `<checkouts>`; use a scratchpad worktree. Never write to production systems. Fix review comments in the PR they were made on. The only exception is a comment completely unrelated to the PR's work: file a follow-up in `<follow-up-repo>`, reply with its link and resolve the thread. Return: verdict, what you posted (URLs), decisions with reasons.

## 1. Resolve the task

- A URL or `owner/repo#N` → as given. `repo#N` → `<org>/repo#N`. `#N` → issue `N` of the current directory's repo. If `<org>` or the current repo can't be resolved, say that a full `owner/repo#N` is needed and stop.
- **No argument:** say that the skill drives one named task and stop. It never picks a task itself.
- A tracker ticket that links to a GitHub issue → drive the GitHub issue, and keep the ticket key for status checks.
- **The task id** `<id>`: `<KEY>-<N>` when the task has a tracker key, else `<repo>-<N>` (for example `api-123`). Use it in the branch name, the PR title, the session name and the ledger file name. It never contains `#`.
- **The repo path** `<repo-path>`: the current directory's checkout if it is the task's repo, else `<checkouts>/<repo>` if it exists, else `gh repo clone <owner>/<repo> <scratchpad>/<repo>`. Use it everywhere a step needs the local repo.
- Load everything: `gh issue view <N> -R <owner>/<repo> --json title,body,state,labels,assignees,comments,closedByPullRequestsReferences`.
- **Find the PRs** with one lookup, and use no other PR search in this skill. Use closing references, timeline cross-references, and a search that matches `<id>` as a whole word, only in the branch name or PR title. Mentions in PR bodies and comments are noise, for example from Dependabot or sibling PRs. Search the org (`org:<org>`) when the task has a tracker key, and only the task's repo (`repo:<owner>/<repo>`) when it has none. Use GraphQL, because `gh search prs --json` has no branch name, and page past the 30-result default:
  ```sh
  prs() { gh api graphql --paginate -f q='<id> <org:… or repo:…> is:pr' -f query='
    query($q: String!, $endCursor: String) { search(query: $q, type: ISSUE, first: 100, after: $endCursor) {
      pageInfo { hasNextPage endCursor }
      nodes { ... on PullRequest { repository { name } number state author { login } headRefName title } } } }' \
    --jq '.data.search.nodes[] | select((.headRefName + " " + .title) | test("\\b<id>\\b"; "i"))'; }
  ```
  An automation's PR may not carry `<id>`: for those, the closing references and cross-references are the source.
- Start the **decision log** and the **follow-up ledger** (§ 7) in `<scratchpad>/drive-task-to-done-<id>.md`, so that they survive a context compaction.

## 2. Route: who does the work

Decide once, then check again on every state change. Log the choice and the reason.

- **An automation owns it**: it is named in `<automations>` and is active on the task (its labels, its comments, its plan). Wait for it (§ 4), and review its plan and PRs as § 3 and § 5 say.
- **Local** in every other case: no automation owns it, the automation is **stuck** (§ 4), or a follow-up is small enough that it is faster to do here.
- **A new task with no activity yet**, when `<automations>` is set: wait one poll window (§ 4) for an automation to pick it up before routing it local, so that two workers don't build the same thing.

**Taking over from an automation:** add its takeover label, if `<automations>` names one, to the task and to its PR. Post one comment that says the local session took over and why. While an automation still owns a task, that takeover label is the only label this skill adds to it. Never remove an automation's gate labels, and never edit its state comments.

### 2a. Mark the task started

Once the task is routed, and again after a takeover, show that work has begun. Do it before any waiting or building, and log it in the decision log. Each step changes nothing when it is already done, so a re-run is safe.

- **Assignee:** if the task is **unassigned** and routed local, or after a takeover, assign it to me (`gh issue edit <N> -R <owner>/<repo> --add-assignee @me`, or the tracker's own assign call for a tracker ticket). Read the assignee back to confirm it, because some tracker APIs accept a malformed value without an error and ignore it. Never reassign a task that belongs to another person or to an automation.
- **Status** (only when `<started>` is set): move the task to it only from the tracker's not-started state (for example "To Do"). Never move it out of any other state here. A tracker transition is chosen by the status it leads **to**, not by its label: list the transitions and pick the one whose target is `<started>`, then read the status back. A task an automation owns keeps the automation's status flow: skip it unless you took over.

## 3. Detect state and dispatch

Compute the state again from live data on every pass. Never trust the log over `gh`. This table is the resume logic.

| State | Signal | Action |
|---|---|---|
| Closed | Issue closed | Check that the close is real: a merged PR that **declares** it closes the task, or a stated conclusion. If any PR of the task is still open, the close came too early: reopen the issue with a one-line comment and continue with § 5. Otherwise § 6–8. |
| Automation working | Its labels show planning or implementing | Wait (§ 4). |
| Plan waits for approval | The automation's approval gate | Review the plan (§ 3a). |
| PR open | Any open PR for the task | § 5 for each PR. |
| Partly merged | Some PRs merged, some open | § 6 for the merged ones, § 5 for the rest. |
| Blocked | A blocked label, or a comment that asks for something | If it waits on a **decision the owner would make**, answer it in a comment as that decision, and log the reason. If it waits on someone else, keep working on everything else and report it. |
| Local, no PR | Routed local, nothing built | § 3b. |

### 3a. Reviewing an automation's plan

Check the plan against the code at `origin/main`, with the same checks as § 3b step 2. If it holds, post an approval comment in the form the automation expects. Every change you require goes into the follow-up ledger as an acceptance criterion for § 5. If it does not hold, post the specific flaws as a comment, and wait for the new plan. If the same flaw survives 2 replans, take over (§ 2). If the plan's gate belongs to another person, proceed, and put that first in the report.

### 3b. Local implementation

1. **Investigate** to a root cause at `origin/main`. Delegate broad searches to an `Explore` subagent.
2. **Plan.** Write a plan and check it yourself: the problem is still present, the fix addresses the root cause and not a symptom, it covers every call site, the evidence is real, and the tests prove the change. The plan also gives what `implement-in-session` § 1 collects: the approach (the owning layer, the flow, any contract change), acceptance criteria with the check that proves each one, whether a user can see the change, what stays unchanged, options rejected and why, and what outside the repo is affected. Put the outside effects in the follow-up ledger (§ 7). Post the plan on the task under `## Local plan (drive-task-to-done)`. Never use a plan header that an automation parses (see `<automations>`): it would take the local plan as its own. After a takeover, the automation's approved plan is the plan: post a local plan only for what changes.
3. **Give the build to `implement-in-session`.** Read its `SKILL.md` and `cli.md`. Its "Autonomous callers" section applies, because this skill is the starting session. Build its brief from the plan, with the approver "the owner, via drive-task-to-done (autonomous)", and add:
   - the autonomy preamble;
   - the branch `fix/<id>-<slug>`; after a takeover with an open PR, "continue `<repo>#<PR>` on its branch and open no new PR";
   - the PR: `<id>` in the title, the attribution line at the end, and in the body `Closes <task url>` if this is the task's only or last PR, else `Refs <task url>` (GitHub closes the issue on the first merge of any `Closes` PR);
   - no reviewer requests, because this skill gets the approval (§ 5b). The repo's own gates still apply;
   - tests: what the repo's CLAUDE.md allows locally; CI runs the rest.

   Start it with the command in `cli.md`, with `<id>` as its `<ticket>` and `<repo-path>` as its repo folder. Log its name, short id and worktree in the ledger.
4. **While it runs**, it is the only writer to its branch: never push, fix or resolve threads on its PR yourself. Answer its questions at once and log each answer. Record its denied permissions as items for the owner. Wait as § 4 says.
5. **When it reports done**, check its result yourself (the diff against the plan, CI, the threads) before you trust it. Copy its decision summary into the decision log. Then go to § 5.

**Fallback**, when the session cannot start or was stopped as stuck (§ 4): build here in a scratchpad worktree (`git -C <repo-path> worktree add <scratchpad>/wt-<id> -b <branch> origin/main`, or the existing PR branch). Follow the same brief inline: grouped review rounds with verifiers, the repo's `ui-review` skill if a user can see the change, the decision summary in the PR, commits of explicit paths, and `gh pr create` as above.

## 4. Waiting (and stall detection)

Wait **in the background only**. Never use a foreground `sleep`, and never poll more often than every 4 minutes. Use the `Monitor` tool (load it with ToolSearch `select:Monitor`), or Bash with `run_in_background: true`:

```sh
fp() { gh issue view <N> -R <repo> --json state,labels,comments \
  -q '[.state,([.labels[].name]|sort|join(",")),(.comments|length)]|@tsv'; }
s=$(fp); end=$(( $(date +%s) + 7200 ))
until [ "$(fp)" != "$s" ] || [ "$(date +%s)" -ge "$end" ]; do sleep 240; done; fp
```

- PR fingerprint: `headRefOid`, the `statusCheckRollup` states, the review count and the comment count.
- To see new PRs, run the § 1 lookup in the loop. A new PR is a `repo#number` that the previous result did not have. Once a PR is known, track it by number, never by search.

**Stuck** means any of these: the automation's takeover label is already on the task, the same failure happened 3 times, the same question was asked twice, or 2 hours passed with no fingerprint change while an automation owns an active phase. When the task is stuck, take over (§ 2) and continue as local. Log the evidence.

**A local `implement-in-session` session** reports through messages that arrive in this conversation, so wait on its PR fingerprint and its messages. It is stuck after 2 hours with no message and no PR change: read `claude logs <id>` first. If it is really stuck, tell it to stop (`SendMessage`), wait for its reply, then use the § 3b fallback on its branch.

While you wait, work on anything else you can act on: another PR of the task, or an item in the ledger.

## 5. PR loop (for each PR, until the exit criteria hold)

Each round, refresh: `gh pr view --json state,isDraft,author,labels,reviewDecision,mergeable,mergeStateStatus,headRefOid`, `gh pr checks --required`, and the unresolved review threads (GraphQL, filter on `isResolved` yourself, as in `address-pr-comments` § 3).

- **Review the current head yourself** with a fresh reviewer subagent that gets the diff, the task and the plan, not your conclusions. Each blocking finding is fixed in this PR.
- **An automation's PR**: post the findings as a **Request changes** review with inline comments. Wait (§ 4) for a new head, then review again, and give each earlier thread a verdict. If the same blocking thread is still not fixed after 3 rounds, take over.
- **A local PR** (mine, or after a takeover):
  - **Its `implement-in-session` session still exists** (`ListAgents`) → it does the fixing. Each round, send it the findings, the reviewers' comments and red CI with `SendMessage`. Wait for its report, then check the new head.
  - **No session** → first make sure a scratchpad worktree exists on the PR branch (`git -C <repo-path> fetch origin <branch> && git -C <repo-path> worktree add <scratchpad>/wt-<id> -B <branch> origin/<branch>`, or reuse the § 3b one). Then run `address-pr-comments <N> in <repo>` with the preamble, plus "work in the worktree `<scratchpad>/wt-<id>`: skip your repo-path resolution and the `gh pr checkout`, and never use a checkout under `<checkouts>`". Fix red CI in the same worktree.
- **Bot reviewers** count like human ones: fix every real finding, including Lows, in one batch per round. Reply "not a defect" with the reason only when a finding is wrong. Stop the comment loop when no new comments arrive, or after 5 rounds, and report what is left.
- **Required changes** from the plan review or a code review are done **in this PR**, never deferred to a ticket. The only exception is a comment completely unrelated to the PR's work: file a follow-up in `<follow-up-repo>`, reply with its link, resolve the thread, and log it in the ledger. More work or a bigger diff is never a reason.
- **A change a user can see** needs the repo's `ui-review` (if it has one) before the merge, plus anything the repo's CLAUDE.md or the extra rules require. Check that the PR description records it.

**Exit criteria:** no unresolved threads, every required check green, `mergeable == MERGEABLE`, the review requirement met, and your review covered the **current** head. If the head moved since your last review, review again.

### 5b. Approval for a PR I authored

I cannot approve my own PR. Ask `<approver>`, then keep monitoring until the exit criteria hold.

1. **Ask only when the PR is ready:** not a draft, required checks green, your review left nothing blocking, and no unresolved threads.
2. **Send one short, self-contained request** in the way `<approver>` names: the PR URL, the task, a summary in one or two lines, the plan link, and "CI green at `<short sha>`; leave blocking issues as review comments on the PR". Log the link to the request.
3. **Monitor** as § 4 says, with the latest review on the PR, and the reply thread too if the request went to a chat.
   - **Approved** on the current head → continue to the exit criteria, then § 6.
   - **Changes requested** → fix them as for any local PR, wait for green, then ask again in the same thread. A push that dismisses an approval gets the same request.
   - **No response in 2 hours** → send one reminder in the same thread. After 2 more hours with nothing, the PR **waits on the approver**: drive everything else, and report it with the link.
4. **No `<approver>`, or it refuses** → report the PR as waiting on a human approver, with the reason. Never pick a human reviewer yourself.

## 6. Merge (and stop there)

1. **Read the final diff yourself** against the plan's load-bearing claims. Green CI with an approval is not a review. **A PR this run authored merges only with an approval from someone other than its author on the current head**, even when the repo's branch protection would allow less. Without one, don't merge: list the PR under "Needs you". Never merge in an archived repo (`gh api repos/<repo> --jq .archived`), or in a repo the extra rules protect.
2. **Order:** merge in the required order (the reader before the writer, the schema before its consumer).
3. **Stacked children first.** Deleting a branch closes every open PR based on it, and a closed PR cannot be retargeted. List them with `gh pr list -R <repo> --base <branch> --state open --json number`, retarget each with `gh pr edit <child> -R <repo> --base main`, and confirm the change before you merge the parent.
4. **Merge** with the method of the repo's recent merges: `gh pr merge <N> -R <repo> --<merge|squash> --delete-branch`. Confirm that the branch is gone (`git ls-remote --heads origin <branch>`), and delete it if it is not. **Never `git reset` after a merge.**
5. **Do not release or deploy.** Never publish a release and never push a tag. Write down whether the merged-but-unreleased state is **safe to sit in**, and why. Follow the extra rules for release requests.
6. **The tracker after the merge:** read the status again. A merge is not a deploy: move the ticket to done only if that is the tracker's convention for merged work. A parent stays open while any child is open.

## 7. Follow-ups: do them or file them, never drop them

The ledger holds every follow-up the run creates or finds: required changes from the plan review, deferred plan items, out-of-scope review suggestions, sibling tasks in the same code, open children, verification after the release, and the cleanup of worktrees and background jobs.

- **In scope and doable now** → do it, through § 3b or the current PR.
- **Otherwise file it** in `<follow-up-repo>`, never in a code repo it does not belong to. Search first (`gh search issues`), so that you do not file a duplicate. Link it to the task.
- **Verification after the release** → file it as a follow-up that names the exact check, and list it in the report.

## 8. Completion gate

Clean up first: remove the scratchpad worktrees (`git worktree remove`), and stop or collect every background poll. After the merge, run `claude rm <session id>` for each `implement-in-session` session this run started (the preamble pre-grants it). Check `git -C <repo-path> worktree list` and remove any worktree left behind. Then run `is-done`.

- **"Still mine"** rows → do them now (the preamble pre-grants `is-done`'s "ask first"), then run `is-done` again.
- **Waiting on time or automations** → go back to § 4 and keep driving.
- Stop only when the verdict is ✅, or 🟡 with every remaining row truly for a human: a release, a required human reviewer, verification on a platform you cannot drive, or a decision outside the owner's delegation. If everything left waits on humans and nothing changed for 2 hours, stop and report instead of polling forever. A re-run resumes from § 3.

## 9. Final report

1. **Verdict line**, for example `✅ owner/repo#123 done — merged, unreleased (safe to sit: …)` or `🟡 … — 2 items need you`.
2. **Decision log** table, most important first. Columns: decision · reason (one line) · who decided (this skill, a local session, an automation, a reviewer) · link. Include the local sessions' decision summaries, routing, takeovers, plan verdicts, the thread resolutions (counts, with a link per PR), merges, follow-ups filed, and anything skipped. Put any approval of another person's gate first.
3. **Done**, with evidence (a URL or a sha).
4. **Needs you**: only items no agent can do, each with its exact next step.
5. **Links:** task, plan, approval, PRs, follow-ups.

Mark every inference as an inference.

## Rules

- **No questions mid-flight.** The preamble pre-grants every "ask" in a sub-skill. Decide, log, continue.
- **Merge** only after § 6.1, then confirm that the branch was deleted. A PR this run authored never merges without a non-author approval on its current head. **Never release or deploy.**
- **Tests:** follow the repo's CLAUDE.md for what runs locally; CI runs the rest.
- **Git:** never `git add -A`; stage explicit paths. Never switch branches in a checkout under `<checkouts>`; use scratchpad worktrees, `git show origin/main:<path>`, or `pr/<N>` refs. Never force-push a shared branch.
- **Production is read-only:** no live payment keys, no writes to production databases or third-party production systems.
- **Automation etiquette:** never remove an automation's gate labels, never edit its state comments, never change its phase labels. Approval is a comment.
- **Primary sources:** check each load-bearing claim against `origin/main` or live telemetry before you act on it.
- **Never invent a status.** A background job with no completion notification is still running. Report anything unverified as unverified.
- Delegate heavy reads and fixes to subagents to save context, and always give them the preamble.
