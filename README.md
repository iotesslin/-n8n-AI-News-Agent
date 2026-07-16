# AI News Agent

An n8n automation that collects AI-related news daily, scores and summarizes it with an LLM (Gemini), routes it through a human-approval step in Google Sheets, and sends a weekly curated HTML digest by email.

Built for internal team "AI Sharing" sessions, but the scoring criteria and news sources are easy to adapt to any team or industry.

## How it works

Two independent n8n workflows communicate only through a shared Google Sheet — no direct workflow-to-workflow calls, so each can be debugged, scheduled, or re-run on its own.

```
Workflow A (daily)                         Workflow B (weekly)
──────────────────                         ───────────────────
RSS feeds (multiple sources)               Read "Pending" rows where
   │                                       Approved = TRUE and Sent = FALSE
   ▼                                                │
Merge + dedupe against "Processed" history          ▼
   │                                       Sort by score, build HTML digest
   ▼                                                │
Filter to last 7 days, sort newest-first            ▼
   │                                       Send via SMTP
   ▼                                                │
Cap to top N items                                  ▼
   │                                       Mark rows as Sent = TRUE
   ▼                                                │
Single batched Gemini call                          ▼
(scores + summarizes all items at once)    Clear "Pending" sheet entirely
   │                                                │
   ▼                                                ▼
Score ≥ threshold → write to "Pending"     Prune "Processed" to the last
All items (any score) → "Processed"        14 days, deduped by link, and
                                            rewrite it back
```

- **Workflow A** runs once a day, gathers candidate news, scores it with a single batched LLM call (not one call per article — this matters for free-tier API rate limits), and files it into two Google Sheet tabs: `Pending` (score ≥ threshold, awaiting human sign-off) and `Processed` (everything, regardless of score, kept as a full history and dedupe source).
- **Workflow B** runs once a week. A human reviews `Pending` beforehand and ticks the `Approved` checkbox on whatever should go out. The workflow sends one digest email, marks those rows as sent, then resets `Pending` for the next cycle and trims `Processed` down to a rolling 14-day window.

## Files

| File | Description |
|---|---|
| `AI_News_Agent_A_collection_and_rating.json` | Daily collection, deduplication, and Gemini-based scoring workflow |
| `AI_News_Agent_B_review_and_send.json` | Weekly review digest, email send, and sheet cleanup workflow |

## Prerequisites

- An n8n instance (Cloud or self-hosted)
- A Google account, with:
  - A Google Sheet with two tabs: `Pending` and `Processed` (schema below)
  - Google Sheets OAuth2 credentials configured in n8n
- A Gemini API key ([aistudio.google.com/apikey](https://aistudio.google.com/apikey))
- An SMTP-capable email account for sending the digest

## Setup

### 1. Create the Google Sheet

Create a spreadsheet with two tabs and these exact column headers in row 1:

**`Pending`**
```
Title | Summary | Category | Score | Reason | Link | Publication Date | Approved | Sent
```

**`Processed`**
```
Title | Summary | Category | Score | Reason | Link | Publication Date | Processing Date
```

Format `Approved` and `Sent` as checkboxes — apply the checkbox format to the header row only, not a large pre-formatted range, or new rows can get appended far below your actual data.

### 2. Set up credentials in n8n

- **Google Sheets OAuth2**: `Credentials → New → Google Sheets OAuth2 API`, authorize with the Google account that owns the sheet above.
- **SMTP**: `Credentials → New → SMTP`. For Gmail, this requires an [App Password](https://myaccount.google.com/apppasswords) (two-step verification must be enabled first) — host `smtp.gmail.com`, port `587`, STARTTLS on.
- **Gemini API key**: no n8n credential needed — it's passed as a header value directly in the HTTP Request node (see placeholders below).

### 3. Import the workflows

Import both JSON files into n8n (`Import from File`, or paste the JSON directly onto an empty canvas).

### 4. Replace the placeholders

Both files use `{PLACEHOLDER}` tokens for anything environment-specific. Search each file for these and replace them, then reselect the credential dropdowns on every Google Sheets / SMTP node (placeholder credential IDs won't resolve to anything real):

| Placeholder | Where | Replace with |
|---|---|---|
| `{Sheet ID}` | Both workflows, all Google Sheets nodes | The spreadsheet ID from its URL (`.../d/<this part>/edit`) |
| `{Processed Sheet GID}` | Both workflows, all nodes referencing the `Processed` tab | The `gid` number of the `Processed` tab (visible in its URL after you click into it) |
| `{Gemini API Key}` | Workflow A, "Gemini Ratings and Summary" node header | Your Gemini API key |
| `{SMTP_USER}` | Workflow B, "SMTP" node | The sending email address |
| `{Recipient}` | Workflow B, "SMTP" node | Recipient address(es), comma-separated for multiple |
| `REPLACE_WITH_YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Every Google Sheets node | Reselect the credential from the dropdown in the n8n UI |
| `REPLACE_WITH_YOUR_SMTP_CREDENTIAL_ID` | Workflow B, "SMTP" node | Reselect the credential from the dropdown in the n8n UI |

The `Pending` tab is assumed to be the first tab in the spreadsheet (`gid=0`); if it isn't, update that value too.

### 5. Adjust the scoring prompt for your team

The prompt lives in Workflow A's "Combined Batch Request" node. It's written for a generic PC/hardware R&D team — edit the department background, keyword examples, and category list to match your own team's focus before relying on the scores.

Keep in mind this prompt is sent to an external LLM API on every run — avoid putting confidential company names, partner lists, or other sensitive internal details directly in it.

### 6. Test before activating

1. Run Workflow A manually (`Execute workflow`). Check that `Pending` and `Processed` receive rows.
2. Tick `Approved` on a row or two in `Pending`.
3. Run Workflow B manually. Confirm the email arrives, `Sent` gets marked, `Pending` clears, and `Processed` is pruned/deduped correctly.
4. Toggle both workflows to **Active** once you're satisfied.

## Data model

### `Pending`

| Column | Description |
|---|---|
| Title | LLM-generated short headline |
| Summary | 2–3 sentence summary |
| Category | LLM-assigned category label |
| Score | 0–10 integer relevance score |
| Reason | One-line justification for the score |
| Link | Original article URL — used as the dedupe/match key |
| Publication Date | The article's original publish date |
| Approved | Checkbox — human sign-off to include in this week's digest |
| Sent | Checkbox — set automatically once the digest is sent |

### `Processed`

Same first seven columns as `Pending`, minus `Approved`/`Sent`, plus:

| Column | Description |
|---|---|
| Processing Date | Timestamp of when the system scored this item (distinct from the article's own publish date) |

`Processed` logs **every** scored item regardless of score, both to prevent re-processing the same article and to let a reviewer catch anything the LLM under-scored.

## Scoring

Each article is scored 0–10 by a single batched LLM call (all candidate articles are sent in one request, not one request per article — see [Design notes](#design-notes)) across four dimensions:

- **internal_relevance** — relevance to your team's tech stack and work
- **efficiency_value** — potential to improve individual/team productivity
- **must_know** — industry-significance regardless of direct relevance
- **learning_value** — educational value even without immediate work relevance

Suggested score bands: 9–10 (must-read for everyone), 7–8 (highly relevant), 5–6 (relevant to specific roles), 3–4 (marginal), 0–2 (not recommended). The default threshold for entering `Pending` is 7.

## Design notes

A few non-obvious decisions, useful if you're modifying this:

- **Batched LLM scoring**: all candidate articles for a run are packed into a single prompt and scored in one Gemini call, rather than one call per article. This keeps daily API usage low enough to stay comfortably within free-tier rate limits.
- **Model alias, not a pinned version**: the Gemini model is referenced by a `-latest` alias rather than a specific dated version, since Google periodically deprecates specific model versions for new usage.
- **Dedup depends on `Processed`, not `Pending`**: `Pending` is fully cleared every week, so long-term dedup history lives entirely in `Processed`. The 14-day retention window is intentionally wider than the 7-day news collection window to leave a safety margin.
- **RSS feeds aren't inherently time-limited**: some RSS sources return their entire history rather than just recent items, so the workflow does its own date filtering rather than trusting the feed.

## Known limitations

- No fallback notification when no items are approved in a given week (the send step is silently skipped).
- Recipient list is static; not wired up to any directory/group source.
- All recipients get the same content — no role- or team-based personalization.
- HTML digest styling hasn't been tested against Outlook's desktop rendering engine, which has weaker CSS support than most webmail clients.

## License

Add a license of your choice here.
