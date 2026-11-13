# Setup Guide

This guide rebuilds PropLead Concierge from the files in this repository. It assumes you are a beginner. Everything uses free accounts. **Never paste real keys into any file in this repository or into a chat.** Enter them only inside n8n Credentials.

Estimated time: 60 to 90 minutes.

## What you need

| Service | Used for | Free option |
|---|---|---|
| n8n (Cloud trial or self-hosted) | Runs the four workflows | n8n Cloud trial (14 days, 1000 executions at the time of the original build) |
| Supabase | Postgres database and pgvector | Free project (pauses after about 7 days without activity) |
| Groq | Primary AI model | Free tier |
| OpenRouter | Backup AI model | Free models |
| Mistral AI | Embeddings | Free/experiment tier |
| Telegram | Alerts | Bot API is free |

Free quotas and model IDs change. Check current limits in each service console before you start. A model used in this project was retired during the build, so confirm the IDs in step 5.

## Step 1. Create the database

1. Create a free Supabase project.
2. Open **SQL Editor** and run `database/schema.sql` (safe to run more than once).
3. Run `database/seed_london.sql`. This creates the London agency, 20 properties and 17 knowledge-base documents.
4. Check: `select (select count(*) from properties) props, (select count(*) from kb_documents) kb;` should show 20 and 17.

## Step 2. Collect the credentials

Make a list using `.env.example` as the checklist. You need:

1. **Postgres connection details** from Supabase, Project Settings, Database (host, port, database, user, password).
2. **Groq API key**, **OpenRouter API key**, **Mistral API key** from each console.
3. **Telegram bot token**: in Telegram, talk to `@BotFather`, send `/newbot`, and copy the token.
4. **Telegram chat id**: send any message to your new bot, then open `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in a browser (replace `<YOUR_TOKEN>` yourself) and read the `chat` `id` number. Do not share that URL.
5. **A webhook token**: any random string of 32 or more characters.

## Step 3. Put the chat id in the database

Run in the SQL Editor (replace the number with your own chat id):

```sql
update public.agency_config
set settings = jsonb_set(settings, '{telegram_chat_id}', '"123456789"')
where is_active;
```

## Step 4. Create the n8n credentials

In n8n, open **Credentials** and create these, using exactly these names so the imported workflows can find them:

| Credential name | Type | What to enter |
|---|---|---|
| Postgres account | Postgres | Supabase connection details (enable SSL if required) |
| Groq API | Header Auth | Name `Authorization`, value `Bearer <your Groq key>` |
| Openrouter LLM Fallback | Header Auth | Name `Authorization`, value `Bearer <your OpenRouter key>` |
| Mistral Cloud account | Mistral Cloud API | Your Mistral key |
| Telegram PropLead | Telegram API | Your bot token |
| Webhook Header Auth | Header Auth | Name `X-API-Key` (or any header name you prefer), value = your webhook token |

## Step 5. Check model settings

The model IDs are stored in the database, not in the workflow. The seed sets:

- Groq: `openai/gpt-oss-120b`
- OpenRouter: `openrouter/free`
- Mistral embeddings: `mistral-embed` (1024 dimensions; the database column must match)

If a provider has retired a model, update `agency_config.settings.llm` in the database. Do not edit the workflow.

## Step 6. Import the workflows

1. In n8n, import the four files from `workflows/`: `WF-D_KB_Ingestion.json`, `WF-C_Global_Error_Handler.json`, `WF-A_Intake_Pipeline.json`, `WF-B_Daily_Digest.json`.
2. Open each workflow and check that every node that needs a credential shows the correct one. Select it manually if a node shows a warning.
3. In **WF-C**, open the node **Alert Config** and replace `REPLACE_WITH_TELEGRAM_CHAT_ID` with your chat id.
4. In **WF-A**, **WF-B** and **WF-D**, open Settings and set **Error workflow** to **WF-C Global Error Handler**.

## Step 7. Embed the knowledge base

1. Open **WF-D KB Ingestion** and click **Execute workflow** once.
2. Check in Supabase: `select count(*) filter (where embedding is not null) from kb_documents;` should equal 17.
3. Running WF-D again should report "no pending docs".

## Step 8. Activate

1. Activate (publish) **WF-C** and **WF-A**.
2. Leave **WF-B** inactive. You can run it manually with **Execute workflow**.
3. Open the Webhook node in WF-A and copy the **Production URL** (it contains `/webhook/`, not `/webhook-test/`).

## Step 9. Send a test enquiry

Replace the two placeholders (the header name must match step 4):

```bash
curl -i -X POST "YOUR_PRODUCTION_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_WEBHOOK_TOKEN" \
  -d '{"name":"Test Client","email":"test.client@example.com","phone":"+44 7700 900199","message":"I would like to view the 2 bed flat in Camden this week. Budget 2600 a month, I need to move by 1 December.","preferred_area":"Camden","budget":"2600/month","bedrooms":2,"source":"website"}'
```

Expected: HTTP 200 with `"status": "accepted"`, a category, `"duplicate": false` and a `reply`. A HOT result also sends a Telegram alert. Sending the same email again returns `"duplicate": true`.

## Step 10. Run the test suite (optional)

```bash
export WEBHOOK_URL="YOUR_PRODUCTION_WEBHOOK_URL"
export PROPLEAD_HEADER="X-API-Key"
export PROPLEAD_TOKEN="YOUR_WEBHOOK_TOKEN"
python3 tests/proplead_tests.py labeled      # 20 London leads
python3 tests/proplead_tests.py edge         # edge cases
```

Before running `labeled` again, delete earlier test leads with `database/reset_test_leads.sql`. If you do not, every lead is treated as a duplicate and the result will be wrong. The expected labels were tuned on these 20 leads, so a high score here is not an independent measure.

## Step 11. Try a second agency (optional)

1. Run `database/seed_manchester.sql`. It makes Manchester the only active agency.
2. If the model settings look wrong, run `database/fix_manchester_llm.sql`.
3. Run WF-D again to embed the Manchester documents.
4. Send a Manchester-style enquiry. No workflow edit is needed.
5. To return to London, run `database/seed_london.sql` and WF-D again.

## Self-hosting (not yet tested)

The workflows use standard n8n nodes. Importing them into a self-hosted n8n Community Edition should work, but this has not been tested. Re-create the credentials and re-check each node after import.

## Before you publish your own copy

Follow `SECURITY_NOTES.md`: scan for secrets, keep the chat id and instance URL out of files and screenshots, and keep all data synthetic.
