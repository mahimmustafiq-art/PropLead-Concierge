# Security Notes

This document explains how PropLead Concierge protects data and secrets, what was removed before publishing, and what is not protected. It describes a portfolio project that uses synthetic data only.

## 1. Data in this project

All demonstration data is synthetic: a fictional agency, invented people, `example.com` email addresses, reserved test phone numbers and invented properties. No real customer or client data was used. Do not send real personal data to a copy of this system.

## 2. Where secrets live

| Secret | Where it belongs | In this repository? |
|---|---|---|
| Webhook token | n8n credential `Webhook Header Auth` | No |
| Database password and connection details | n8n credential `Postgres account` | No |
| Groq, OpenRouter, Mistral keys | n8n credentials | No |
| Telegram bot token | n8n credential `Telegram PropLead` | No |
| Telegram chat id | Database setting and WF-C node `Alert Config` | **Placeholder only** (`REPLACE_WITH_TELEGRAM_CHAT_ID`) |
| Instance URL and webhook URL | Your own n8n account | No |

`.env.example` contains placeholders only. The workflows do not read `.env` at runtime.

## 2b. What was removed from the workflow exports

Credential blocks on every node, the stored chat id in WF-C, the error-workflow link, the instance id and version metadata. The workflows are exported inactive.

## 3. Controls implemented

| Area | Control |
|---|---|
| Access | Webhook protected by a header token; wrong or missing token is rejected by n8n |
| Database | Row-level security enabled on all six tables with no public policies; functions granted to the service role only; fixed search path; parameterised queries |
| Input | Validation of name, email, phone and length; spam rules; message truncated at 2000 characters |
| Prompt injection | Pattern detection; flagged leads skip all AI calls and use rules only; the AI sees the lead message only inside delimiters and is told never to follow instructions inside it |
| AI output | Strict JSON validation; scores clamped; replies limited to facts from the database and knowledge base; guard blocks invented numbers or references, promise words, viewing-promise phrases and long replies |
| Decisions | The AI never controls authentication, permissions or database access; SPAM and INVALID are decided by rules only |
| Privacy in logs | Run and error logs hold lead ids, never raw email, phone or message text |
| Alerts | Telegram HOT alerts contain score, area, budget, bedrooms and lead id only |
| Error handling | WF-C removes emails, phone-like numbers and bearer or key-like strings before alerting and logging; it cannot trigger itself |
| Duplicates | Advisory locks make duplicate handling safe under concurrent requests |
| Limits | HTTP timeouts on every external call; no loops; at most three AI or embedding calls per new lead |

## 4. What is NOT protected (be honest with users)

- The leads table stores the normalised email, phone, name and the latest message. There is no retention policy, consent flow or deletion tool.
- The lead name, message and matched property data are sent to Groq or OpenRouter, and the message text to Mistral. Free-tier providers may retain prompts.
- A single shared webhook token protects the endpoint. There is no per-client token, rate limiting or IP filtering, and no daily cap on AI calls, so a flood of unique requests could use up free quotas.
- Rejections for a wrong token or malformed JSON happen inside n8n and are not recorded in the run log.
- n8n and the database connection use a powerful database role. In a real deployment, use least-privilege roles.
- The system was tested on a trial environment, not hardened for production.

## 5. Pre-publication checklist

Run these checks on every folder before it goes online.

1. Search all files for key patterns and identifiers, for example:
```bash
grep -rIn -E "gsk_|sk-or-|Bearer [A-Za-z0-9]{12}|eyJ[A-Za-z0-9]{20}|bot[0-9]{6,}:|\"credentials\"|instanceId|supabase\.co|webhook/[a-z0-9-]+" .
```
2. Confirm WF-C `Alert Config` still shows the placeholder, not your chat id.
3. Confirm no file contains your n8n instance name, webhook URL or Supabase project reference.
4. Check screenshots: crop or blur browser address bars (they show the instance name), emails, project ids, tokens and Authorization headers. Never open the n8n Credentials page while recording or capturing.
5. Confirm all emails are `example.com` or other reserved example domains and all phone numbers are test numbers.
6. Do not publish files from old working folders. Use only the clean repository folder.

## 6. If a secret leaks

1. Revoke or rotate it at once in the provider console (Groq, OpenRouter, Mistral, Telegram via BotFather, Supabase database password).
2. Change the webhook token and update the n8n credential and your test environment.
3. Remove the secret from the repository history, not only the latest commit.
4. Review recent executions and logs for unexpected activity.

## 7. Reporting an issue

If you find a security problem in this project, contact the author, Khalid Mostafiq Mahim, through the platform where you found the project. Do not post secrets or exploit details publicly.
