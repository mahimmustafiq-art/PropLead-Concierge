# Cost Estimate
Built entirely on free tiers; no payment method was added. Free quotas and limits were not re-verified in each console during this project; check current limits before relying on them.
| Item | Used for | Cost during project |
|---|---|---|
| n8n Cloud trial (14 days, 1000 executions) | Orchestration | 0 |
| Supabase free project | Postgres + pgvector | 0 (pauses after ~7 days idle; WF-B keep-alive) |
| Groq free tier | Primary LLM | 0 |
| OpenRouter free models | Fallback LLM | 0 |
| Mistral free/experiment tier | Embeddings | 0 |
| Telegram Bot API | Alerts | 0 |
Per new lead: 2 external AI calls normally (embedding + Groq), 3 on fallback.
Production path: import the four JSON files into self-hosted n8n Community Edition (server cost only), keep Supabase, and replace free LLM tiers with paid ones if volume or reliability requires. Prices are not quoted here because they were not verified.
