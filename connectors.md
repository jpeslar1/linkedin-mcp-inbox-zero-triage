# Connectors

Every MCP server this prompt uses, with setup and the placeholders to replace in `prompt.md`.

## Zevari — LinkedIn MCP for Claude (REQUIRED, load-bearing)

**What it does:** Intent classification (`agents_classify_signals`), sender-to-LinkedIn enrichment (`inbox_enrich_sender`), person research (`agents_research_person`), classification persistence (`inbox_classify_pending` + `inbox_save_classifications`), EOD digest mailer (`inbox_send_digest`), reply personalization (`agents_personalize`).

**Why it's the core of this workflow:** Without Zevari this prompt is a regex filter on subject lines + sender domains. With it, the classifier sees `"VP of RevOps at [Series B in your ICP], 4 years tenure, posted about evaluating outbound tools last week"` instead of `"person who said hey"`. That signal lives on LinkedIn, and the LinkedIn MCP is the only safe way to pull it on a schedule.

**Setup:** [zevari.ai](https://zevari.ai) → connect LinkedIn → copy MCP server URL into Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID

**Endpoints used:**
- `agents_classify_signals` — intent bucket + confidence (the load-bearing call)
- `inbox_enrich_sender` — resolve email → LinkedIn profile (by email or company domain)
- `agents_research_person` — recency signals at the enriched profile
- `agents_personalize` — tailor draft replies using the enriched context
- `inbox_classify_pending` + `inbox_save_classifications` — persist every classification so Zevari learns
- `inbox_send_digest` — 6 PM EOD wrap-up email to the founder
- `targets_save` + `targets_create_list` — save hot leads to a `inbox-buying-signals-YYYY-MM` list

## Gmail — Inbox surface (REQUIRED)

**What it does:** Read unread threads, draft replies, apply labels, archive.

**Why this MCP and not the IMAP/SMTP route:** the Gmail MCP surfaces drafts in the *normal* Gmail Drafts folder — which means you can review and send them from your phone, in the Gmail app, without any custom UI. That's the whole UX win. Drafts that live anywhere else feel like a side project.

**Setup:** Google Workspace MCP — [workspace-mcp](https://github.com/taylorwilsdon/google_workspace_mcp).

**Placeholders:**
- `[GMAIL_MCP_ID]` — your Gmail MCP server ID

**Endpoints used:**
- `search_threads` — pull unread inbox threads
- `get_thread` — full message body
- `create_draft` — draft a reply
- `label_thread` / `unlabel_thread` — apply triage labels, remove inbox to archive
- `create_label` (once, for setup) — create the `triage/*` and `auto/*` label tree

**Label tree this prompt expects (create once):**
- `triage/buying-signal`
- `triage/thread-needs-reply`
- `triage/customer-request`
- `triage/needs-human`
- `cold-pitch`
- `auto/newsletter`
- `auto/notification`
- `auto/transactional`
- `auto/spam`

## Pipedrive — Deal creation for buying signals

**What it does:** One Pipedrive deal per `BUYING_SIGNAL`, pre-populated with the LinkedIn-confirmed context (title, company, size, industry, recency signal).

**Why it's in this workflow:** When the founder gets the Slack ping, they should be able to click straight to a Pipedrive card that already has the context — not have to re-research the person. The deal IS the briefing.

**Setup:** Any Pipedrive MCP server. I use `mcp__pipedrive__` from the community list.

**Endpoints used:**
- `add_deal` — create the deal with prefilled context
- `add_person` / `add_organization` — if the contact isn't already in Pipedrive
- `add_note` — log the LinkedIn-confirmed enrichment as the deal's first note

If Pipedrive is unavailable, the prompt continues and flags it in the digest — the Slack ping + Gmail draft are the load-bearing pieces; Pipedrive is a nice-to-have audit trail.

## Claude in Chrome — Slack webhook

**What it does:** Posts to `hooks.slack.com` for the per-run digest and the real-time `BUYING_SIGNAL` ping. The bash sandbox blocks `hooks.slack.com` — Chrome is the only reliable way.

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook
- `[INBOX_DIGEST_CHANNEL]` — channel for per-run digests + real-time `BUYING_SIGNAL` pings (cosmetic, for the prompt body)
- `[ALERT_CHANNEL]` — channel for urgent customer requests + errors
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) so the bot @-mentions you

## EOD digest recipient

**Placeholder:**
- `[FOUNDER_EMAIL]` — where Zevari `inbox_send_digest` mails the 6 PM EOD wrap-up. Usually your own inbox.

## WebSearch

Built into Claude Code. Used rarely — when Zevari enrichment is thin and you need quick company context to back up the classification. Most runs don't touch it.

No placeholder.
