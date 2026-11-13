# Database Design (v0.1)

Postgres (Supabase) + pgvector. Embedding column: `vector(1024)` (mistral-embed). Index: HNSW, `vector_cosine_ops`. Query metric: cosine (`<=>`). Dimension must match everywhere (column, RPC arg, WF-D).
All tables have `id uuid pk default gen_random_uuid()`, `created_at timestamptz default now()`, RLS enabled, no public policies (n8n uses service_role).

## agency_config
agency_name, city, country, currency_code, currency_symbol, timezone, service_areas text[], contact_email, business_hours jsonb, telegram_label, `settings jsonb` (hot_min=70, warm_min=40, stale_days=3, max_message_chars=2000, llm.groq_model, llm.openrouter_model, embed.model, rag.top_k, rag.min_similarity, digest_hour), is_active, updated_at. Unique: agency_name.

## properties
agency_id fk, ref (unique per agency), title, area, postcode_district, listing_type (rent/sale), price numeric, bedrooms int, bathrooms int, property_type, status (available/let/sold/withdrawn), description, available_from date, updated_at.
Indexes: (agency_id, status, area), (agency_id, listing_type, price, bedrooms).

## leads
agency_id fk, name, email_norm, phone_norm (E.164-style digits), message_latest, interaction_count, preferred_area, budget_min, budget_max, bedrooms, timeline_text, source, service_area_ok bool, injection_suspected bool, score_rules, score_llm, score_final, category (HOT/WARM/COLD/SPAM/INVALID), score_reason, ai_status (OK/FALLBACK/UNAVAILABLE), llm_provider, rag_status (OK/UNAVAILABLE/SKIPPED), matched_property_refs text[], reply_draft, hot_alerted_at, first_seen_at, last_contact_at, updated_at.
Dedupe: partial unique indexes `(agency_id, email_norm) WHERE email_norm IS NOT NULL` and `(agency_id, phone_norm) WHERE phone_norm IS NOT NULL`.
Other indexes: (agency_id, category, updated_at), (agency_id, last_contact_at).

## kb_documents
agency_id fk, doc_key (unique per agency), title, category, content, content_hash, embedding vector(1024), embed_model, embedded_at, updated_at. One embedding per document (docs kept short, < 8k tokens). HNSW index on embedding.

## run_log (no raw PII)
agency_id, lead_id (nullable), workflow, execution_id, path (NEW/DUPLICATE/REJECTED/UNAUTHORIZED/DIGEST/KB_INGEST), category, http_status, ai_status, llm_provider, rag_status, calls_used int, duration_ms, notes (short).

## error_log (no raw PII)
workflow_name, execution_id, node_name, failure_stage, error_message (truncated 500), lead_id nullable, notified bool.

## Functions (RPC)
- `upsert_lead_dedup(...)` - atomic: find by email_norm or phone_norm, update if found (is_duplicate=true, no new row), else insert. Returns lead id, is_duplicate.
- `match_properties(agency, listing_type, budget, area, bedrooms)` - only status='available'; ranked by area match, price fit, bedroom fit; limit 3.
- `match_kb(agency, query_embedding vector(1024), k, min_similarity)` - cosine similarity search.
- `digest_stats(agency, since)` and `stale_leads(agency, days)` - for WF-B.
- `touch_keepalive()` - trivial read for keep-alive.
All functions: `security invoker`/fixed `search_path`, parameterized, granted to service_role only.

## Portability rule
Manchester = new agency_config row + properties + KB docs only. Workflows pick the active agency from config; no workflow edits.
