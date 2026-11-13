# PropLead Concierge: Master Case Study

**Project:** AI lead qualification, property matching and escalation for a real-estate agency
**Author:** Khalid Mostafiq Mahim, AI automation developer
**Status:** Self-initiated portfolio project, built and tested on an n8n Cloud trial. All data is synthetic.

---

## 1. Honest framing

PropLead Concierge is a project I designed and built myself to demonstrate AI workflow automation. There was no client, no budget and no production deployment. The agency (Harbor & Pine Realty), its properties, its policy documents and every test lead are fictional. The system ran on an n8n Cloud trial and free tiers of the supporting services. Every result in this document comes from my own tests, and each number is shown with its limits.

## 2. The business problem

For a small property agency, an enquiry is perishable. Someone who needs to move in two weeks and asks to view a flat tomorrow will usually go with whichever agent answers first. In a manual process this goes wrong in predictable ways:

- Urgent leads wait in the same queue as people who are only browsing.
- Duplicate enquiries from the same person create confusion.
- Replies written quickly can quote a price or a listing that is no longer available.
- Nobody notices which warm leads have gone quiet.

## 3. Objectives

1. Qualify every enquiry in seconds and put urgent ones in front of a human immediately.
2. Recommend only properties that are currently available.
3. Draft replies that use only facts from the database and the agency's own policy documents.
4. Keep the AI from deciding things it should not decide (security, routing rules, permissions).
5. Fail safely: if a service is down, still save the lead and tell a human.
6. Make the system work for another agency or city by changing data, not workflows.

## 4. The solution at a glance

An enquiry arrives at a token-protected webhook. The system cleans and checks it, finds or updates the lead, matches available properties, retrieves relevant policy text, asks an AI model for a score and a draft reply, combines that with transparent business rules, checks the reply, stores everything, and, if the lead is HOT, sends a Telegram alert that contains no personal contact data.

| Category | Rule | What happens |
|---|---|---|
| HOT | final score 70 or more | Instant Telegram alert |
| WARM | 40 to 69 | Included in the daily digest |
| COLD | below 40 | Saved and logged |
| SPAM / INVALID | deterministic rules only | Rejected with HTTP 422 and logged |

## 5. How one enquiry flows (WF-A Intake Pipeline, 34 nodes)

1. **Webhook Intake** checks a secret header token. A wrong or missing token is rejected by n8n before the workflow starts.
2. **Load Agency Config** reads the active agency from the database: name, city, currency, service areas, hours, thresholds and model IDs. Nothing agency-specific is hard-coded in the workflow.
3. **Validate and Normalize** trims and checks name, message length (2000 characters maximum), email and phone, flags spam indicators (too many links, repeated characters, known spam phrases, all-capitals text) and prompt-injection patterns, parses the budget and bedrooms, and checks the area against the configured service areas.
4. **Upsert Lead** performs an atomic duplicate check by email or phone. A duplicate updates the existing lead and returns immediately with zero AI calls.
5. **Match Properties** returns up to three properties with status "available", ranked by bedroom fit and price fit.
6. **Embed Query Mistral** and **Search KB** turn the enquiry into a 1024-dimension embedding and fetch the most relevant policy documents from Supabase pgvector. If this fails, the pipeline continues without retrieval and records it.
7. **Groq Chat** produces one JSON object: intent, urgency and seriousness scores, extracted details, a spam flag and a draft reply. **OpenRouter Chat** is the only backup. Both outputs are validated strictly before use.
8. **Score and Guard** combines the rules score with the AI score, applies caps, and checks the reply (see section 6).
9. **Update Lead**, **Respond Accepted**, **Send HOT Alert** and **Log New Run** store the result, answer the caller, alert on HOT, and write a log row that holds no raw email, phone or message.

Injection-flagged leads skip all AI calls. Out-of-area leads get a polite template reply and are capped at COLD.

## 6. Key design decisions

**Hybrid scoring.** Rules contribute up to 60 points (budget fit, time signals, contact completeness, specificity, area and property match). The AI contributes up to 40 (intent 20, urgency 10, seriousness 10). Scores are validated and clamped. The AI never decides alone, and SPAM or INVALID is decided only by deterministic rules.

**Deterministic caps.** After my first labeled run the AI scored some leads too generously. I added two caps: a lead without any urgency or viewing signal cannot be HOT, and a lead with no matching available property cannot be WARM or HOT. Out-of-area leads are COLD.

**Grounded replies with a guard.** The AI sees only the delimited lead message, the matched properties and the retrieved policy text. A guard then rejects any reply that contains a number or property reference that is not in the allowed facts, promise words, viewing-promise phrases such as "I'll arrange a viewing", or more than 120 words, and substitutes a template reply.

**Call budget of three.** A new lead can use at most one embedding call, one Groq call and one OpenRouter call. The OpenRouter call doubles as the retry. Duplicates and injection-flagged leads use none.

**Failure behaviour.** Groq failure goes to OpenRouter. Both failing gives a rules-only score and `ai_status = UNAVAILABLE`. If the database is unavailable the webhook returns 503 and a separate error workflow alerts me on Telegram.

**Location-agnostic design.** A second agency (Manchester) was added through database rows only: agency configuration, properties and knowledge-base documents.

## 7. What I built

I designed the architecture and wrote everything in the repository: the database schema and functions, the synthetic data sets and knowledge-base documents, four n8n workflows, the scoring rules, the reply guard, the test runner, the labeled test set and the documentation.

| Component | Detail |
|---|---|
| Workflows | WF-A (34 nodes), WF-B (20), WF-C (6), WF-D (13) |
| Database | Supabase Postgres with pgvector; tables agency_config, properties, leads, kb_documents, run_log, error_log; row-level security enabled with no public policies |
| Knowledge base | 17 London documents and 7 Manchester documents |
| Properties | 20 London and 13 Manchester synthetic listings (some deliberately let, sold or withdrawn) |
| Models | Groq `openai/gpt-oss-120b`, OpenRouter `openrouter/free`, Mistral `mistral-embed` |
| Notifications | Telegram Bot API |

## 8. Testing and results

All results are from real runs on the n8n Cloud trial (2 to 5 November 2026). Full evidence: `docs/test_results.md`.

**Labeled set (7 HOT, 7 WARM, 6 COLD).** Run 1 scored 16/20 (80%), which missed the 85% target: three WARM leads came out HOT and one COLD lead came out WARM. After the two caps, Runs 2, 3 and 4 each scored 20/20. **Caveat:** the caps were designed after seeing Run 1 on the same 20 leads, so 20/20 is a tuned result and not an independent test. Run 4 is the final run (all 20 leads received new lead ids, none was treated as a duplicate).

**HOT alert speed (Run 4).** For the seven HOT leads, the alert was recorded 1.5 to 5.9 seconds after the lead was saved (slowest 5.9 s, cause not recorded). This measures database insert to recorded Telegram success. It does not include webhook authentication or delivery to a phone.

**AI usage.** In Run 4 the seven HOT leads each scored 92 or 95 (rules 58 to 60 plus AI 34 or 35), used Groq, retrieved knowledge-base text, and used two calls. Across all runs the maximum was three calls, only when Groq failed.

**Reply safety (Run 4).** A SQL audit found all 20 stored replies passed the property-reference and promise checks (18 written by the AI, 2 safe templates). I also read the seven HOT replies by hand: none promised or confirmed a viewing, and prices and dates matched the database. The audit uses the same patterns as the guard, so it is an automated check rather than a human review of all 20.

**Edge cases.** Duplicate enquiry, phone-only enquiry, very long message (stored as 2000 characters), Spanish enquiry (Spanish reply, but scored WARM because timeline rules recognise English only), prompt injection (COLD, AI skipped), no matching property, out-of-area request, spam and invalid input all behaved as expected. A wrong or missing token returned 403 and malformed JSON returned 422; both are produced by n8n itself, so they do not appear in the run log.

**Failure tests.** With a deliberately invalid database credential, the webhook returned 503 and the error workflow sent a Telegram alert and wrote an error log row (tested on version 1.0). With a deliberately wrong Groq key, the OpenRouter fallback produced a FALLBACK result using three calls (version 1.0). In Run 4 one lead naturally hit a case where both AI providers produced no usable output; it still returned 200 with a rules-only COLD result and a template reply after about 40 seconds. The cause of that failure was not recorded.

**Portability.** Three Manchester leads (HOT, WARM, COLD as expected) used Manchester hours, listings and areas with no workflow change (tested on version 1.0).

## 9. Problems found and fixed (honest log)

| Problem | Fix |
|---|---|
| Telegram alert failed on special characters | Switched to HTML mode and escaped text |
| Run 1 only 80% | Two deterministic caps |
| Fallback model returned a JSON key with a leading space | Trim key names; values still validated |
| Fallback model returned empty output | Larger token limit and longer timeout |
| Manchester data used a retired model ID | Corrected seeds and a config fix script |
| Review: a reply could promise a viewing | Guard and prompt now block viewing promises |
| Review: AI wrote references with a non-ASCII hyphen | Hyphens and dashes normalised before the guard |
| Review: a curly apostrophe bypassed the promise check | Curly apostrophes normalised, audit query updated |

## 10. Business value (design intent, not measured)

I did not measure any business outcome, because there was no live business. The intended benefits are structural: urgent leads are surfaced immediately with a reason, replies stay within known facts, duplicate enquiries merge, stale leads are listed in a daily digest, and a failure produces an alert instead of silence. Any claim about time saved, conversion or revenue would be invented, so this document makes none.

## 11. Limitations

- Synthetic data, one trial environment, no production use.
- The draft reply is returned in the webhook response. The system does not email, text or message leads.
- Timeline rules recognise English only.
- 20/20 is tuned on the same leads. No independent holdout set has been run.
- The reply guard cannot catch every phrasing and allows third-person phrases such as "an agent will confirm a slot".
- Two database steps (property matching and lead update) continue on error without a dedicated alert.
- The daily digest uses a fixed 08:00 schedule and was verified by manual run only; the schedule was kept inactive.
- There is no rate limit and no daily cap on AI calls; a flood of unique leads could use up free quotas.
- Free-tier AI services can change, throttle or retain prompts. OpenRouter free models were unstable.
- Not tested: Telegram being down, embedding service failure, a scheduled digest run, self-hosted n8n.
- Database-down and fallback tests were run on version 1.0; version 1.1 changed only the scoring step, the prompt and the validation step.

## 12. Not built

CRM integration, dashboard or user interface, outbound email or SMS or WhatsApp, voice agents, multi-agent systems, payments.

## 13. Possible future improvements (planned only, not implemented)

Independent holdout test with fresh leads; error alerts after the matching and update steps; a daily AI-call cap and rate limiting; digest hour and timezone read from configuration; outbound reply delivery with human approval; multilingual timeline rules; self-hosted n8n deployment test.

## 14. What this project demonstrates

Designing an AI workflow with deterministic safeguards around the model, grounding generated text in retrieved facts, testing honestly (including the failure and the tuning caveat), handling provider failure, protecting personal data, and keeping a system configurable instead of hard-coded.
