---
name: scholar-inbox
description: Summarize latest Scholar Inbox digest by scraping scholar-inbox.com directly via Chrome (primary) or parsing the Gmail digest email (fallback). Posts to Slack #dimz-digest and archives high-score papers to Notion.
---

Scrape the latest Scholar Inbox digest, summarize each paper, post to **#dimz-digest** (`C0ALWLHUN2G`), and archive papers scoring 7+ to the Notion "Scholar Digest Archive" (data source `da036b39-948b-4948-892b-1dda6ff0661d`).

## Critical: Source-of-truth strategy (read this first)

The Gmail MCP `get_thread` tool **only returns the message snippet, not the full HTML body**. The "View Full Digest" link, paper titles, arXiv IDs, and figure URLs are all in the body. Do **not** rely on Gmail as the primary source. Do **not** fabricate paper titles, authors, or arXiv IDs to fill the gap — past runs did this and produced placeholder author fields in Notion that had to be backfilled.

Use this priority chain. Stop at the first step that yields the full digest HTML / paper list:

1. **Primary — Scholar Inbox via Chrome.** Navigate to https://www.scholar-inbox.com/ in a connected Chrome browser; the user is logged in there and the digest page lists all papers with arXiv links and figures already structured.
2. **Fallback A — Cached digest URL.** On every successful run, save the digest URL to `<workspace>/scholar-digest-state.json`. On the next run, if Chrome is unavailable, `web_fetch` the cached URL pattern directly (Scholar Inbox digest pages are public once you have the token).
3. **Fallback B — Manual paste.** Post a Slack status note in #dimz-digest asking the user to paste the email body or save it to the workspace. Wait for the next invocation; do not proceed.
4. **Never fabricate.** If all three fail, post a status note (not a digest) and stop. Do not guess at paper titles, authors, or arXiv IDs.

## Step 0: Enrich Incomplete Notion Entries

Before processing the new digest, scan the Notion "Scholar Digest Archive" (data source `da036b39-948b-4948-892b-1dda6ff0661d`) for entries that need backfill.

### Detect incomplete entries

An entry is **incomplete** if any of these is missing:
- `Score` is null, 0, or missing
- `Summary` is empty or missing
- `Authors` is empty, missing, OR a placeholder (e.g., `"X authors (arXiv ...)"`, `"Multi-institution"`, `"<paper> authors"`, or any string ≤ 3 words that doesn't contain a person's first+last name)
- `Relevance` is empty or missing
- `Relevance Justification` is empty or missing
- `Publication Date` is empty or missing

**Authors placeholder detection is critical** — past runs produced strings like "A2Z dataset authors (arXiv 2603.12605)" or "NYU, TU Berlin, U Victoria (multi-institution)". Treat any author string lacking a real first+last name as missing.

### Enrich

For each incomplete entry (process up to 10 per run, prioritize the most-empty ones):
1. Use the `arXiv Link` (or `web_search` the title) to find the paper.
2. `web_fetch` the arXiv abstract page; extract abstract + comma-separated author list (real names).
3. Generate summary + score using the rubric below.
4. `notion-update-page` with `update_properties`.

### Backfill Publication Date

For arXiv papers: the YYMM portion of the arXiv ID gives the year-month (e.g., `2604.02289` → April 2026). Fetch the abstract page for the exact "Submitted on" date when needed. Set `"date:Publication Date:start": "2026-04-02"`.

For non-arXiv papers (OpenReview, journals): `web_search` the publication date.

## Step 1: Get the digest content

### 1a. Try Chrome first

```
mcp__Claude_in_Chrome__list_connected_browsers
```

If at least one browser is connected:
1. Open or reuse a tab via `tabs_context_mcp` / `tabs_create_mcp`.
2. `navigate` to `https://www.scholar-inbox.com/`.
3. `read_page` to get the digest list. Extract paper title, authors, arXiv link, abstract snippet, and figure URLs for each entry.
4. Save the canonical digest URL to `<workspace>/scholar-digest-state.json` (`{"last_digest_url": "...", "last_run": "<ISO date>"}`).

If `list_connected_browsers` returns `[]`, skip to 1b.

### 1b. Try cached digest URL via web_fetch

Read `<workspace>/scholar-digest-state.json`. If it has a `last_digest_url`, attempt `web_fetch` on it. The current digest is usually accessible by replacing the date segment, but **don't guess the URL pattern blindly** — only use what's been observed.

### 1c. Try Gmail snippet for date validation only

Use `search_threads` with `subject:"Scholar Alert" newer_than:2d` to confirm the latest digest exists. The snippet won't have the body but **does** give the digest date and "we found N articles relevant" count — use that to sanity-check what you got from Chrome / cache.

### 1d. Fallback to manual paste

If 1a–1c all fail, post a Slack note in #dimz-digest:

> :warning: Scholar Digest — `<DATE>` — needs manual input
>
> Couldn't auto-fetch the digest (no Chrome browser connected and no cached URL). Either paste the Scholar Inbox email body in chat, save it as `<workspace>/scholar-digest-paste.txt`, or open scholar-inbox.com in Chrome with the Claude extension running. Then re-invoke me.

Stop after posting. Do not score or archive anything.

## Step 2: Per-paper enrichment

For each paper:

1. **Abstract.** If you have the arXiv URL, `web_fetch` the abstract page (`https://arxiv.org/abs/<id>`). If rate-limited, fall back to the snippet from the digest page.
2. **Authors.** Always extract real names from the arXiv abstract page or paper landing page. **Never** write "X authors", "multi-institution", or institutional names alone — those are placeholders. If author extraction fails, leave the field empty rather than fill it with a placeholder.
3. **Score (0–10).** Apply the rubric below.
4. **Summary.** 2–3 sentences answering: (a) what problem? (b) how solved (key methodology)? (c) what novelty?
5. **Figures.** For papers with arXiv IDs, `web_fetch https://arxiv.org/html/<id>` and extract 1–3 figure URLs (skip icons/logos). If rate-limited, mark the gap explicitly: "Figures unavailable — see paper".

## Relevance scoring

Score 0–10 based on alignment with LEO topics + the open Jira tasks (project = LEO).

**Core high-relevance (score 7+):**
- CAD feature-tree / history-tree generation; CAD feature tree editing
- BRep-to-CAD conversion (reverse engineering); STEP / B-Rep / mesh processing
- 3D generative modeling (diffusion, VAE, GAN, flow matching for shapes/meshes)
- CAD translation, parametric modeling, CSG; CadQuery / OpenCascade / FreeCAD automation
- Point cloud / voxel networks for CAD; shape completion, reconstruction, generation
- LLM-based code generation for CAD/3D; part-level 3D generation, assembly generation

**Medium (score 4–6):**
- General diffusion / score-based generative models
- Transformer architectures for 3D
- NeRF / Gaussian splatting; 3D scene understanding; geometric deep learning
- ML training infrastructure / distributed training

Cross-reference open LEO Jira tasks for additional relevance signal.

## Slack posting

Post to **#dimz-digest** (`C0ALWLHUN2G`) sorted by score descending. Post **all** papers regardless of score (low-score papers go to Slack only — not Notion). Use these emojis:

- 🔴 9–10 (Must-Read)
- 🟠 7–8 (Highly Relevant)
- 🟡 5–6 (Moderate)
- 🔵 3–4 (Tangential)
- ⚪ 1–2 (Low)

For each paper, the Slack body should contain:

```
<emoji> *<score>/10 — <title>*
arXiv:<id> | <publication month-year>
<2–3-sentence structured summary>
:link: https://arxiv.org/abs/<id>
:bulb: <one-line relevance justification referencing LEO topics or Jira tasks>
```

Reply to the main digest message with figure thumbnails in a thread (one reply per paper that has figures). Skip the thread reply for papers where figures couldn't be retrieved — say so explicitly: "Figures unavailable — see paper."

## Notion archive (papers scoring 7+ ONLY)

Papers below 7: Slack only, **no Notion entry**.

For each paper with score ≥ 7:

### Properties

Use `notion-create-pages` with `parent: {type: "data_source_id", data_source_id: "da036b39-948b-4948-892b-1dda6ff0661d"}`:

- `Paper Title` — the actual paper title
- `date:Date:start` — digest date (ISO, e.g. `"2026-04-24"`)
- `date:Publication Date:start` — paper's submission date (ISO, e.g. `"2026-04-23"`)
- `Score` — number 0–10
- `Relevance` — one of `"🔴 Must-Read"`, `"🟠 Highly Relevant"`, `"🟡 Moderate"`, `"🔵 Tangential"`, `"⚪ Low"`
- `Authors` — comma-separated **real** names. Never a placeholder. If unknown, leave empty.
- `Summary` — the 3-question structured summary
- `arXiv Link` — `https://arxiv.org/abs/<id>`
- `Relevance Justification` — one-line reason referencing topics / Jira tasks

Do **not** set `Read by` or `Figure URLs` — managed manually.

### Detailed page content

After creating the page, `notion-update-page` with `replace_content` to add:
- H1 title with score emoji + arXiv link
- Authors line
- "Structured Summary" section with **What problem? How solved? What novelty?** subsections
- "Why This Matters for LEO" section cross-referencing relevant Jira tasks and core topics
- "Key Figures" section with `![caption](url)` markdown images from arXiv HTML — or, if figures couldn't be retrieved, a single line: "Figures unavailable — see [arXiv](https://arxiv.org/abs/<id>)"

## Deduplication

Before creating Notion entries, fetch recent rows from `collection://da036b39-948b-4948-892b-1dda6ff0661d` and skip any paper that already exists by:
1. Exact title match (case-insensitive)
2. Same arXiv ID in the URL
3. Title differs only in punctuation, capitalization, or version suffix

Report skipped papers in a Slack thread reply: "ℹ️ Skipped N already-archived papers: …"

## Configuration

| Setting | Value |
|---|---|
| Slack channel | `#dimz-digest` (`C0ALWLHUN2G`) |
| Notion data source | `da036b39-948b-4948-892b-1dda6ff0661d` |
| Gmail search (date check only) | `subject:"Scholar Alert" newer_than:2d` |
| Min score for Notion archive | 7 |
| Min score for Slack | 0 (post all) |
| Max incomplete-entry enrichments per run | 10 |
| Cache file | `<workspace>/scholar-digest-state.json` |

## Important

- **This is a non-interactive scheduled task.** Don't ask for confirmation before posting; just post.
- **Never fabricate paper titles, authors, or arXiv IDs.** If you can't retrieve the digest, post a status note instead. The Apr 23 2026 run wrote placeholder author strings into Notion (`"A2Z dataset authors"`, `"NYU, TU Berlin, U Victoria (multi-institution)"`); those required manual cleanup.
- **Author extraction is required** before creating a Notion entry. If the arXiv abstract page doesn't load, skip Notion archival for that paper (still post to Slack with `Authors: unknown`).
- The figure-extraction step is best-effort. Rate-limited arXiv HTML fetches should be reported as "Figures unavailable", not silently dropped.
