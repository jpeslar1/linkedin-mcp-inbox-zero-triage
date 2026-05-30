# Inbox Zero Triage — 30-Minute Sweep

A Claude Code prompt that triages every unread email in my Gmail inbox by **LinkedIn-confirmed sender intent**. Drafts replies to buying signals, routes customer requests, crushes newsletters, archives cold pitches. Runs every 30 minutes.

The reason this prompt exists: every "AI inbox" tool I tried (SaneBox, Superhuman AI, Shortwave, Gmail's built-in priority inbox) classifies on subject-line patterns + sender domain + bayesian reply history. None of them know that the sender is the VP of RevOps at a Series B in my ICP. That signal lives on LinkedIn, and the LinkedIn MCP (Zevari) is the only safe way to pull it on a schedule.

---

## Overview

You are running the inbox triage sweep. For each unread Gmail thread:

1. Pull sender + subject + body via Gmail MCP.
2. Classify intent via Zevari `agents_classify_signals` → `BUYING_SIGNAL` / `WARM_REPLY` / `COLD_INBOUND` / `CUSTOMER_REQUEST` / `SPAM` / `NEWSLETTER` / `NOTIFICATION`.
3. If sender looks human (not `@notifications.*`, not `@noreply.*`, not `@updates.*`, not `@mailer-daemon.*`) → enrich via Zevari `inbox_enrich_sender` to resolve to a live LinkedIn profile (by email first, then company domain).
4. For enriched humans → Zevari `agents_research_person` for role / seniority / recency.
5. Route by intent (see routing table below).
6. Log every classification via `inbox_classify_pending` + `inbox_save_classifications` so Zevari learns your judgment.
7. Slack digest at the end of the run.
8. At 6 PM local: Zevari `inbox_send_digest` → email the EOD wrap-up to `[FOUNDER_EMAIL]`.

Run cadence: every 30 minutes. Stop processing if more than 80 unread threads — flag in Slack and ask whether to back-process.

---

## ERROR & APPROVAL NOTIFICATIONS

If anything stops the task or needs approval, send to `[ALERT_CHANNEL]` via Chrome (bash sandbox blocks `hooks.slack.com`):

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on `chrome://`, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Inbox Triage — Action Needed*\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — intent classification, sender enrichment, person research, classification logging, EOD digest, reply personalization
- **Gmail:** `mcp__[GMAIL_MCP_ID]` — read threads, draft replies, label, archive
- **Pipedrive:** `mcp__pipedrive__` — deal creation for `BUYING_SIGNAL`
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook
- **WebSearch:** for context-gathering when enrichment is thin (rare)

See `connectors.md`.

---

## STEP 1 — PULL UNREAD THREADS

Call Gmail MCP `search_threads` with the query:

```
label:inbox AND label:unread
```

Take all results, up to 80. If > 80, post a Slack notice and process only the most recent 80; the rest get picked up next run.

For each thread, pull:
- Sender email + name
- Subject
- First message body (full text)
- Thread ID
- Any existing labels

---

## STEP 2 — DOMAIN PRE-FILTER

Before sending anything to Zevari, classify obviously non-human senders by domain pattern:

Auto-label + archive immediately (skip classification entirely):
- `noreply@*`, `no-reply@*`, `notifications@*`, `updates@*`, `alerts@*`, `mailer-daemon@*`, `bounces@*`, `postmaster@*` → label `auto/notification`
- Known transactional senders (Stripe, AWS, GitHub notifications, calendar invites) → label `auto/transactional`

Everything that survives this filter goes to Zevari.

---

## STEP 3 — CLASSIFY INTENT (ZEVARI)

For each surviving thread, call Zevari `agents_classify_signals`:

```
{
  sender_email,
  sender_name,
  subject,
  body,
  intents: ["BUYING_SIGNAL", "WARM_REPLY", "COLD_INBOUND", "CUSTOMER_REQUEST", "SPAM", "NEWSLETTER", "NOTIFICATION"]
}
```

The response returns:
- `intent` — the classified bucket
- `confidence` — 0.0 to 1.0
- `reasoning` — one-line why

Confidence handling:
- `>= 0.8` = high confidence → act per routing table
- `0.5 - 0.8` = medium → act per routing table BUT do not auto-send anything (drafts only, no Slack ping unless `BUYING_SIGNAL`)
- `< 0.5` = low → label `triage/needs-human`, leave unread, skip routing

---

## STEP 4 — ENRICH HUMAN SENDERS (ZEVARI)

For threads classified as `BUYING_SIGNAL`, `WARM_REPLY`, `COLD_INBOUND`, or `CUSTOMER_REQUEST`:

Call Zevari `inbox_enrich_sender` with `{ email, name, company_domain_from_email }`. This resolves the sender to a LinkedIn profile via:
1. Direct email match in Zevari's index
2. Fallback: company domain → company LinkedIn → person search by name within company

If enrichment succeeds, you get back: `linkedin_url`, `current_title`, `current_company`, `company_size`, `company_industry`, `tenure_in_role`.

If enrichment fails (no LinkedIn match) → continue, but mark `enriched: false` in the classification log. Routing still happens, just without the LinkedIn context.

For enriched contacts only: also call Zevari `agents_research_person` to get recency signals (recent posts, recent activity, recent comments) — this is what turns "VP at a company" into "VP at a company who just posted about evaluating outbound tools last week."

---

## STEP 5 — ROUTING TABLE

For every classified thread, route per intent:

### BUYING_SIGNAL
Examples: "interested in a demo", "send me pricing", "what's your enterprise plan", "can we get on a call this week".

1. Draft a Gmail reply via `create_draft`. Use Zevari `agents_personalize` with the enriched LinkedIn context to tailor the draft (reference their role/company/recent signal in one line).
2. Apply Gmail label `triage/buying-signal`.
3. Slack ping `[INBOX_DIGEST_CHANNEL]` in real time with: sender name, sender title @ company, one-line subject context, draft preview link.
4. Save the contact to Zevari hot-leads list via `targets_save` + `targets_create_list` (list name: `inbox-buying-signals-YYYY-MM`).
5. Create a Pipedrive deal via `mcp__pipedrive__` `add_deal` — prefill with: contact name, organization, deal title (`"[Company] - inbound buying signal [date]"`), stage = "Inbound - Qualifying", notes with the LinkedIn-confirmed context.

### WARM_REPLY
Examples: someone replying to your outbound, replying to an intro you made, replying to a meeting follow-up.

1. Draft a contextual reply via `create_draft`, using `agents_personalize` to reference the thread history.
2. Apply Gmail label `triage/thread-needs-reply`.
3. Do NOT Slack ping individually — these show up in the digest.

### CUSTOMER_REQUEST
Examples: support ticket, billing question, feature request from an existing customer.

1. Apply Gmail label `triage/customer-request`.
2. Check the body for urgency keywords: `urgent`, `down`, `broken`, `not working`, `outage`, `ASAP`, `escalate`, `cancel`. If any match → Slack ping `[ALERT_CHANNEL]` immediately with sender + subject + one-line.
3. Do NOT draft a reply (support has their own flow). Just route + label.

### COLD_INBOUND
Examples: someone pitching you their SaaS, agency outreach, recruiter pitch, "do you have 15 min to chat about our X."

1. Apply Gmail label `cold-pitch`.
2. Archive (remove `inbox` label).
3. Optional auto-reply "not interested" — ONLY if `inbox_save_classifications` shows this sender has been classified `COLD_INBOUND` once before within the last 90 days. First-time pitches get archived silently to avoid training cold senders that this is a real inbox.

### NEWSLETTER
1. Apply label `auto/newsletter`.
2. Archive.

### NOTIFICATION
1. Apply label `auto/notification`.
2. Archive.

### SPAM
1. Apply label `auto/spam`.
2. Archive (do not mark "Report spam" — that's destructive; let Gmail's built-in filter handle confirmed spam).

---

## STEP 6 — LOG CLASSIFICATIONS

After all routing is done, call Zevari `inbox_classify_pending` to flush the in-memory classification batch, then `inbox_save_classifications` to persist them.

Payload per classification:
```
{
  thread_id,
  sender_email,
  intent,
  confidence,
  enriched: true/false,
  linkedin_url: optional,
  current_title: optional,
  current_company: optional,
  action_taken: "drafted" | "labeled-archived" | "needs-human" | "support-routed" | "deal-created",
  override: false  // set true if the user later manually re-classifies in Gmail
}
```

This is non-optional. Every classification — even obvious newsletters — needs to write back so Zevari learns your judgment over time. Skip this step and tomorrow's classifier is starting from zero.

---

## STEP 7 — SLACK DIGEST

Post the run summary to `[INBOX_DIGEST_CHANNEL]` via Chrome `javascript_tool`.

Format:

```
<@[YOUR_SLACK_USER_ID]> 📬 *Inbox Triage — 30-min Sweep*

🕒 Run: [HH:MM local]
📥 Unread processed: [N]

🟢 Buying signals: [N]
[for each: • [Sender Name] ([Title] @ [Company]) — [one-line subject context] — [draft ready]]

✉️ Warm replies awaiting attention: [N]
[for each: • [Sender Name] — [subject]]

🛟 Customer requests routed: [N] — [urgent: N if any]
🧊 Cold pitches archived: [N]
📰 Newsletters archived: [N]
🔔 Notifications archived: [N]
🚫 Spam: [N]
❓ Needs human (low confidence): [N]

LinkedIn enrichment rate: [N/M humans, %]
```

---

## STEP 8 — EOD WRAP-UP (6 PM ONLY)

If the current run is the last one of the day (>= 6:00 PM local), call Zevari `inbox_send_digest` with:

```
{
  to: "[FOUNDER_EMAIL]",
  summary_window: "today",
  highlight_buying_signals: true,
  highlight_warm_replies: true
}
```

This emails `[FOUNDER_EMAIL]` a "you have inbox zero, here's what mattered today" wrap-up — every buying signal of the day with the LinkedIn-confirmed context, every customer request that was routed, count of warm replies still waiting, total newsletters + cold pitches burned.

This is the "calm at the end of the day" deliverable. Do not skip it.

---

## STEP 9 — REFLECTION

- Any senders where `inbox_enrich_sender` failed but the sender clearly was a human → flag for manual LinkedIn URL backfill.
- Any classifications you'd disagree with → flag in Slack so they get re-classified next run (Zevari learns).
- Any urgency-keyword false positives in `CUSTOMER_REQUEST` → tune the keyword list.
- Total run time + classification volume → for capacity planning.

---

## Schedule

Every 30 minutes during waking hours (7:00 AM - 11:00 PM local). Outside those hours: pause (incoming threads still get classified next morning). EOD wrap-up only fires on the run that crosses 6:00 PM.
