# Test Results (real runs, 2–5 Nov 2026, n8n Cloud trial)
Expected and Observed are kept separate. Evidence = pasted run output, DB rows or Telegram screenshots held by the project owner.

## 1. Labeled London leads
| Run | Workflow version | Agreement | Verdict |
|---|---|---|---|
| 1 | WF-A before tuning | 16/20 = 80% | FAIL (target >= 17/20) |
| 2 | WF-A with two score caps | 20/20 = 100% | PASS (see caveat) |
| 3 | WF-A v1.1 (stronger reply guard, hyphen normalisation), after clean reset, 2-5 Nov 2026 (exact day not recorded) | 20/20 = 100%, no duplicates (all 20 new lead_ids) | PASS (same 20 leads, still tuned) |
| 4 | WF-A v1.1 + hyphen and apostrophe normalisation, clean reset (reset query returned 0), 5 Nov 2026 | 20/20 = 100%, no duplicates (20 new lead_ids) | PASS (same 20 leads, still tuned) |

CSV evidence in `tests/`: `results_labeled.csv` is Run 4 and `results_edge.csv` is the v1.1 edge re-run. Both were transcribed from the runner's console output (same columns the runner saves; edge bodies are cut at 200 characters by the runner).

An earlier v1.1 attempt scored 0/20 because the 20 test leads were not deleted first, so every request was treated as a duplicate. That run is invalid and discarded.

Run 1 misses: L10, L11, L13 expected WARM, got HOT (scores 73, 70, 78, just above HOT 70); L16 expected COLD, got WARM. Telegram showed 10 HOT alerts instead of 7.
Fix: HOT needs a time/viewing signal (time score >= 8); no matched property caps at COLD.
Run 2 DB scores: HOT leads 93-95; L10 rules 43 + LLM 30 = 73 -> capped 69 (WARM); L13 46 + 33 = 79 -> 69; L16 17 + 29 = 46 -> 39 (COLD); L11 68 (no cap); COLD leads 18-39.
**Caveat:** the caps were designed after seeing run 1 on the same 20 leads, so 100% is tuned, not an independent test.

## 2. Edge cases
| # | Case | Expected | Observed | Verdict |
|---|---|---|---|---|
| E01 | Duplicate | 200, duplicate:true, no new row | 200, same lead_id; DB interaction_count=2, one row | PASS |
| E02 | Missing email | 200 | 200, WARM, phone-only | PASS |
| E03 | Malformed JSON | 400 malformed_json | 422 `Failed to parse request body` (returned by n8n before the workflow) | DEVIATION |
| E04 | Very long message | 200, truncated | 200; DB message length = 2000 | PASS |
| E05 | Non-English | reply in same language | Spanish reply, LON-R002; WARM 69 | PASS (limitation below) |
| E06 | Prompt injection | not HOT, AI skipped | COLD 18, injection_suspected=true, ai_status UNAVAILABLE, calls 0 | PASS |
| E07 | No matching property | none invented | COLD 39, refs empty, reply says no 6-bed | PASS |
| E08 | Out of area (Leeds) | COLD, area not covered | COLD 27, service_area_ok=false, reply says not covered | PASS |
| E09 | Simulated LLM failure | fallback chain | See section 3 | PASS |
| E10 | Supabase unavailable | 503 + WF-C alert | 503 `{"status":"error","reason":"service_unavailable"}` (curl, 3 Nov 03:38 GMT); Telegram `ERROR \| WF-A Intake Pipeline \| node Stop Supabase Unavailable \| exec 86` at 9:38 local; error_log row 03:38:46 (stage database, notified true) | PASS |
| E11 | Wrong / missing token | 401 | 403 `Authorization data is wrong!` for both | DEVIATION |
| X01 | SPAM | 422 | 422 SPAM too_many_links | PASS |
| X02 | INVALID | 422 | 422 INVALID valid_email_or_phone_required | PASS |

Limitation: the timeline rules only recognise English words, so the Spanish "esta semana" lead scored WARM 69, not HOT.

## 2b. Edge cases re-run on WF-A v1.1 (2-5 Nov 2026, exact day not recorded; after labeled run 3)
Expected and observed kept separate. Observed = runner output (`tests/results_edge.csv`) plus one DB query on the new rows.
| # | Expected | Observed | Verdict |
|---|---|---|---|
| E01 | 200, duplicate:true, no new row | 200, `duplicate:true`, same lead_id as L01 (`72f4f1b7`). DB `interaction_count` for that lead was not shown in this run's query | PASS (response); DB count shown earlier on v1.0 |
| E02 | 200 accepted | 200, WARM, phone-only, matched LON-R003 | PASS |
| E03 | 400 (contract) | 422 `Failed to parse request body` (n8n, before the workflow) | DEVIATION (unchanged) |
| E04 | 200, truncated to 2000 | 200 WARM; DB message length = 2000 | PASS |
| E05 | reply in Spanish, no invented facts | 200 WARM, Spanish reply, matched LON-R002 (reply text read only in truncated runner output) | PASS (limitation: English-only timeline rules, WARM not HOT) |
| E06 | not HOT, AI skipped | 200 COLD, injection_suspected = true, ai_status UNAVAILABLE, llm_provider none | PASS |
| E07 | no property invented | 200 COLD, matched refs empty, reply says no 6-bedroom house | PASS |
| E08 | COLD, area not covered | 200 COLD, matched refs empty, reply starts "Unfortunately we d..." (truncated in output) | PASS |
| E11a/b | 401 (contract) | 403 `Authorization data is wrong!` for wrong and missing token | DEVIATION (unchanged) |
| X01 | 422 SPAM | 422 SPAM too_many_links | PASS |
| X02 | 422 INVALID | 422 INVALID valid_email_or_phone_required | PASS |
E09 and E10 were not re-run on v1.1 (E09 fallback and E10 503 were proven on v1.0; the v1.1 changes do not touch the DB-unavailable branch or the fallback wiring).

## 3. Fallback evidence (E09, Groq key deliberately wrong)
| Attempt | Result |
|---|---|
| 1 (llmfail1) | Groq 401, OpenRouter answered but a JSON key had a leading space -> rejected -> ai_status UNAVAILABLE, calls 3 (rules-only path PASS) |
| 2 (llmfail2) | OpenRouter returned empty content (reasoning model used all 1200 tokens, finish_reason length) -> UNAVAILABLE, calls 3 |
| 3 (llmfail3) | After fixes: ai_status FALLBACK, provider openrouter, rules 55 + LLM 28 = 83, reply=llm, calls 3 |
OpenRouter free router picks a different model each time, so success is not guaranteed.

## 4. Other acceptance evidence
| Item | Evidence | Verdict |
|---|---|---|
| n8n trial usage | Screenshot 5 Nov 2026: 11 days left, 158/1000 executions (earlier: 3 Nov 14 days left, 138/1000; 2 Nov 68/1000) | PASS (evidence held by project owner) |
| Call cap <= 3 | max calls_used = 3 over all NEW runs; 3 calls whenever Groq fails (FALLBACK or UNAVAILABLE), otherwise 2 (duplicates and injection: 0) | PASS |
| Duplicate | E01 | PASS |
| HOT Telegram alert | alerts received for every HOT lead; text has no email/phone/message | PASS |
| Alert timing | `hot_alerted_at - first_seen_at` for 7 HOT leads (run 4, 5 Nov 2026): 1.5-5.9 s, 7/7 alerted; the 5.9 s request was the slowest and the cause is not recorded. Earlier v1.1 runs: 1.5-2.9 s. Measures lead insert to recorded Telegram success; excludes webhook auth and delivery to the phone. Telegram screenshots show minute-level arrival | PASS (limits stated) |
| WF-B digest | counts matched DB (HOT 10, WARM 11, COLD 11, SPAM 1, INVALID 1, total 34) | PASS |
| WF-B stale leads | 2 WARM leads aged 5 days; digest WARM 11 -> 9 and lists both ids with 5d | PASS |
| WF-C | forced error -> Telegram + error_log row, email scrubbed | PASS (earlier chat) |
| WF-D | 17 embeddings first run (user-reported). London active (query), kb_documents London 17/17 and Manchester 7/7 embedded. London re-run 3 Nov 03:45 UTC: run_log KB_INGEST, rag_status SKIPPED, calls 0, "no pending docs (all embeddings up to date)". It confirms up-to-date state, not a fresh re-embedding | PASS (no-pending path) |
| Manchester portability | M01 HOT, M02 WARM, M03 COLD, replies used Manchester hours/refs/areas, alert label Manchester; no workflow edit | PASS |
| Manchester run 1 | seed not applied: tests hit London data (invalid run, discarded). Run 2 after seed: Groq 404 because the seed carried an old model ID; DB fix then rerun passed | see fixing_note |

## 4b. E10 method
Credential on node `Load Agency Config` only was switched to a deliberately invalid Postgres host, one valid lead was sent to the production webhook, then the original credential was restored. WF-C used the working credential, so its error_log insert succeeded.

## 4c. Reply safety audit (AC4)
History: (a) the first v1.1 audit missed replies written with a non-ASCII hyphen (LON‑R003); (b) a reviewer found that a curly apostrophe ("I’ll arrange a viewing") bypassed the promise guard. Both are fixed in Score and Guard (Unicode hyphens/dashes become "-", curly apostrophes become "'"), and the audit SQL (query 6) now normalises both.

Run 4 (final), 20 leads:
- SQL audit (query 6): refs_ok = true and no_promise = true for all 20. 18 replies came from the LLM and 2 from the safe template (L17 out of service area, which always uses the template, and L20, see 4d).
- The 7 HOT replies were read by hand. None promises or confirms a viewing; each says an agent will contact the lead to discuss viewing times (L01 mentions the lead's own Thursday/Friday evening request). Prices and dates match the database: LON-R003 £1950, LON-S008 £395,000, LON-R011 £2850 available from 1 December, LON-R008 £3600, LON-R005 £2300; LON-S007 is described as within the £830000 budget (listed £820000). L07 (move within 10 days) gets a safe reply that does not mention that LON-R005 is available from 25 November.
- Reply text such as "an agent will ... confirm a slot" is third-person and is intentionally allowed (agents do confirm times). The guard blocks first-person promises, not third-person ones.
- Limit: the audit uses the same patterns as the guard; only the 7 HOT replies were checked by a human.

## 4d. Natural both-providers-failed case (run 4, L20)
Lead fa37938f: ai_status UNAVAILABLE, llm_provider none, calls_used 3, duration 39.4 s, score_reason "rules=16 ... llm=n/a final=27 reply=template | rules_only_scaled", category COLD (as expected). The Groq and OpenRouter calls both produced no usable output; the cause of the failure is not recorded and is not known. The request still returned 200 with a COLD result and a safe template reply, which shows the rules-only path working on its own, not simulated. This is an UNAVAILABLE row, not a FALLBACK row; the FALLBACK evidence is E09 attempt 3 (section 3).

## 4e. Regression status after v1.1
Labeled runs 3 and 4, the reply audit and edge cases E01-E08, X01, X02, E11 were run on v1.1 (section 2b). E09 and E10 were run on v1.0 only.

## 5. Not tested
Telegram down, Mistral failure, WF-B no-activity template path, WF-B scheduled trigger (kept inactive).

## 6. Acceptance criteria (the brief's AC1-AC12)
AC1 unauthorized/malformed rejected: PASS (403/422 returned by n8n before the workflow, so not written to run_log) | AC2 duplicate creates no new row: PASS | AC3 >= 85% on 20 labeled leads: PASS (20/20, tuned on the same leads) | AC4 grounded replies, no promises: PASS (v1.1; SQL audit 20/20, 7 HOT replies read by hand) | AC5 prompt injection not obeyed: PASS | AC6 fallback chain: PASS (Groq -> OpenRouter -> rules-only) | AC7 HOT alert within 60 s: PASS (max 5.9 s insert to recorded alert) | AC8 daily digest matches DB: PASS | AC9 no secrets in submitted files: PASS (sanitized exports scanned) | AC10 <= 3 calls and quotas: PASS (max calls_used 3; trial 158/1000 executions) | AC11 README and diagram: PASS | AC12 portability (Manchester, no workflow edit): PASS
