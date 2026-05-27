---
name: scholar-digest
description: >
  Daily Google Scholar Alert digest processor. Reads "Scholar Alert" emails from
  Gmail, extracts individual papers, summarizes each in 2-3 sentences with 1-3
  relevant images and a relevance score (0-10), and posts the digest to the
  #papers Slack channel. Use this skill whenever the user mentions "scholar
  digest", "paper digest", "scholar alerts", "check my scholar emails",
  "papers from gmail", "paper summary", "research digest", "morning papers",
  or any variation of processing Google Scholar email alerts into summaries
  for Slack. Also trigger when the user says "run the paper digest", "check
  for new papers", "scholar alert", or "/schedule" combined with papers or
  scholar.
---

# Scholar Alert Digest Skill

Process Google Scholar Alert emails into concise paper summaries with relevance scores and images, posted to Slack #papers.

## Overview

Dimi receives Google Scholar Alert emails (subject contains "Scholar Alert" or "Google Scholar Alert"). This skill:

1. Searches Gmail for recent Scholar Alert emails
2. Extracts paper titles, authors, links, and snippets from each email
3. For each paper: fetches the abstract, generates a 2-3 sentence summary, scores relevance (0-10), and finds 1-3 illustrative images
4. Posts a formatted digest to **#papers** (`C05KDV8L8HG`) on Slack

## Relevance Scoring

Score each paper 0-10 based on alignment with Dimi's research areas at Leo AI Research:

| Score | Meaning |
|-------|---------|
| 9-10  | Directly relevant — 3D generative models, CAD generation, mesh-to-CAD, CadQuery, STEP/B-Rep processing |
| 7-8   | Strongly related — diffusion models for 3D, VAE architectures for shapes, point cloud generation, voxel representations |
| 5-6   | Moderately related — general 3D deep learning, geometric deep learning, shape representation, neural implicit surfaces |
| 3-4   | Tangentially related — general generative models (images, text), transformer architectures, ML infrastructure |
| 1-2   | Weakly related — broadly ML/AI but not geometry or generative |
| 0     | Not relevant — unrelated field entirely |

**Key high-relevance topics** (score 7+):
- 3D generative modeling (diffusion, VAE, GAN for shapes/meshes)
- CAD translation, reverse engineering, parametric modeling
- Mesh processing, B-Rep, STEP files, constructive solid geometry
- CadQuery, OpenCascade, FreeCAD automation
- Point cloud processing, voxel networks
- Shape completion, reconstruction, generation
- LLM-based code generation for CAD/3D

**Medium-relevance topics** (score 4-6):
- General diffusion models, score-based generative models
- Transformer architectures, attention mechanisms
- Neural radiance fields (NeRF), gaussian splatting
- Image generation, video generation
- ML training infrastructure, distributed training

## Step-by-Step Workflow

### Step 1: Find Scholar Alert Emails

Use `tool_search` to discover Gmail tools:

```
tool_search(query="Gmail search messages")
```

Then search for recent Scholar Alert emails:

```
Gmail search for: "subject:Scholar Alert newer_than:1d"
```

If Gmail tools are not available, ask the user to either:
- Paste the Scholar Alert email content directly
- Forward the email content to the chat

Parse the email body to extract individual papers. Scholar Alert emails typically contain:
- Paper title (linked to Google Scholar or publisher)
- Authors list
- Short snippet/excerpt
- Source publication

### Step 2: Extract Paper Details

For each paper found in the email:

1. **Title**: Extract the paper title from the email HTML/text
2. **Authors**: Extract author names
3. **Link**: Extract the Google Scholar link or direct publisher link
4. **Snippet**: The brief excerpt Google Scholar includes

Collect all papers into a list for processing.

### Step 3: Fetch Full Abstracts

For each paper, try to get the full abstract:

1. Use `web_fetch` on the paper's link (often leads to arXiv, IEEE, ACM, etc.)
2. If that fails, use `web_search` with the exact paper title to find it
3. Extract the abstract from the fetched page
4. If no abstract is available, use the snippet from the email

### Step 4: Generate Summary + Score

For each paper, produce:

- **Summary**: 2-3 sentences covering:
  - What the paper does / proposes
  - Key methodology or approach
  - Main result or contribution
- **Relevance Score**: 0-10 based on the scoring rubric above
- **Score Justification**: One short phrase explaining why (e.g., "Direct: 3D mesh generation with diffusion")

### Step 5: Find Illustrative Images

Finding real, verified figure URLs is critical. Never guess or construct URLs without actually fetching the page first — broken image links in Slack are worse than no images at all.

Many papers (especially from conference proceedings like AAAI, CVPR, NeurIPS) don't have arXiv preprints, so a single-source approach will miss most of them. Use this multi-source fallback chain, trying each source in order and moving on only when the previous one fails.

#### Source 1 — arXiv HTML (best for arXiv papers)

If the paper has an arXiv ID, fetch the HTML version using `web_fetch`:
```
web_fetch(url="https://arxiv.org/html/PAPER_ID", prompt="List all image URLs (src attributes of <img> tags) on this page. Return only the full URLs, one per line. Skip any images smaller than 100px or that look like icons/logos.")
```

The returned URLs are real, verified figure URLs. Pick 1-3, prioritizing:
- Figure 1 (usually the method overview / architecture diagram)
- Key results figures (qualitative comparisons, generated outputs)
- Skip tiny icons, logos, and LaTeX-rendered equation images

#### Source 2 — Project page / GitHub (best for conference-only papers)

Many papers have companion project pages with high-quality teaser images. Search for them:
```
web_search("<paper title> project page")
web_search("<first author surname> <short paper name> github.io")
```

When you find a project page, fetch it and extract image URLs:
```
web_fetch(url="<project_page_url>", prompt="List all image URLs on this page that look like paper figures, teaser images, or method diagrams. Return full URLs only.")
```

#### Source 3 — Semantic Scholar (to discover missing arXiv IDs)

Semantic Scholar often knows the arXiv ID for papers that weren't linked in the digest email:
```
web_fetch(url="https://api.semanticscholar.org/graph/v1/paper/search?query=<url-encoded-title>&limit=1&fields=title,openAccessPdf,externalIds", prompt="Return the arXiv ID, DOI, and open access PDF URL if available.")
```

If this reveals an arXiv ID you didn't already have, go back to Source 1. Note: Semantic Scholar has rate limits — if you get a 429, skip it.

#### Source 4 — OpenReview (for conference papers)

Top conferences (AAAI, NeurIPS, ICLR, ICML) host papers on OpenReview:
```
web_search("openreview <paper title> <conference> <year>")
```

OpenReview pages may link to PDFs or supplementary materials. Use Chrome (if available) to navigate these pages, as they sometimes block direct `web_fetch`.

#### Source 5 — Author's institutional page

Search for the paper on the authors' homepages:
```
web_search("<last author name> <first few title words> publications")
```

Author pages sometimes host project pages, slides, or supplementary materials with figures.

#### When no figures are found

After exhausting all sources, clearly note the gap rather than making something up:
- In Slack: "No public figures available — [venue] proceedings only"
- In Notion: "Figures will be added when a preprint becomes available"
- Link to the proceedings page or venue URL if you have one

**Important**: Slack auto-unfurls direct image URLs (`.png`, `.jpg`) in messages. Include them as bare URLs on their own line in the message body — Slack will render them inline.

### Step 6: Post to Slack

Post the digest to **#papers** channel (`C05KDV8L8HG`).

**Important**: Ask the user for confirmation before sending to Slack. Show them a preview of the message first.

Format each paper as a Slack message block. If there are many papers (>5), split across multiple messages to stay within Slack's 4000-char limit.

#### Message Format Template

```
📚 *Scholar Alert Digest — [DATE]*

---

*[RELEVANCE_EMOJI] [SCORE]/10 — [PAPER_TITLE]*
_[AUTHORS]_
[2-3 SENTENCE SUMMARY]
🔗 [LINK]
💡 _Relevance: [JUSTIFICATION]_

---

[NEXT PAPER...]
```

**Relevance emojis by score:**
- 9-10: 🔴 (must-read)
- 7-8: 🟠 (highly relevant)
- 5-6: 🟡 (moderately relevant)
- 3-4: 🔵 (tangential)
- 1-2: ⚪ (low relevance)

**Sorting**: Present papers sorted by relevance score (highest first).

**Images**: Post the main digest as one message, then reply in a thread with 1-3 figure images per paper. Each thread reply should contain:
- Paper title and score emoji
- 1-3 direct image URLs (bare URLs on their own line — Slack auto-unfurls these)
- Brief caption describing what the figure shows

Thread reply format per paper (with figures):
```
[RELEVANCE_EMOJI] *[PAPER_TITLE] — Key Figures*
Fig 1: [Brief caption]
[DIRECT_IMAGE_URL]

Fig 2: [Brief caption]
[DIRECT_IMAGE_URL]
```

Thread reply format per paper (no figures available):
```
[RELEVANCE_EMOJI] *[PAPER_TITLE]*
No public figures available — [VENUE] proceedings only.
[LINK_TO_PROCEEDINGS_OR_PAPER_PAGE]
```

Group all papers into a single thread reply, organized by whether figures were found or not. Papers with verified figures first, then papers without.

### Step 7: Thread with Details (Optional)

If the user wants more detail, reply in a thread under the main digest message with:
- Full abstracts
- Additional images
- Links to related papers in the team's history

## Notion Deduplication

Before adding papers to the Notion "Scholar Digest Archive" database, always check for existing entries to avoid duplicates. This matters because digests often overlap across days — the same paper can appear in consecutive Scholar Inbox emails, and re-runs of the skill on the same day would otherwise create duplicate rows.

### How to Check

Before creating Notion pages, fetch recent entries from the database and compare against the papers you're about to insert:

```
notion-fetch(url="collection://da036b39-948b-4948-892b-1dda6ff0661d")
```

Scan the returned rows' "Paper Title" and "arXiv Link" fields against your paper list.

### Matching Rules

A paper is considered a duplicate if **any** of these match an existing row:

1. **Exact title match** (case-insensitive) → skip
2. **Near-identical title** — differs only in punctuation, capitalization, or a trailing version suffix like "v2" → skip
3. **Same arXiv ID** in the URL (e.g., both contain `2603.24278`) → skip, even if titles differ slightly

If none of these match, proceed with insertion.

### Reporting

When papers are skipped, include a note at the end of the digest output so there's visibility:
- In Slack thread: "ℹ️ Skipped N papers already in Notion: [list of titles]"
- In the task summary: list the skipped titles and why they matched

This keeps the database clean without silently dropping papers.

## Edge Cases

- **No Scholar Alert emails found**: Report "No new Scholar Alert emails found in the last 24 hours." Suggest checking Gmail directly or adjusting the time window.
- **Gmail tools unavailable**: Ask the user to paste the email content. Parse it the same way.
- **Paper link unreachable**: Use the email snippet for the summary, note that the full abstract wasn't accessible.
- **Very long digest (>10 papers)**: Split into multiple Slack messages. Consider only posting papers with score ≥ 3 and mentioning the count of lower-scored papers.
- **Duplicate papers across alerts**: Deduplicate by title before processing. Also check Notion database before insertion (see Notion Deduplication above).
- **Non-English papers**: Summarize in English regardless of the paper's language.

## Configuration

| Setting | Value |
|---------|-------|
| Slack Channel | #papers (`C05KDV8L8HG`) |
| Gmail Search | `subject:Scholar Alert newer_than:1d` |
| Min Score to Post | 0 (post all, sorted by relevance) |
| Summary Length | 2-3 sentences per paper |
| Images per Paper | 1-3 |
| Score Range | 0-10 |

## Important Notes

- Always confirm with the user before posting to Slack
- Use `web_fetch` and `web_search` to get paper abstracts — don't rely solely on email snippets
- Keep summaries concise but informative — the team should know at a glance whether to read the paper
- The relevance score is from Dimi's LeoCAD perspective — 3D generative modeling and CAD translation are the core focus
- Images should be illustrative (architecture diagrams, result visualizations) not generic stock photos
- If running as a scheduled task, the skill should be non-interactive: skip confirmation and post directly
