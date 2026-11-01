# WhatsApp Task & Reporting Automation (n8n)

A set of n8n workflows I built to run a small company's day-to-day task tracking and sales reporting over WhatsApp. The thinking was simple: the team already lives in WhatsApp, so rather than push them into yet another tool, the bot lets them create tasks, get reminders, and pull the daily sales report by just sending a message.

This repo has the sanitized workflow exports plus notes on how the pieces fit together. Every credential, token, phone number, email and internal file path has been stripped out and replaced with placeholders, so nothing here runs as-is. You would reconnect your own WhatsApp, ClickUp and SMTP credentials to use it.

## What it does

- **WhatsApp webhook bot** (`whatsapp-webhook-bot.json`) is the front door. Every incoming WhatsApp message hits this first. It figures out what the person wants, creates or updates tasks in ClickUp, keeps a little conversation state so multi-step replies make sense, and serves the latest sales report when someone asks for it.
- **Cumulative sales report** (`cumulative-sales-report.json`) runs on a schedule. It picks up the day's sales files, sorts them into the right buckets, runs a Python builder to produce a dated Excel report, and emails it out. If any customer names or account managers can't be matched automatically, it flags them for a human instead of guessing.
- **Missing names resolver** (`missing-names-resolver.json`) is a small self-hosted web page (three webhook endpoints) for reconciling unmatched customer names against the registry, so those cases don't just fail silently.
- **Management tracker chatbot** (`management-tracker-chatbot.json`) is a separate AI-agent-style bot aimed at managers, for updating status in plain language. Still a work in progress, so a few tool nodes are left unconfigured on purpose.
- **Supporting jobs**: a morning digest of what's due or overdue, scheduled reminders with "done / not yet" buttons, a weekly purge of completed tasks, and a one-off setup script that creates the Excel file the task bot writes to.

## A bug that was worth writing down

Report delivery used to be flaky. People said it "takes forever" or just never showed up. It turned out the email step had a bad recipient expression, and because n8n stops the whole workflow on any unhandled node error by default, that one failing email was taking down the WhatsApp file delivery running right next to it. So it wasn't slow, it was failing quietly downstream.

The fix was two parts: correct the recipient lookup, and set that email node to continue on error so a mail problem can no longer kill the WhatsApp side.

## Stack

n8n, WhatsApp (Meta Cloud API and WAHA), ClickUp API, Python for the report building, SMTP for email, and Microsoft Graph / Excel for the tracker. It all runs on a single self-hosted n8n instance.

## Notes

- Client details are anonymized. This is a portfolio copy, not the production export.
- `system-overview.md` has the fuller write-up of how the four main workflows connect.
- `.env.example` lists the credentials you would wire up to run it yourself.
