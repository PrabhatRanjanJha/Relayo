# ⚡ Relayo

**An AI automation system that runs a messaging startup's inbox, billing, and social channels — so a human doesn't have to babysit all three.**

Built in n8n. Four independent workflows. Every risky decision passes through a human before it goes out.

---

## 🧩 Why This Exists

Modeled on a real AI Automation Engineer job posting for an early-stage messaging startup. The ask: a multi-agent system automating email response, billing/invoice monitoring, social media interactions, and routine admin work — with human-in-the-loop oversight, not blind automation.

This is my answer to that brief, built end-to-end as a portfolio project.

## 🚀 What It Does

### 📧 Email Response Handler
Reads every incoming email, figures out what it is, drafts a reply, and decides for itself whether that reply is safe to send or needs a human's eyes first.
- Classifies: support / billing / partnership / spam
- Auto-sends low-risk replies, flags the rest for review
- **Spam gets silently discarded** — no reply, no wasted human time
- Hardened against prompt injection hidden inside email content
- Won't double-process the same email twice

### 💰 Billing & Invoice Monitor
- Generates and logs invoices the moment a payment comes in
- Sweeps for overdue customers on a schedule
- **Never auto-sends a payment dispute reply** — money issues always go to a human first

### 💬 Discord Message Handler
Stands in for LinkedIn/Meta (their APIs aren't open to individual devs). Same classify → draft → decide logic, applied to a live chat channel.
- Polls for new messages (n8n has no native Discord trigger — built a custom polling + state-tracking solution)
- Auto-replies to routine messages
- Sends an instant "we've seen this" acknowledgment on flagged ones, because silence in a chat reads as being ignored

### 🩺 Stale-Flag Monitor + Service Health Check
The system watches itself.
- Flags any human review sitting untouched for 3+ days, broken down by source
- Pings Groq, Gmail, Sheets, and Discord on a schedule and raises an alert the moment one goes down

## 🛠️ Tech Stack

`n8n` · `Groq` — `openai/gpt-oss-120b` (Email Handler, Health Check) & `qwen/qwen3-32b` (Discord Handler) · `Google Sheets` · `Gmail` · `Discord` · `Slack`

## 🏗️ Architecture

Every workflow follows the same backbone:
classify → draft → decide → auto-send OR flag for human review

Flagged items never sit inside a paused, resource-holding execution. They're logged to a sheet, and a **separate, decoupled workflow** picks them up once a human approves — because a human might take minutes or might take hours, and nothing should be stuck waiting on them.

1. **Billing & Invoice Monitor** — Payment-Received → Overdue-Sweep → Send-Warning (decoupled)
2. **Email Response Handler** — Intake & Classification → Send Reviewed Replies (decoupled)
3. **Discord Message Handler** — Intake & Auto-Response → Send Reviewed Discord Replies (decoupled)
4. **Stale-Flag Monitor + Service Health Check** — two independent scheduled watchdogs

## 🔍 How It Works

*Using Email Response Handler as a representative example — Billing, Discord, and the Stale-Flag/Health Check workflows follow the same classify → decide → act-or-flag backbone, just with different triggers.*

```
New email arrives
   │
   ├─ Already processed before? ── Yes ──► stop (no duplicate replies)
   │
   No
   │
Log as processed → classify + draft + decide (one LLM call, strict JSON out)
   │
   ├─ Spam? ── Yes ──► discard silently
   │
   No
   │
   ├─ Flagged? ── Yes ──► review sheet → human edits/approves → sent by a separate scheduled workflow
   │
   No ──► reply sent immediately
```

## 🧠 Key Design Decisions

- **Invoices are templated, never LLM-generated.** Exact amounts shouldn't come from something non-deterministic.
- **Review and send are always two separate workflows.** A paused execution waiting on a human is a wasted resource sitting idle for however long they take.
- **Every classification prompt carries explicit phishing/scam indicators and prompt-injection defenses.** An LLM reading untrusted inbound content has to treat it as data, never as instructions — tested against real injection attempts during development.
- **Billing disputes are *always* flagged**, even though it grows the review queue. An auto-sent "I'll investigate" with no actual investigation happening is a broken promise sitting in a customer's inbox — worse than one more item for a human to glance at.
- **Discord polling, not webhooks.** n8n has no Discord trigger node, and Discord doesn't push events to arbitrary webhooks — so this uses scheduled polling against Discord's REST API, with the last-seen message ID held in n8n's workflow static data (no external sheet needed for one small value).

## ⚙️ Setup

### Prerequisites
- n8n (self-hosted)
- Groq API key — free at [console.groq.com](https://console.groq.com)
- Google account with Sheets + Gmail access
- A Discord bot token ([developer.discord.com](https://discord.com/developers)) with Message Content Intent enabled
- A Slack workspace with a bot token

### Steps
1. Clone this repository
2. Import all four workflow JSON files into n8n
3. Set up credentials: Groq API key, Google Sheets OAuth, Gmail OAuth, Discord bot token, Slack bot token
4. Create the sheets below and point each workflow's Sheets nodes at them
5. Drop in your Discord channel ID and bot token
6. Activate everything

## 📊 Sheets

| Sheet | Purpose |
|---|---|
| **Billing Log** | Customer ID, Name, Email, Plan, Amount, Currency, Payment Date, Due Date, Transaction ID, Invoice ID, Status |
| **Flagged for Review (Billing)** | Overdue customers awaiting human sign-off, with a Flagged Date for staleness tracking |
| **Processed Emails** | Gmail message IDs — prevents double-processing the same email |
| **Flagged Emails for Review** | ID, Email, Original Message, LLM Draft, Classification, Human Draft, Allowed, Status, Flagged Date |
| **Flagged Discord Messages** | Same review pattern, applied to Discord |

## 🙋 About

A self-directed project, built to prove I can design and ship a complete AI automation system — not a demo, but something that actually handles real inboxes, real money, and real conversations, with a human always in the loop where it counts.
