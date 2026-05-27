---
name: pr-reviewer
description: >
  Automated PR review bot powered by Claude. Deploys as a GitHub Action that triggers
  on every PR open and push-to-PR event, reviews the diff for bugs, logic errors,
  security issues, and code style problems, then posts inline review comments and
  edit suggestions directly on the PR. Use this skill whenever the user wants to set up
  automated code review, PR review bots, GitHub Action review pipelines, or mentions
  "review my PRs", "automated code review", "PR comments", "review bot", or any
  variation of AI-powered pull request review and feedback. Also trigger when the user
  wants to customize review rules, adjust review strictness, or debug their PR review
  pipeline.
---

# PR Reviewer — Automated Claude-Powered Code Review

A GitHub Action that automatically reviews pull requests using the Anthropic API. On every PR open or new commit push, it fetches the diff, sends it to Claude for analysis, and posts a single PR review with inline comments.

## What It Does

When a PR is created or updated, the action:

1. Fetches the PR diff via the GitHub API
2. Parses the diff into per-file hunks with precise line number tracking
3. Sends each changed file to Claude for review, focusing on:
   - **Bugs & logic errors** — off-by-ones, null derefs, race conditions, incorrect algorithms
   - **Security issues** — injection, hardcoded secrets, unsafe deserialization, path traversal
   - **Code style & best practices** — naming, complexity, dead code, missing error handling
   - **Large files** — flags files with many added lines and suggests how to split them into focused modules
   - **Large functions/methods** — identifies functions that are too long and recommends decomposition
   - **Code duplication** — detects repeated code blocks across changed files and suggests shared helpers
4. Runs a cross-file duplication detector to catch copy-pasted code blocks
5. Posts a single PR review with inline comments at the exact lines where issues were found
6. Uses GitHub's "suggestion" syntax so reviewers can accept fixes with one click

### Line Numbers in Every Comment

Every review comment includes the exact line number (or line range) where the issue occurs, both in the comment header and in the description. This makes it easy to jump to the right location, even in large diffs. Multi-line issues use GitHub's start_line/line range API for precise highlighting.

### Large File / Large Function Detection

The reviewer checks if a file or function exceeds configurable thresholds:
- **Large files** (`LARGE_FILE_LINE_THRESHOLD`, default 400 lines): suggests splitting into smaller, focused modules with specific groupings of related functionality
- **Large functions** (`LARGE_FUNCTION_LINE_THRESHOLD`, default 60 lines): recommends decomposition into sub-functions, describing what each one would do

### Code Duplication Detection

Two layers of duplication detection:

1. **Claude-powered** — Claude identifies similar patterns and repeated logic within each file's diff
2. **Cross-file detector** — a separate analysis step compares added code blocks across all changed files, catching copy-paste duplication that Claude might miss since it reviews one file at a time. Configurable via `DUPLICATION_MIN_LINES` (default 6).

## Architecture

```
.github/workflows/pr-review.yml    ← GitHub Action workflow (trigger config)
.github/scripts/review_pr.py       ← Core review logic (diff parsing, Claude API, GitHub API)
```

## Setup Instructions

### Step 1: Add the Anthropic API Key as a Repository Secret

1. Go to your repo → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Name: `ANTHROPIC_API_KEY`
4. Value: your Anthropic API key
5. Click "Add secret"

### Step 2: Copy the Workflow and Script into Your Repo

Copy the two files from this skill into your repository:

```bash
# From the skill directory:
mkdir -p .github/scripts
cp scripts/review_pr.py .github/scripts/review_pr.py
cp scripts/pr-review.yml .github/workflows/pr-review.yml
```

Or create them manually — the files are in the `scripts/` directory of this skill.

### Step 3: Commit and Push

```bash
git add .github/
git commit -m "Add automated PR review via Claude"
git push
```

That's it. The next PR opened (or pushed to) will get an automated review.

## Configuration

### Environment Variables (set in the workflow YAML)

| Variable | Default | Description |
|----------|---------|-------------|
| `REVIEW_MODEL` | `claude-sonnet-4-20250514` | Which Claude model to use |
| `MAX_FILES` | `30` | Max files to review per PR (skip if more, to avoid huge PRs) |
| `MAX_DIFF_SIZE` | `800` | Max lines per file diff before truncation |
| `SEVERITY_THRESHOLD` | `info` | Minimum severity to post: `info`, `warning`, `error` |
| `REVIEW_LANGUAGE` | `english` | Language for review comments |
| `LARGE_FILE_LINE_THRESHOLD` | `400` | Suggest splitting files with more added lines than this |
| `LARGE_FUNCTION_LINE_THRESHOLD` | `60` | Suggest splitting functions longer than this |
| `DUPLICATION_MIN_LINES` | `6` | Minimum contiguous added lines to flag as potential duplication |

### Customizing Review Rules

Edit the `SYSTEM_PROMPT` in `review_pr.py` to add project-specific rules. For example:

```python
# Add to the SYSTEM_PROMPT:
PROJECT_RULES = """
Additional rules for this project:
- We use CadQuery for all CAD operations — flag any raw OpenCascade calls
- All public functions must have docstrings
- No print() in production code — use logging module
- Prefer pathlib over os.path
"""
```

### Adjusting Strictness

The `SEVERITY_THRESHOLD` env var controls what gets posted:
- `error` — Only definite bugs and security issues
- `warning` — Bugs + suspicious patterns + style issues (default-ish)
- `info` — Everything including nitpicks and minor suggestions

## How the Review Works

### Diff Parsing

The script fetches the PR diff in unified diff format from the GitHub API, then splits it into per-file sections. For each file it extracts:
- The file path
- Added/modified lines with their **exact line numbers** in the new file
- Surrounding context (3 lines by default from the diff)
- Total added line count (used for large-file detection)

Only changed lines are reviewed — the bot doesn't re-review untouched code.

### Claude Analysis

Each file's diff is sent to Claude with a structured prompt asking for issues in JSON format:

```json
[
  {
    "line": 42,
    "end_line": 58,
    "severity": "warning",
    "category": "bug",
    "title": "Potential null dereference",
    "description": "At line 42, mesh.vertices could be None if loading fails...",
    "suggestion": "if mesh.vertices is not None:\n    process(mesh.vertices)"
  }
]
```

Claude is prompted to report on seven categories: `bug`, `security`, `performance`, `style`, `large-file`, `large-function`, and `duplication`.

### Cross-File Duplication Detection

After Claude reviews individual files, a lightweight duplication detector runs across all changed files. It extracts contiguous blocks of added code (≥ `DUPLICATION_MIN_LINES` lines), normalizes whitespace, and flags blocks that appear in multiple locations. This catches copy-paste duplication that Claude can't detect since it reviews files independently.

### GitHub Review Posting

Issues are collected across all files and posted as a single PR review using GitHub's pull request review API. This means:
- All comments appear as one review, not a flood of individual comments
- Each comment shows the severity emoji, category, and **exact line reference** in its header
- Multi-line issues use GitHub's `start_line` + `line` API for highlighted ranges
- Suggestions use GitHub's ````suggestion` syntax for one-click acceptance
- The review is marked as "COMMENT" (not "REQUEST_CHANGES") to avoid blocking PRs

## Comment Format

Each inline comment follows this format:

```
🟡 **Potential null dereference** (`bug` · `warning` · Lines 42–58)

At line 42, mesh.vertices could be None if loading fails. This would cause
an AttributeError on line 45 when accessing .shape.

​```suggestion
if mesh.vertices is not None:
    process(mesh.vertices)
​```
```

## Limitations

- **Large PRs**: PRs with more than `MAX_FILES` changed files are skipped with a comment explaining why
- **Binary files**: Skipped automatically (images, compiled files, etc.)
- **Generated files**: Files matching common generated patterns (lock files, migrations, etc.) are skipped
- **Rate limits**: The action respects Anthropic API rate limits. Very large PRs may take a few minutes.
- **No full-repo context**: The bot only sees the diff + file paths, not the entire codebase. It may miss issues that require broader context.
- **Cross-file duplication**: The built-in detector only compares added code blocks. It won't catch duplication against existing (unchanged) code.

## Troubleshooting

- **Action doesn't trigger**: Check that the workflow file is on the default branch (usually `main`). GitHub Actions for `pull_request` only trigger if the workflow exists on the base branch.
- **No comments posted**: Check the action logs. Common issues: invalid API key, rate limiting, or the diff was empty/binary-only.
- **Comments on wrong lines**: This can happen if the diff is very large and gets truncated. Increase `MAX_DIFF_SIZE` or review the file manually.
- **Want to skip specific files**: Add patterns to the `SKIP_PATTERNS` list in `review_pr.py`.

## Files

- `scripts/pr-review.yml` → Copy to `.github/workflows/pr-review.yml`
- `scripts/review_pr.py` → Copy to `.github/scripts/review_pr.py`
