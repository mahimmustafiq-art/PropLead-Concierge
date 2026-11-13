# Troubleshooting Guide

Find your symptom, check the likely cause, apply the fix. Most checks use the SQL Editor in Supabase or the **Executions** tab in n8n. Never paste real keys or tokens into chats, tickets or screenshots.

## 1. Webhook responses

| Symptom | Likely cause | Fix |
|---|---|---|
| HTTP **403** `Authorization data is wrong!` | Wrong or missing token, or the header name does not match the credential | Use the exact header name and value from the `Webhook Header Auth` credential. n8n answers this before the workflow starts, so there is no log row. |
| HTTP **404** `webhook ... is not registered` | WF-A is not published, or you used the test URL | Publish WF-A and use the **Production URL** (contains `/webhook/`, not `/webhook-test/`). Re-publish after changing credentials. |
| HTTP **422** `Failed to parse request body` | The request body is not valid JSON | Fix the JSON. n8n rejects it before the workflow starts, so there is no log row. |
| HTTP **422** `status: rejected` with `SPAM` or `INVALID` | The enquiry failed the rules (too many links, spam phrase, no valid email or phone, name or message length) | Expected behaviour. The `reason` field tells you which rule fired. |
| HTTP **503** `service_unavailable` | The database could not be reached when loading configuration or saving the lead | Check the Supabase project is not paused and the `Postgres account` credential is correct. WF-C should have sent a Telegram alert. |
| Very slow first request (about 7 to 12 seconds in testing) | A cold start after idle time | Send one warm-up request. Later requests usually took about 1.5 to 6 seconds. |
| One request took about 40 seconds | Both AI providers produced no usable output and the pipeline waited for timeouts before using rules-only scoring | The lead is still saved and scored. Check `ai_status` and `llm_provider` in the database. |

## 2. Test results look wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Labeled test shows `duplicate: True` for every lead and 0/20 | Earlier test leads were not deleted | Run `database/reset_test_leads.sql`, confirm `leads_left` is 0, then re-run. Discard the invalid run. |
| First Manchester run answered with London data | The Manchester seed was not applied, so London was still active | Run `database/seed_manchester.sql`, then `select agency_name, is_active from public.agency_config;` |
| Manchester run fails with a Groq 404 | An old model ID in the agency settings | Run `database/fix_manchester_llm.sql` or update `settings.llm` |
| Spanish enquiry scores WARM, not HOT | Timeline rules recognise English words only | Known limitation |

## 3. AI and knowledge base

| Symptom | Likely cause | Fix |
|---|---|---|
| `ai_status = UNAVAILABLE`, `llm_provider = none` for a normal lead | Both AI providers failed, or the lead was flagged as a prompt injection (AI is skipped on purpose) | Check `injection_suspected`. If false, check the Groq and OpenRouter credentials, quotas and model IDs. The lead still received a rules-only score. |
| Groq node returns 404 or "model not found" | The model ID was retired | Update `agency_config.settings.llm.groq_model`. Check the provider's current model list. |
| OpenRouter returns empty content | A reasoning model used the whole token limit | Already mitigated (larger limit and timeout). Free models are unstable, so retry later or pin a specific free model ID. |
| `rag_status = UNAVAILABLE` | The embedding call failed or the dimension is not 1024 | Check the Mistral credential and quota. The model must return 1024 values to match the database column. |
| Search finds nothing useful | Knowledge base not embedded or similarity threshold too strict | Run WF-D, then check `select count(*) filter (where embedding is not null) from kb_documents;` |
| Reply is the generic template although the AI worked | The reply guard rejected the AI text (unknown number, promise phrase, over 120 words) | Expected safety behaviour. Check `score_reason` for `reply_guard_rejected`. |

## 4. Telegram

| Symptom | Likely cause | Fix |
|---|---|---|
| No HOT alert arrived | The alert node is set to continue on error, so a green node does not prove delivery | Check `select left(id::text,8), hot_alerted_at from public.leads where category = 'HOT' order by created_at desc limit 5;` and the latest `error_log` rows. |
| Error "can't parse entities" | Special characters in Telegram HTML mode | Already fixed by escaping. If you edit the alert text, escape `&`, `<` and `>`. |
| Error alerts do not arrive | Chat id still the placeholder in WF-C, or the bot has no permission to message you | Set the real chat id in WF-C `Alert Config`, and send your bot a message first. |
| Telegram does not show a new alert | Telegram Web needs a refresh | Reload the tab and scroll to the bottom. |

## 5. Platform limits

| Symptom | Likely cause | Fix |
|---|---|---|
| Supabase queries fail after a quiet week | The free project paused | Resume it in the dashboard. WF-B includes a keep-alive query, but it only runs if WF-B runs. |
| n8n workflows stop running | The n8n Cloud trial ended or the execution quota is used | Export the workflows, or move to self-hosted n8n. |
| Webhook works but no error alerts | Error workflow is not set | In WF-A, WF-B and WF-D settings, select WF-C as the **Error workflow**. |
| Digest not sent at 08:00 | WF-B schedule is inactive by design | Activate it, or run it manually. The hour is fixed at 08:00 in the workflow. |

## 6. Questions people ask

**Why did the lead not receive the reply by email?** The system does not send messages to leads. The draft reply is returned in the webhook response for the calling system to use.

**Why does a lead with a clear need stay COLD?** Out-of-area leads and leads with no matching available property are capped at COLD by design.

## 7. Useful SQL (also in `database/evidence_queries.sql`)

```sql
-- Latest leads and how they were scored
select left(id::text,8) id, category, score_rules, score_llm, score_final, ai_status, llm_provider, rag_status
from public.leads order by created_at desc limit 20;

-- Latest runs and AI calls used (must be 3 or fewer for new leads)
select created_at, path, category, http_status, ai_status, calls_used, duration_ms
from public.run_log order by created_at desc limit 20;

-- Recent errors
select created_at, workflow_name, node_name, failure_stage, notified
from public.error_log order by created_at desc limit 10;

-- Which agency is active
select agency_name, is_active from public.agency_config;
```
