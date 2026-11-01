# the company WhatsApp Automation - System Overview for Review

**Prepared for:** Automation team review
**Date:** 2026-10-26
**Scope:** The four connected systems behind the company's WhatsApp task bot and reporting pipeline - the WhatsApp Bot, the Cumulative Sales Report generator, the Missing Names Resolver, and the Management Tracker chatbot. (A few other workflows exist in the n8n instance - a one-time setup script, a scheduled reminder/digest job, a weekly cleanup job, and an earlier alternate report bot - but those are outside this overview's scope since they're either support utilities or superseded.)

Exported workflow JSON files for all four are included alongside this document. Note: the exports carry node logic and configuration only - credentials (WhatsApp/Graph API tokens, SMTP login, ClickUp API key) are stored separately in n8n and are **not** in the files. To actually run any of these, credentials need to be reconnected on your end.

---

## 1. How the pieces fit together

**WhatsApp Official API - Webhook** is the hub. It's the single entry point for every incoming WhatsApp message (via Meta's Graph API webhook). Depending on what the user asks for or where they are in a conversation flow, it:
- Parses intent and creates/updates tasks in ClickUp (the core "task bot" behavior - assigning work to Alex, Sam, etc.)
- Tracks lightweight conversation/session state so multi-step replies (like "yes/no" or filling in a due date) make sense
- Looks up the sender's registered contact record and, if they're asking for the cumulative sales report, finds the latest generated report file and delivers it - both as a WhatsApp document attachment and via email
- Sends a generic acknowledgment message back over the Graph API for anything it doesn't have a specific flow for

**Cumulative Sales Report** generates the actual report file on its own schedule - it's the producer. It pulls sales data and writes a dated report (Excel) to a shared folder. The webhook bot above is the consumer: when someone messages in asking for "the report," it just looks in that folder for the newest file and sends it - it doesn't generate anything itself.

**Missing Names Resolver** is the contact-registry safety valve. The webhook bot only knows how to serve a report (or route a task) to someone if their WhatsApp number is matched to a known contact record. When a number comes in that isn't registered, the bot's lookup returns a "not registered" outcome instead of failing silently. The Resolver is a small set of web endpoints (a resolver page, a "get current state" endpoint, and an "action" endpoint) that lets someone manually reconcile unmatched numbers against names/contacts, rather than that being a dead end.

**Management Tracker (chatbot)** is a separate, management-facing conversational flow - effectively a second bot aimed at tracking/summarizing status for management rather than individual task assignees. It's built as an AI-agent-style flow rather than the simpler rule-based intent parsing in the main webhook bot.

In short: the webhook bot is the always-on front door; the Sales Report workflow feeds it data on a schedule; the Names Resolver patches gaps in who the front door recognizes; and the Management Tracker is a parallel, higher-level view for leadership rather than a dependency of the other three.

---

## 2. Status: what's live vs. in progress

| Workflow | Status | Notes |
|---|---|---|
| WhatsApp Official API - Webhook | **Live** | Core bot in daily use. One fix is built and saved but not yet published - see Bugs below. |
| CUMULATIVE SALES REPORT | **Live** | Scheduled report generation is working. |
| MISSING NAMES RESOLVER | **Live** | Deployed; low-traffic utility, used as needed rather than constantly. |
| MANAGEMENT TRACKER CHATBOT | **In progress** | Several nodes in this workflow are explicitly marked "not configured yet." Should not be treated as production-ready - worth a closer look before relying on it. |

---

## 3. Bugs found this review cycle

**Cumulative report delivery was unreliable ("takes forever to come through" / sometimes never arrives) - root cause found, fix built, not yet live.**

The webhook bot's report-delivery branch runs two steps in parallel after a report request is recognized: sending the report as a WhatsApp document, and emailing a copy. The email step ("Send Report Email") had a broken expression for the recipient address - it referenced `.item.json.email` instead of `.first().json.email`, which resolved to empty under certain item-linking conditions, causing a "No recipients defined" error.

The real issue wasn't just that one step failing - it's that n8n's default "On Error" behavior is **Stop Workflow**, which aborts the *entire* execution immediately on any unhandled node error, including sibling branches that are otherwise unrelated and haven't even started yet. So when the email step failed, it was killing the WhatsApp document delivery in the same breath, even though that half of the flow had nothing wrong with it. This explains the "sometimes it just never shows up" symptom - it wasn't slow, it was failing silently downstream of a parallel error.

**Fix (built and saved, not yet published):**
1. Corrected the recipient expression to `{{ $('Lookup Report & Contact').first().json.email }}`
2. Changed "Send Report Email"'s On Error setting from Stop Workflow to Continue, so an email failure no longer takes down the WhatsApp delivery

This needs one more "Publish" click on the live workflow to go into effect - flagging it here so the team is aware it's an active, confirmed-but-not-deployed fix rather than something already shipped.

**One correction worth flagging for transparency:** earlier in this project, a diagnosis mistakenly concluded the report lookup itself was broken (based on a flawed test), and a change was shipped that temporarily disconnected a working feature. That was caught via user feedback, root-caused properly, and fully reverted/republished before the actual fix above was developed. Mentioning it so the review isn't surprised by version history showing a revert.

---

## 4. Notes for reviewers

- The **contact registry** (phone-number-to-name/email mapping) that the webhook bot's lookup depends on is read from a flat JSON config file on the host machine, not a database - worth discussing whether that should move to something more robust as usage grows.
- **Management Tracker Chatbot** uses an AI-agent node pattern, which is architecturally different from the rule-based parsing in the main bot - if the team ends up maintaining both, it's worth deciding whether to standardize on one approach.
- All four workflows currently run on a single self-hosted n8n instance on a Windows machine - there's no redundancy/failover, which may be worth a conversation depending on how business-critical this becomes.
