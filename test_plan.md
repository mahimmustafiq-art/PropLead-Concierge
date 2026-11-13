# Test Plan (final)
Results with evidence are in `test_results.md`. Nothing is PASS without evidence.

Runner: `tests/proplead_tests.py` (modes `labeled`, `edge`, `manchester`). Env vars: `WEBHOOK_URL`, `PROPLEAD_HEADER`, `PROPLEAD_TOKEN`.
Reset between runs: `database/reset_test_leads.sql`. Evidence queries: `database/evidence_queries.sql`.

1. Labeled dataset: 20 London leads (7 HOT, 7 WARM, 6 COLD), target >= 17/20.
2. Edge cases: E01 duplicate, E02 missing email, E03 malformed JSON, E04 long message, E05 non-English, E06 prompt injection, E07 no match, E08 out of area, E09 LLM failure (wrong Groq key), E10 Supabase unavailable (wrong Postgres credential), E11 wrong/missing token. Extra: X01 SPAM, X02 INVALID.
3. Manchester portability: 3 leads, DB config only, no workflow edit.
4. WF-B digest: normal data, then stale-lead data. WF-C: forced error.
