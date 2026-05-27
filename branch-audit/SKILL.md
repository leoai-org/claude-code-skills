---
name: branch-audit
description: >
  Daily branch audit that SSHes into remote dev machines (trex, ada, spark, scaleway),
  scans git branches and recent commits in the LeoCAD repo, cross-references progress
  against open Jira tasks in the LEO project, and updates Jira with progress summaries
  and status transitions. Use this skill whenever the user asks for a daily standup,
  branch check, progress audit, code-vs-Jira sync, or wants to know what's been
  implemented and what's missing across their machines. Also trigger when the user
  mentions "check my branches", "what have I done this week", "update Jira from git",
  "sync branches with Jira", or any variation of auditing development progress across
  multiple machines.
---

# Branch Audit Skill

Audit git branches across multiple remote machines, compare progress against Jira tasks, and update Jira accordingly.

## Overview

Dimi develops on the same repo (LeoCAD) across four remote machines. Branches and Jira tasks are linked loosely — there's no strict naming convention. This skill bridges that gap by:

1. SSHing into each machine and collecting branch + commit data from the last 7 days
2. Pulling all open (non-Done) Jira tasks assigned to Dimi from the LEO project
3. Using the commit messages, changed files, and branch names to infer which commits relate to which Jira tasks
4. Generating a progress report: what's been implemented, what's still missing (relative to each task's description and acceptance criteria)
5. Updating Jira: adding a progress comment to each relevant task, and transitioning status where appropriate

## Machine Configuration

| Machine   | SSH Host   | Repo Path                                      |
|-----------|------------|-------------------------------------------------|
| trex      | `trex`     | `/home/trex/PycharmProjects/Dimion/LeoCAD`     |
| ada       | `ada`      | `/home/dimion/projects/LeoCAD`                  |
| spark     | `spark`    | `/home/dimion/LeoCAD`                           |
| scaleway  | `scaleway` | `/home/dimion/LeoCAD`                           |

## Jira Configuration

- **Cloud ID**: `856b109d-cf57-4873-a390-a5d3b6308c00`
- **Project**: LEO
- **Account ID**: `712020:28347a35-464b-4138-be67-84623227f716`
- **JQL filter**: `project = LEO AND assignee = currentUser() AND statusCategory != Done ORDER BY status ASC`
- **Statuses**: To Do → In Progress → PR → Done

## Step-by-Step Workflow

### Step 1: Collect Git Data from All Machines

SSH into each machine and run git commands to gather:

```bash
# For each machine, run via SSH:
ssh <host> "cd <repo_path> && git fetch --all 2>/dev/null; \
  echo '=== BRANCHES ==='; \
  git branch -a --sort=-committerdate | head -30; \
  echo '=== RECENT COMMITS (7 days) ==='; \
  git log --all --oneline --since='7 days ago' --author-date-order --format='%H|%an|%ad|%s' --date=short; \
  echo '=== CHANGED FILES (7 days) ==='; \
  git log --all --since='7 days ago' --name-only --format='COMMIT:%H' "
```

Run all four SSH commands and collect the output. If a machine is unreachable, note it and continue with the others — don't let one failure block the whole audit.

Keep the raw output organized by machine for the analysis step.

### Step 2: Pull Open Jira Tasks

Use the Jira MCP tools to fetch all open tasks:

- Use `searchJiraIssuesUsingJql` with the JQL filter above
- Request fields: `summary`, `description`, `status`, `issuetype`, `priority`
- For tasks that have acceptance criteria in their description, extract those specifically — they're the benchmark for "what's missing"

### Step 3: Match Commits to Jira Tasks

This is the inference step. Since branch names don't follow a strict `LEO-XXX` convention, use multiple signals to match commits to tasks:

1. **Direct references**: Look for Jira keys (LEO-XXX) in branch names or commit messages — these are strong matches
2. **Keyword overlap**: Compare commit messages and changed file paths against task summaries and descriptions. Look for overlapping terminology (e.g., a commit mentioning "diffusion" likely relates to diffusion-related tasks, "VAE" commits relate to VAE tasks, "crop" relates to crop fix tasks)
3. **File path signals**: If changed files are in directories that clearly map to a feature area (e.g., `vae/`, `diffusion/`, `step_analyzer/`), use that as a hint
4. **Branch name semantics**: Even without Jira keys, branch names like `feature/vae-normals` or `fix/crop-threshold` carry intent

Build a mapping: `{task_key: [list of relevant commits with machine info]}`. Some commits may map to multiple tasks, and some tasks may have no matching commits — both are fine.

### Step 4: Analyze Progress vs. Acceptance Criteria

For each Jira task, assess:

- **What's been implemented**: Summarize the relevant commits — what do they actually do? Reference specific files changed and commit messages.
- **What's missing**: Compare against the task description and acceptance criteria. If the task says "Add batching to the generative model" and commits only show initial scaffolding, note that the core batching logic is still missing.
- **Which machines have the work**: Note where the relevant branches/commits live. If work is only on one machine and not pushed, flag that.
- **Branch drift**: If the same branch exists on multiple machines but at different commits, flag the divergence.

### Step 5: Generate the Report

Present a summary to the user organized by task, showing:

- Task key + summary + current status
- Relevant branches and which machines they're on
- Commits from the last 7 days that relate to this task
- Progress assessment: what's done vs. what's still needed
- Recommended Jira action (update status, add comment, no change)

Also include an "unmatched commits" section for any recent work that doesn't clearly map to a Jira task — this might indicate work that needs a new task created.

### Step 6: Update Jira (with user confirmation)

Before making any changes, present the planned updates and ask for confirmation. The updates are:

**Comments**: For each task with matching commits, add a structured comment like:

```
🔄 Branch Audit — [date]

**Recent activity (last 7 days):**
- [machine]: [branch] — [commit summary]
- [machine]: [branch] — [commit summary]

**Progress**: [brief assessment]
**Still needed**: [what's missing per acceptance criteria]
```

**Status transitions**: Suggest transitions based on the evidence:
- **To Do → In Progress**: If there are recent commits clearly working on this task
- **In Progress → PR**: Only if there's evidence of a PR or the branch appears complete relative to the acceptance criteria
- Don't transition to Done — that should always be manual

Ask the user to confirm which updates to apply. Apply them using the Jira MCP tools:
- `addCommentToJiraIssue` for comments (load this tool via tool_search first)
- `transitionJiraIssue` for status changes (use `getTransitionsForJiraIssue` first to get valid transition IDs)

## Edge Cases

- **Machine unreachable**: Log it, skip it, mention it in the report. Don't fail the whole audit.
- **No commits in 7 days**: Report "no recent activity" for that machine. Still useful to know.
- **Task has no description**: Note that the task lacks acceptance criteria, making it hard to assess what's missing. Suggest the user flesh out the description.
- **Commits on detached HEAD or stash**: If you can capture stash info, include it as a bonus. Not critical.
- **Very large number of commits**: Summarize by theme rather than listing every commit. Focus on the most recent and most relevant.

## Important Notes

- The SSH hosts (trex, ada, spark, scaleway) are configured in `~/.ssh/config`. If SSH keys aren't set up in the current environment, the skill should guide the user to set them up or offer to use a different approach (like a script the user runs manually).
- Always `git fetch --all` before collecting data to ensure you're seeing the latest remote state.
- The matching step is fuzzy by design — it's better to over-match (and let the user correct) than to miss connections. Present matches with confidence indicators if possible.
- Keep Jira comments concise and scannable. Engineers skim these.
