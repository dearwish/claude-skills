---
name: address-pr-comments
description: Address PR review comments — fix valid issues, reply to and resolve all threads, then verify the PR title and description are up to date.
allowed-tools: Bash(gh *), Bash(git *), Read, Edit, Grep, Glob, Agent
context: fork
argument-hint: [<pr-number>] in <repo-name> (e.g. 251 in my-api, or just "in my-api" to use the current branch's PR)
---

# Address PR Review Comments

**Org context:** read `~/.claude/private/context.md` first. It defines `<org>` (GitHub org), `<follow-up-repo>` (where follow-up issues go) and `<KEY>` (issue-tracker key prefix), plus any extra org rules. If the file is missing, ask me for these values.

Review and address all unresolved PR comments, then ensure the PR title and description are current.

## Steps

### 1. Parse arguments

The argument format is `[<pr-number>] in <repo-name>`. The PR number is optional. The full GitHub slug is `<org>/<repo-name>`.

Resolve the local repo path dynamically from the repo name:

1. Set `repo_slug=<org>/<repo-name>`.
2. If the current working directory is already inside a git checkout, check whether it matches the target repository:
   ```sh
   current_root="$(git rev-parse --show-toplevel 2>/dev/null)" && \
   current_slug="$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)"
   ```
3. If `current_slug` equals `repo_slug`, use `current_root` as the repo directory.
4. Otherwise, check for a local checkout at `~/work/<repo-name>` and use it if present.
5. Otherwise, clone the repository into the current working directory and then resolve its top-level path:
   ```sh
   gh repo clone "$repo_slug"
   repo_dir="$(git -C "<repo-name>" rev-parse --show-toplevel)"
   ```

`cd` into the resolved repo directory for all subsequent operations.

**If no PR number is provided**, find the PR associated with the current branch:
```sh
gh pr view --repo <org>/<repo-name> --json number --jq '.number'
```
This uses the current branch to look up the open PR. If no PR is found, report the error and stop.

### 2. Update the branch with latest main

Check out the PR branch and bring it up to date before doing anything else:

```sh
gh pr checkout <pr-number> --repo <org>/<repo-name>
git pull                    # pick up commits pushed to the PR branch by others
git fetch origin main
```

If `origin/main` is not already contained in the branch (`git merge-base --is-ancestor origin/main HEAD` fails), merge it:

```sh
git merge origin/main
```

- **If the merge is clean**, push it.
- **If there are conflicts**, resolve each one by combining the intent of both sides — keep the PR's changes AND the changes from main; never blindly take one side. Read the surrounding code to understand what each side was doing. After resolving, verify the build passes, then `git commit` (keep the default merge message) and push.
- **Never rewrite the merge afterwards** (no `git reset` after merging) — dropping the merge parent silently discards main's history and causes phantom PR conflicts.

### 3. Fetch unresolved review threads

Use the GitHub GraphQL API to get all unresolved review threads. **Paginate** to ensure no threads are missed — PRs with many review rounds can exceed 50 threads.

Fetch the first page (fetch all comments per thread so categorization uses the full discussion, not just the first message):
```sh
gh api graphql -f query='{ repository(owner: "<org>", name: "<repo-name>") { pullRequest(number: <pr-number>) { reviewThreads(first: 100) { pageInfo { hasNextPage endCursor } nodes { id isResolved isOutdated comments(first: 100) { nodes { databaseId body path line author { login } } } } } } } }' --jq '.data.repository.pullRequest.reviewThreads'
```

Note: `reviewThreads` does **not** accept a `filterBy` argument — passing one fails with `Field 'reviewThreads' doesn't accept argument 'filterBy'`. Resolved threads must be filtered client-side on `isResolved`.

If `pageInfo.hasNextPage` is true, fetch subsequent pages using the `after` cursor:
```sh
gh api graphql -f query='{ repository(owner: "<org>", name: "<repo-name>") { pullRequest(number: <pr-number>) { reviewThreads(first: 100, after: "<endCursor>") { pageInfo { hasNextPage endCursor } nodes { id isResolved isOutdated comments(first: 100) { nodes { databaseId body path line author { login } } } } } } } }' --jq '.data.repository.pullRequest.reviewThreads'
```

Continue until `hasNextPage` is false. Collect all nodes across pages, then keep only those with `isResolved == false`. Extract the relevant fields from each thread — use the first comment for file/line context and the last comment for the most recent request:
```
{threadId: .id, commentId: .comments.nodes[0].databaseId, path: .comments.nodes[0].path, line: .comments.nodes[0].line, body: .comments.nodes[0].body, lastBody: .comments.nodes[-1].body}
```

If there are no unresolved threads, skip to step 8.

### 4. Fetch the original issue/plan for scope context

Before categorizing comments, determine what this PR is supposed to do by looking up its source task:

1. **Find the linked issue/ticket** — parse the PR body for a GitHub issue link (`https://github.com/<org>/.../issues/<N>` or `#<N>`) or a Jira ticket key (`<KEY>-<N>`). If the PR was opened by an automation, the link is usually in the PR body.
2. **Fetch the issue/ticket** — read the GitHub issue body and comments (`gh issue view <N> -R <org>/<repo-name> --json body,comments`) or the Jira ticket description, acceptance criteria, and comments.
3. **Fetch the approved plan** — scan the issue body and comments, or the Jira ticket comments, for a `## Fix Plan` or `## Implementation Plan` header. This is the approved scope of work.

If no linked issue/plan is found (e.g., a standalone PR), skip the scope check in step 5 — treat all comments as if they are in scope.

Keep the issue body and plan text as the **scope reference** for step 5.

### 5. Categorize each comment

For each unresolved comment, read the referenced file and line to understand the context. If the thread's last comment is your own older split proposal ("I'd like to handle this in a separate follow-up…"), read the reviewer's replies after it: if they agreed, file the follow-up as for **Unrelated**; otherwise categorize it fresh below. Categorize as:

- **Actionable** — A valid request: a bug, missing error handling, incorrect logic, a style violation per project conventions, or a change the reviewer asks for. **The default is to do it in this PR**, including when it touches code the PR did not change or goes beyond the approved plan. "Not in the plan" and "my diff didn't touch that line" are not reasons to defer: the plan bounds what you do unprompted, and a review comment is a fresh instruction from the person whose approval the PR waits on. Fix every real finding, Lows included.
- **Not applicable** — The comment refers to code or files that are not part of this PR (e.g., gitignored files), or a later commit already addressed it.
- **Not a defect** — The comment is wrong: it does not reproduce, or it describes the intended behaviour. Check the reviewer's claim against the code before you put a comment here. "The fix is more work than it's worth" is not this category, and not a reason for any other.
- **Unrelated** — The only exception to "do it in this PR": the comment is completely unrelated to this PR's work. It is about another service or feature, and no path leads from this diff to it. More work, a bigger diff, a refactor the fix pulls in, tidiness, or "not in the plan" never make a comment unrelated: those are **Actionable**. When in doubt, it is **Actionable**.

### 6. Fix actionable comments

For each actionable comment:
1. Read the file and understand the issue
2. Apply the fix using the Edit tool
3. Stage the changed file(s)

After all fixes are applied:
1. Verify the build passes
2. Commit all fixes in a single commit with message: `fix: address PR review comments`
3. Push to the branch

### 7. Reply and resolve all threads

**CRITICAL: You MUST post a reply before resolving any thread. Never call the resolve mutation without first confirming the reply API call succeeded. A resolved thread with no reply is a bug.**

For each comment, reply with a concise message explaining what was done, then resolve the thread:
- **Fixed** — `"Fixed — <brief description of what changed>."`
- **Not applicable** — `"<explanation of why this doesn't apply>."`
- **Not a defect** — `"<why the code is right: what you checked and what it showed>."`
- **Unrelated** — file a follow-up, then reply `"Unrelated to this PR (<what it is about instead>). Filed as <follow_up_url>."`. Do not wait for the reviewer to agree. GitHub issues go to `<follow-up-repo>`, never to the code repo itself. Use a temp file for the body so multi-line context, links, quotes, and code identifiers are preserved safely:
  ```sh
  issue_body_file="$(mktemp)"
  cat <<'EOF' > "$issue_body_file"
  <description with link to the review comment>
  EOF
  follow_up_url="$(gh issue create -R <follow-up-repo> --title "<title>" --body-file "$issue_body_file")"
  rm -f "$issue_body_file"
  ```
  One follow-up per unrelated comment; never fan one comment out into several issues.

Reply using the REST API (write the body to a temp file to avoid shell quoting issues with quotes, newlines, or code identifiers):
```sh
reply_file="$(mktemp)"
cat <<'EOF' > "$reply_file"
<reply>
EOF
REPLY_RESULT=$(gh api repos/<org>/<repo-name>/pulls/<pr-number>/comments \
  -F body=@"$reply_file" \
  -F in_reply_to=<commentId>)
rm -f "$reply_file"
# Only proceed to resolve if the reply was posted successfully
echo "$REPLY_RESULT" | grep -q '"id"' || { echo "Reply failed — skipping resolve for thread <threadId>"; continue; }
```

Then resolve the thread using GraphQL:
```sh
gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "<threadId>"}) { thread { isResolved } } }'
```

Batch multiple resolve mutations into a single GraphQL call when possible:
```sh
gh api graphql -f query='mutation { a: resolveReviewThread(input: {threadId: "<id1>"}) { thread { isResolved } } b: resolveReviewThread(input: {threadId: "<id2>"}) { thread { isResolved } } }'
```

### 8. Verify PR title and description (only if changes were pushed)

**Skip this step** if no actionable comments were fixed in steps 5–7 (i.e., the PR content did not change this cycle). The title and description are already up to date from the last time they were verified.

If changes were pushed, fetch the current PR title and body:
```sh
gh pr view <pr-number> --repo <org>/<repo-name> --json title,body
```

Also fetch the full diff to understand what the PR actually does:
```sh
gh pr diff <pr-number> --repo <org>/<repo-name> | head -200
```

Check if the title and description accurately reflect the current state of the PR. If they are stale or inaccurate, update them:
```sh
gh pr edit <pr-number> --repo <org>/<repo-name> --title "<new-title>" --body "<new-body>"
```

### 9. Report back

Summarize what was done:
- Whether main was merged in and whether conflicts were resolved
- Number of comments addressed (fixed / not applicable / not a defect / unrelated)
- Follow-up issues filed for unrelated comments (with links)
- Whether the PR description was updated
- Link to the PR
