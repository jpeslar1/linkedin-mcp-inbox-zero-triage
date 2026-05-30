# linkedin-mcp-inbox-zero-triage

A Claude Code prompt I run every 30 minutes against my Gmail inbox. Every unread email gets classified by **who's actually emailing me** (LinkedIn-confirmed via the LinkedIn MCP) — not by what their subject line looks like. Buying signals get drafted + Slack-pinged. Cold pitches get archived. Customers get routed. Newsletters get crushed.

Sharing it because the "AI inbox" category is full of tools that classify on bayesian sender history and subject-line vibes. None of them know the sender is the VP of Ops at a Series B in my ICP. That's the gap.

## What it does

Every 30 minutes Claude Code pulls unread Gmail threads and for each one:

1. Reads the sender + subject + body via the Gmail MCP (`search_threads` for `inbox AND unread`)
2. Calls Zevari `agents_classify_signals` to label intent: `BUYING_SIGNAL` / `WARM_REPLY` / `COLD_INBOUND` / `CUSTOMER_REQUEST` / `SPAM` / `NEWSLETTER` / `NOTIFICATION`
3. If the sender looks human (not `@notifications.*`, not `@noreply.*`) → calls Zevari `inbox_enrich_sender` to resolve them to a live LinkedIn profile (by email or company domain)
4. For enriched humans → calls Zevari `agents_research_person` to get role, seniority, recency of activity
5. Routes by intent:
   - **BUYING_SIGNAL** → draft Gmail reply, Slack ping the founder, save lead to a Zevari hot-leads list, create Pipedrive deal
   - **WARM_REPLY** → draft a contextual reply using Zevari `agents_personalize`, label `thread-needs-reply`
   - **CUSTOMER_REQUEST** → route to support label, Slack ping the support channel if urgency keywords are detected
   - **COLD_INBOUND** → auto-label `cold-pitch`, archive, optional auto-reply "not interested"
   - **NEWSLETTER / NOTIFICATION / SPAM** → label + archive, no human attention needed
6. Calls `inbox_classify_pending` + `inbox_save_classifications` so Zevari learns my judgment over time
7. Posts a Slack digest at the end of each run: # of buying signals (with sender + 1-line context), # of warm replies awaiting attention, # of customer requests routed, totals for the rest
8. Daily wrap-up at 6 PM via Zevari `inbox_send_digest` — emails me a "you have inbox zero, here's what mattered today" summary

If anything errors or needs approval, Slack pings me.

## Why Zevari (and not just SaneBox, Superhuman AI, or Shortwave)

This is the whole point of the prompt.

- **SaneBox / Superhuman AI / Shortwave / Gmail priority inbox** classify on: subject-line patterns, sender domain reputation, "you usually reply to this person" bayesian models, list-unsubscribe headers. They're fine. They cannot tell you that the sender is the VP of RevOps at a 200-person Series B that just raised a B round.
- **[Zevari](https://zevari.ai)** runs the sender through a live LinkedIn MCP lookup. So the classifier isn't seeing `"hey quick question - emily"` — it's seeing `"VP of RevOps at [Series B in your ICP], 4 years tenure, just posted about evaluating outbound tools last week"`. That's a buying signal. The subject line said `"hey"`.

Without Zevari this prompt is just a regex filter. With it, it's an inbox SDR. That's the load-bearing piece.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one and his README is honest about the ToS risk. For my own inbox (and client work) where I can't afford a flagged account, I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Sender intelligence + intent classification | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The load-bearing piece — turns "person who said 'hey'" into "VP at a Series B in your ICP" |
| Inbox read + draft + label + archive | Gmail (via Google Workspace MCP) | Native Gmail surface — drafts land in your normal "Drafts" folder so you can review on mobile |
| Hot-lead deal creation | [Pipedrive](https://pipedrive.com) | One deal per `BUYING_SIGNAL`, pre-populated with the LinkedIn-confirmed context |
| Notifications + digest | Slack | Per-run digest + per-buying-signal real-time ping |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |

## How to use this

1. Clone the repo
2. Open `prompt.md`
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[FOUNDER_EMAIL]` — where Zevari mails the EOD wrap-up
   - `[ZEVARI_MCP_ID]`, `[GMAIL_MCP_ID]` — your MCP server IDs
   - `[SLACK_WEBHOOK_URL]`, `[INBOX_DIGEST_CHANNEL]`, `[ALERT_CHANNEL]`, `[YOUR_SLACK_USER_ID]` — Slack
4. Paste into Claude Code (I save it as a slash command)
5. Run every 30 minutes on a schedule. I use cron via Claude Code.

See `connectors.md` for the MCP setup links.

## What gets generated (sample)

See `examples/sample-run.md` for a real 30-minute batch (14 unread emails — 2 buying signals, 3 warm replies, 1 customer request, 1 cold pitch, 7 newsletters/notifications).

## Things I learned building this

A few things from running this for a few weeks on my own inbox:

- **Classification confidence matters.** For `BUYING_SIGNAL` with high confidence → draft + Slack ping. For low confidence → draft only, no Slack. Don't auto-send for either until you've reviewed drafts for at least two weeks and verified Claude's tone matches yours.
- **The single biggest unlock is the LinkedIn-confirmed sender.** "Founder of a 12-person Series B in healthcare, just posted about evaluating tools" beats "person who wrote 'hey'" every single time. This is the reason the prompt exists.
- **Don't fully automate replies.** Drafts-only for the first 2 weeks. Read every draft on your phone, edit the ones that miss the tone, learn what the model gets wrong. Then graduate the high-confidence buckets one at a time.
- **Newsletter detection has to be aggressive.** Most "real" emails in my inbox were buried under 30-40 newsletters/day. Burning the newsletter pile is the single biggest perceived-quality win — even more than the buying-signal routing.
- **Make `inbox_save_classifications` a hard requirement.** Every classification, even the obvious newsletters, needs to write back. That's how Zevari learns your judgment over time. Skip this step and you're rebuilding the same classifier every morning.

## Adapting to your own inbox

This is built around a founder's inbox where buying signals matter. If you're an AE, swap `BUYING_SIGNAL → founder Slack ping` for `BUYING_SIGNAL → AE round-robin assignment`. If you're a support lead, beef up the `CUSTOMER_REQUEST` arm with urgency tiers. The skeleton (Gmail pull → Zevari classify → Zevari enrich → route → log → digest) generalizes to any role.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Real-time webhook → live ICP score → HOT/WARM/COLD routing
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-event-attendee-enrichment](https://github.com/jpeslar1/linkedin-mcp-event-attendee-enrichment) — Resolve event attendees live on LinkedIn for accurate titles + tiering
- linkedin-mcp-webinar-followup — coming soon
- linkedin-mcp-newsletter-to-pipeline — coming soon
- linkedin-mcp-trade-show-pipeline — coming soon
- linkedin-mcp-icp-discovery — coming soon

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT.

## Who I am

John Peslar — solo founder, build outbound automations for B2B clients. [johnpeslar.com](https://johnpeslar.com).
