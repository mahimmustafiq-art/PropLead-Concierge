# Fixing Note
| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | Telegram HOT alert failed | HTML parse mode, unescaped text | Escape text (earlier chat) |
| 2 | Labeled run 1 only 80% | LLM scored generously; no deterministic limit | Caps: no urgency -> not HOT; no property match -> not WARM |
| 3 | Fallback test returned UNAVAILABLE | OpenRouter model wrote key `" seriousness"` with a space | Trim JSON key names in both parsers (values still strictly validated) |
| 4 | Fallback returned empty content | Reasoning model spent all 1200 tokens thinking | max_tokens 3000, OpenRouter timeout 40 s |
| 5 | Manchester slow, Groq 404 | `seed_manchester.sql` carried old model ID `llama-3.3-70b-versatile` | `fix_manchester_llm.sql`; seeds corrected to `openai/gpt-oss-120b` |
| 6 | First Manchester run used London data | Seed had not run | Run discarded, redone |
| 7 | M03 took 145 s once | Not explained (slow OpenRouter suspected, unconfirmed) | Not reproduced after Groq fix |
| 8 | Review: a reply could promise a viewing (AC4) | Guard only blocked numbers, refs and promise words | v1.1: guard blocks "I/we will/can/shall arrange, book, schedule, confirm, reserve, hold, secure" and "viewing is booked/confirmed"; prompt forbids viewing promises |
| 9 | LLM wrote property refs with a non-ASCII hyphen (LON‑R003), invisible to the ref check | Guard and audit SQL matched only the ASCII hyphen | Score and Guard converts Unicode hyphens/dashes to "-" before the guard and before storing the reply |
| 10 | Review: curly apostrophe (I’ll arrange) bypassed the promise guard | Regex matched only the ASCII apostrophe | Score and Guard also converts ’ ‘ and ʼ to "'"; audit SQL (query 6) normalises hyphens and apostrophes. Re-test evidence: see test_results.md |

Deviations kept: wrong token gives 403 and malformed JSON gives 422 (both returned by n8n itself, not written to run_log); OpenRouter acts as the retry (3-call cap); WF-B schedule is a fixed 08:00; WF-C holds the Telegram chat id so alerts work without Supabase.

Known limits: English-only timeline rules; caps tuned on the same 20 leads; the reply guard blocks first-person viewing promises but not third-person ones such as "our agent will book a viewing" (agents do confirm times, so this is a business decision) and cannot catch every phrasing; silent continue-on-error on Match Properties and Update Lead; digest fixed at 08:00; OpenRouter free models are unstable.
