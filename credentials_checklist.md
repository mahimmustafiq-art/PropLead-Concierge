# Credential Checklist (final). Never paste secrets in chat or commit them.

| # | Service | What | Configure in n8n as |
|---|---|---|---|
| 1 | n8n Cloud | Trial workspace | - |
| 2 | Supabase | Postgres connection details | Credential `Postgres account` (Postgres) |
| 3 | Groq | API key | `Groq API` (Header Auth: `Authorization` = `Bearer <key>`) |
| 4 | OpenRouter | API key (free models) | `Openrouter LLM Fallback` (Header Auth: `Authorization` = `Bearer <key>`) |
| 5 | Mistral AI | API key | `Mistral Cloud account` (Mistral Cloud API) |
| 6 | Telegram | Bot token | `Telegram PropLead` |
| 7 | Telegram | Chat ID (not secret) | `agency_config.settings.telegram_chat_id` and WF-C node `Alert Config` |
| 8 | Webhook auth | Random token 32+ chars | `Webhook Header Auth` (header name of your choice, e.g. `X-API-Key`) |

No payment method on any service. Gemini is not used.
Exported workflow JSON contains no keys (credentials are removed).
