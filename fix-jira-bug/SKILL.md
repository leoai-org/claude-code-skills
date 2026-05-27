---
name: fix-jira-bug
description: End-to-end workflow for picking up an unassigned Jira bug,
  investigating the root cause in the codebase, implementing a fix,
  opening PRs, iterating on reviews, and assigning to the right reviewer.
---

# Fix Jira Bug — End-to-End

Pick up an unassigned bug from Jira, investigate the root cause, implement a fix, open PRs, iterate on reviews until green, and assign to the best reviewer.

Can be started with:
- A specific Jira issue key: `/fix-jira-bug SW-2883`
- No arguments (will search for the most urgent unassigned bug in the current sprint): `/fix-jira-bug`

## Phase 1: Find the Bug

### If a Jira issue key is provided:
1. **Fetch the issue** using `getJiraIssue` with the provided key.

### If no issue key is provided:
1. **Get Jira cloud ID** using `getAccessibleAtlassianResources`.
2. **Search for open bugs in the current sprint** using JQL:
   ```
   type = Bug AND sprint in openSprints() AND status != Done ORDER BY priority ASC
   ```
   if no open bugs in spring, take from backlog.
3. **Parse results** and filter for bugs that are:
   - **Unassigned** (`assignee` is null)
   - **Not started** (status is `To Do`)
4. **Pick the most urgent one** — highest priority first, then oldest.
5. **Fetch full issue details** using `getJiraIssue`.

### Present the chosen bug:
Show the user: issue key, priority, summary, and description. Confirm before proceeding.

## Phase 2: Investigate the Root Cause

1. **Understand the bug** — Read the Jira description, linked Sentry errors, and any comments.
2. **Search the codebase** for related code:
   - Search for error messages, keywords, and related function names using `Grep` and `Glob`.
   - Use `Task` with `Explore` subagent for broad searches across multiple repos.
3. **Trace the error path** — Follow the code from where the error is thrown to where it should be caught.
4. **Identify the root cause** — Determine exactly what's wrong and why.
5. **Present findings** to the user with a clear explanation of:
   - What's happening
   - Why it's happening
   - What the fix should be

## Phase 3: Set Up Isolated Worktrees

**CRITICAL**: Never modify the main working directories directly — other developers or agents may be using them. Always use `git worktree`.

For **each repo** that needs changes:

1. **Create a worktree** in an isolated directory:
   ```bash
   cd <main-repo-path>
   git fetch origin main
   git worktree add <worktree-path> -b fix/<branch-name> origin/main
   ```
   Place worktrees under a shared workspace directory (e.g., the current project dir).

2. **Symlink `node_modules`** from the main repo to avoid reinstalling:
   ```bash
   ln -s <main-repo-path>/node_modules <worktree-path>/node_modules
   ```

3. **Verify the worktree is clean** and on the correct base branch.

## Phase 4: Implement the Fix

1. **Make the code changes** in the worktree(s).
2. **Add tests** if applicable — unit tests covering the new behavior.
3. **Verify compilation**:
   ```bash
   cd <worktree-path> && npx tsc --noEmit --pretty
   ```
   If there are pre-existing errors not related to your changes, verify they exist on the unmodified branch too.
4. **Run relevant tests**:
   ```bash
   cd <worktree-path> && npx jest <test-file> --no-cache
   ```

## Phase 5: Commit, Push, and Open PRs

For each repo with changes:

1. **Review the diff**:
   ```bash
   git -C <worktree-path> diff
   ```
2. **Commit** with a descriptive message referencing the Jira issue:
   ```bash
   git -C <worktree-path> add <specific-files>
   git -C <worktree-path> commit -m "Fix: <description>

   <details>

   Fixes <JIRA-KEY>

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
   ```
3. **Push** the branch:
   ```bash
   git -C <worktree-path> push -u origin <branch-name>
   ```
4. **Create a PR** using `gh`:
   ```bash
   cd <worktree-path> && gh pr create --title "<title> (<JIRA-KEY>)" --body "$(cat <<'EOF'
   ## Summary
   - <bullet points explaining what and why>

   ## Test plan
   - [ ] <test checklist>

   Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

## Phase 6: Review & Fix Loop

For each PR, iterate until green:

### 6a. Wait for CI and reviews
```bash
sleep 900  # 15 minutes
```

### 6b. Check CI status
```bash
gh pr view <pr-number> --json statusCheckRollup --jq '.statusCheckRollup[] | "\(.name): \(.conclusion // .status)"'
```
- If any check **failed**, read the logs, fix the issue, commit, and push.

### 6c. Check for review comments
```bash
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments --jq '[.[] | {author: .user.login, body: .body[:200], id: .id}]'
```

### 6d. Address each comment
- **Actionable feedback** — Fix it in code, commit, push, then reply and resolve the comment thread.
- **Questions or nits** — Reply explaining your reasoning, then resolve.
- **Reply and resolve** each addressed comment:
  ```bash
  # Reply to the comment
  gh api repos/<owner>/<repo>/pulls/comments/<comment-id>/replies -f body="<response>"
  ```
  Then **resolve the thread** so reviewers can see it's been handled. Use the GraphQL API to resolve review threads:
  ```bash
  # First, find the thread ID for the comment
  gh api graphql -f query='
    query($owner: String!, $repo: String!, $pr: Int!) {
      repository(owner: $owner, name: $repo) {
        pullRequest(number: $pr) {
          reviewThreads(first: 100) {
            nodes {
              id
              isResolved
              comments(first: 1) {
                nodes { body }
              }
            }
          }
        }
      }
    }' -f owner=<owner> -f repo=<repo> -F pr=<pr-number>
  # Then resolve each addressed thread
  gh api graphql -f query='
    mutation($threadId: ID!) {
      resolveReviewThread(input: {threadId: $threadId}) {
        thread { isResolved }
      }
    }' -f threadId=<thread-id>
  ```

### 6e. Repeat
Go back to step 6a. **Maximum 5 rounds** to prevent infinite loops. After 5 rounds, report remaining issues and stop.

### Exit criteria
- All CI checks pass (green)
- No unresolved review comments
- No new comments after the latest push

## Phase 6.5: Test Gate — leo-agents Only

The `leo-agents` repo does **not** run tests automatically on PR push. Tests are triggered by adding a label.

After the review loop (Phase 6) completes for a `leo-agents` PR:

### 6.5a. Add the test label
```bash
gh pr edit <pr-number> --add-label "run-tests"
```

### 6.5b. Wait for tests to run
```bash
sleep 900  # 15 minutes
```

### 6.5c. Check test results
```bash
gh pr view <pr-number> --json statusCheckRollup --jq '.statusCheckRollup[] | "\(.name): \(.conclusion // .status)"'
```

### 6.5d. If tests failed:
1. Read the failing test logs to understand what broke.
2. **Remove the label** so tests don't re-trigger prematurely:
   ```bash
   gh pr edit <pr-number> --remove-label "run-tests"
   ```
3. Fix the issue in code, commit, and push.
4. **Go back to Phase 6** (the review & fix loop) — the new push may trigger new review comments that need addressing.
5. After Phase 6 completes again, return here to step 6.5a and re-add the label.

### 6.5e. If tests passed:
Proceed to Phase 7 (Assign Reviewer). The PR is green.

## Phase 7: Assign the Best Reviewer

1. **Find the right person** using git blame on the files you changed:
   ```bash
   git -C <worktree-path> log --format='%an <%ae>' -- <changed-files> | sort | uniq -c | sort -rn | head -5
   ```
2. **Cross-reference with repo collaborators**:
   ```bash
   gh api repos/<owner>/<repo>/collaborators --jq '.[].login'
   ```
3. **Assign the top contributor** as reviewer:
   ```bash
   gh pr edit <pr-number> --add-reviewer <github-username>
   ```

## Phase 8: Final Summary

Present a summary table:

| Item | Details |
|------|---------|
| **Jira Issue** | KEY - Summary |
| **Root Cause** | Brief explanation |
| **Fix** | What was changed and why |
| **PRs** | Links to all PRs with CI status |
| **Reviewer** | Who was assigned and why |

## Important Rules

- **Always use `git worktree`** — never modify main working directories.
- **Never force-push.** Always use regular `git push`.
- **Never commit secrets** (`.env`, credentials, tokens).
- **Always run tests** before pushing.
- **Use `git add <specific-files>`** instead of `git add -A`.
- **Maximum 5 review rounds** per PR.
- **Reference the Jira issue key** in commit messages and PR titles.
- **Symlink `node_modules`** into worktrees instead of reinstalling.
- If a fix spans multiple repos, **create companion PRs** and cross-reference them.
