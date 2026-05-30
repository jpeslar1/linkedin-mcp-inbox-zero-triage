# Sample run

A real 30-minute sweep (sanitized — names and companies changed) from a recent Tuesday at 10:30 AM.

## Setup

- 14 unread threads in inbox
- All connectors healthy
- Last sweep was 30 minutes ago (so this is a fresh batch)

## Step 2 — domain pre-filter

3 threads auto-archived before reaching Zevari:
- `notifications@github.com` — PR review reminder → `auto/notification`
- `no-reply@stripe.com` — payout confirmation → `auto/transactional`
- `updates@linear.app` — sprint summary → `auto/notification`

11 threads survive to classification.

## Step 3 — Zevari classification (11 threads)

| # | Sender | Subject | Intent | Confidence |
|---|---|---|---|---|
| 1 | emily.r@northwind-health.com | "quick q on pricing" | BUYING_SIGNAL | 0.91 |
| 2 | marcus@finchcapital.vc | "intro to portfolio co - quick chat?" | BUYING_SIGNAL | 0.86 |
| 3 | sarah.k@bluepine-ops.io | "re: your note from last week" | WARM_REPLY | 0.94 |
| 4 | dwilliams@harborline-labs.com | "re: demo follow-up" | WARM_REPLY | 0.89 |
| 5 | priya@solstice-systems.com | "checking back in" | WARM_REPLY | 0.78 |
| 6 | support-ticket@customer-via-helpdesk | "URGENT: webhook failing on prod" | CUSTOMER_REQUEST | 0.96 |
| 7 | jaden.outbound@growth-rocket.io | "scale your outbound 10x" | COLD_INBOUND | 0.93 |
| 8 | hello@morningbrew.com | "the daily brief" | NEWSLETTER | 0.99 |
| 9 | digest@substack.com | "this week on stratechery" | NEWSLETTER | 0.97 |
| 10 | weekly@lennysnewsletter.com | "lenny's pod #142" | NEWSLETTER | 0.99 |
| 11 | newsletter@firstround.com | "first round review weekly" | NEWSLETTER | 0.98 |
| 12 | digest@theinformation.com | "the information daily" | NEWSLETTER | 0.99 |
| 13 | hi@producthunt.com | "today's top launches" | NEWSLETTER | 0.97 |
| 14 | win@cb-insights.com | "the briefing" | NEWSLETTER | 0.99 |

*(table shows 14 to keep numbering aligned with the original threads; the 3 pre-filtered ones were #8-10 in the original pull — re-sorted here for clarity)*

**Net:** 2 buying signals, 3 warm replies, 1 customer request (urgent), 1 cold pitch, 7 newsletters/notifications.

## Step 4 — Zevari enrichment (6 humans)

| Sender | Enrichment result |
|---|---|
| emily.r@northwind-health.com | ✅ VP RevOps @ Northwind Health (240 employees, healthcare SaaS, Series B). Posted about "evaluating outbound stack" 6 days ago. |
| marcus@finchcapital.vc | ✅ Partner @ Finch Capital. References a portfolio co (Verity Logistics, 80 employees, logistics SaaS). |
| sarah.k@bluepine-ops.io | ✅ Head of Ops @ Bluepine (Series A, RevOps tooling). Prior outbound thread confirmed. |
| dwilliams@harborline-labs.com | ✅ Founder @ Harborline Labs (12 employees, biotech tooling). Demo'd 4 days ago. |
| priya@solstice-systems.com | ✅ VP Sales @ Solstice Systems (Series C, 600 employees). Last touch 3 weeks ago. |
| jaden.outbound@growth-rocket.io | ✅ "Growth Specialist" @ Growth Rocket (8 employees, agency). Outreach role. |

Enrichment rate: 6/6 (100%) on the human senders. (Customer request #6 came in via the support helpdesk address — skipped enrichment, routed straight to support.)

## Step 5 — routing

### 🟢 BUYING_SIGNAL (2)

**Emily R. — Northwind Health, VP RevOps**
- Draft created in Gmail (referenced her "evaluating outbound stack" post from 6 days ago).
- Slack ping fired to `#inbox-triage`.
- Saved to Zevari list `inbox-buying-signals-2026-05`.
- Pipedrive deal created: `"Northwind Health - inbound buying signal 2026-05-26"`, stage `Inbound - Qualifying`.

**Marcus — Finch Capital, Partner (warm intro ask)**
- Draft created (acknowledged the portfolio company by name).
- Slack ping fired.
- Pipedrive deal created on Verity Logistics (the portfolio co), Marcus tagged as the source.

### ✉️ WARM_REPLY (3)

- Sarah K. → draft replied with contextual reference to last week's note. Label `triage/thread-needs-reply`.
- D. Williams → draft replied referencing the demo. Label `triage/thread-needs-reply`.
- Priya → draft replied with re-engagement nudge. Label `triage/thread-needs-reply`.

No real-time Slack pings — these surface in the digest.

### 🛟 CUSTOMER_REQUEST (1, urgent)

- Support ticket: "URGENT: webhook failing on prod" → label `triage/customer-request`, Slack ping fired to `[ALERT_CHANNEL]` because `URGENT` matched the urgency keyword list.

### 🧊 COLD_INBOUND (1)

- Jaden @ Growth Rocket → labeled `cold-pitch`, archived. First-time sender per `inbox_save_classifications`, so no auto-reply (silent archive).

### 📰 NEWSLETTER / 🔔 NOTIFICATION (7)

All labeled and archived: morningbrew, stratechery, lenny's, first round review, the information, product hunt, CB insights.

## Step 6 — classification log

All 14 threads written back via `inbox_save_classifications`. Two flagged for confidence-tuning review (#5 Priya WARM_REPLY at 0.78 was medium — draft made but no Slack ping per the confidence rule).

## Step 7 — Slack digest posted to `#inbox-triage`

```
📬 Inbox Triage — 30-min Sweep

🕒 Run: 10:30 local
📥 Unread processed: 14

🟢 Buying signals: 2
• Emily R. (VP RevOps @ Northwind Health, 240-person Series B healthcare) — "quick q on pricing" — draft ready
• Marcus (Partner @ Finch Capital) — intro ask for Verity Logistics (80-person logistics SaaS) — draft ready

✉️ Warm replies awaiting attention: 3
• Sarah K. — re: your note from last week
• D. Williams — re: demo follow-up
• Priya — checking back in (medium confidence — review)

🛟 Customer requests routed: 1 — urgent: 1
🧊 Cold pitches archived: 1
📰 Newsletters archived: 6
🔔 Notifications archived: 3
🚫 Spam: 0
❓ Needs human (low confidence): 0

LinkedIn enrichment rate: 6/6 humans (100%)
```

## Step 8 — EOD wrap-up

(Not this run — 10:30 AM. The 6 PM run fired `inbox_send_digest` separately and mailed the daily wrap to `[FOUNDER_EMAIL]`.)

## Step 9 — reflection notes

- Emily R.'s "evaluating outbound stack" post was the unlock. Without the LinkedIn enrichment surfacing that recency signal, the classifier might have called this a `WARM_REPLY` (she's already in our CRM from an old touch). The post-recency signal flipped it to `BUYING_SIGNAL` correctly.
- 7 of 14 emails were newsletters. The newsletter-burn is the single biggest perceived-quality win — getting them out of sight is most of the felt benefit before the buying-signal routing kicks in.
- Priya at medium confidence (0.78) was the right call — she did re-engage, but it's a soft check-in, not a buying signal. Drafted only, no ping. The confidence threshold is doing its job.
- Total run time: ~3 min.
