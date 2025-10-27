# PRD — Multi-Tenant Print-on-Demand Backend (v0.5)

## 1. Purpose
Reusable backend for multi-tenant print-on-demand art storefronts with Printify fulfillment, Stripe payments, frame-size registry, vector search, AI upscaling, and robust automation.

## 2. In-Scope (MVP)
- Modules: TenantAuth, Catalog, FrameRegistry, Orders, Providers, AI, VectorSearch, CRM, Automation, Security, Monitoring
- Providers: Printify (primary), future: Staples, Amazon
- Auth: Supabase (email, Google, Apple), webhook HMAC validation, Row-Level Security (RLS)
- Payments: Stripe + Stripe Tax
- Products: Posters and Canvases
- Image Pipeline: fal.ai async upscale (>300 DPI)
- Vector Embeddings: pgvector + CLIP (IVFFlat index)
- Refund Policy: Lenient, auto-approve within 30 days
- Order Hold: Enabled, 15 minutes duration before provider submission

## 3. Out-of-Scope (MVP)
- Staples integration (deferred)
- Amazon Store Connector (Phase 2)
- Vision-based wall matching and AR preview (Phase 2–3)
- Artist Royalties (Phase 3)

## 4. Non-Functional Requirements
**Security:** Webhook HMAC validation, encrypted PII, Supabase RLS isolation, GDPR compliance
**Performance:** <500ms search latency, 1000 concurrent checkouts, 99.9% uptime
**Compliance:** GDPR + AI art license verification

## 5. Risk Mitigation
- **Webhook Failures:** 3 retries, 5-min polling fallback using Printify GET orders
- **fal.ai Cost Cap:** Limit 1000 jobs per tenant/month, alert at 80%
- **IKEA Integration:** Manual curation fallback, Phase 2 paid API (Zyla or similar)

## 6. Testing Strategy
**Unit:** dpi_calc, webhook_parser, payment_flow
**Integration:** mock Printify webhooks, fal.ai async jobs, Stripe refunds
**Load:** pgvector latency under concurrency, order submission throughput
**Validation:** ikea_frames.source_hash consistency, schema diffs

## 7. Metrics & Monitoring
- KPIs: Order success >99%, webhook success >98%, avg API <500ms
- Tools: Sentry, Prometheus, Supabase Analytics
- Logging: Structured JSON with trace_id per request

## 8. Core Data Schema
- **tenants:** id, name, primary_domain, settings_json
- **roles:** user_id, tenant_id, role
- **artworks:** id, tenant_id, title, creator, source_type, license_type, storage_paths, status
- **products:** id, artwork_id, title, category
- **product_variants:** id, product_id, sku, substrate, size_id, base_cost_cache, markup_rule_id
- **frame_types:** id, label, region
- **frame_sizes:** id, frame_type_id, label, width_in, height_in, width_mm, height_mm, has_mat, mat_window_w, mat_window_h
- **provider_sizes:** id, provider, provider_product_id, width_in, height_in
- **size_reconciliations:** frame_size_id, provider_size_id, fit_mode, dpi_result, status
- **affiliate_frames:** size_id, retailer, asin_or_sku, url, region, api_provider
- **orders:** id, tenant_id, user_id, stripe_intent_id, order_type, status, totals_json, provider_orders_json, shipping_address_json, refund_status
- **fulfillments:** id, order_id, provider_order_id, tracking_json
- **ai_jobs:** id, artwork_id, job_type, status, webhook_token
- **ai_results:** id, job_id, result_url, metadata_json
- **vector_embeddings:** artwork_id, embedding_vector
- **crm_contacts:** id, user_id, tenant_id, email, name, phone
- **crm_tickets:** id, contact_id, order_id, tenant_id, status, subject
- **crm_interactions:** id, ticket_id, user_id, message, type
- **ikea_frames:** id, series, product_name, article_number, region, product_url, inner_opening_w_mm, inner_opening_h_mm, mat_window_w_mm, mat_window_h_mm, outer_w_mm, outer_h_mm, color_finish, has_mat, price_minor, currency, availability_status, images_json, source_timestamp, last_verified_at, source_hash

## 9. API Summary
**Frame Registry:**
- GET /v1/frame-types
- GET /v1/frame-types/{id}/sizes
- POST /v1/dpi/calc → {pixels_w, pixels_h, print_w_in, print_h_in}
- GET /v1/reconcile?frame_size_id=&provider=
- GET /v1/affiliates/frames?size_id=&region=
Policy: OK ≥300 DPI, Warn 200–299, Block <200

**Print Provider:**
- get_product_catalog(), get_variants(id), get_shipping_quote(), submit_order(), get_order_status(), parse_webhook()
- Provider: Printify (true), Staples (false)

## 10. Pipelines
**AI Pipeline:** Validate aspect ratio; if DPI < threshold, queue fal.ai upscale job. fal.ai auth: Bearer key. On complete, webhook → ai_results update.
**Vector Pipeline:** Generate CLIP embedding on upload → store in pgvector. Endpoints: /v1/search/similar, /v1/search/curated.

## 11. Payments
- Stripe + Stripe Tax
- Merchant of Record: Owner
- Lifecycle: checkout → payment_intent → on_hold (15-min job → submit_order)
- Refunds: auto_approve within 30 days

## 12. CRM
- Auto-tickets: payment_fail, shipping_issue, refund
- Notify: email + webhook

## 13. Automation Jobs
- Nightly: provider_catalog_sync, affiliate_refresh, ikea_harvest, low_dpi_report
- Minute: order_hold_release
- Event: fal_ai_webhook, crm_notifications
- Idempotent: true, retry logic enabled, DLQ fallback, pgcron scheduled

## 14. Deployment
Containerized; platform: Render/Fly/AWS; DB: Postgres+pgvector; Auth: Supabase; Monitoring: Prometheus + Sentry

## 15. Appendix
**Printify API:** Bearer auth; endpoints catalog, variants, shipping, orders, webhooks. send_to_production=false for hold. webhook: order:updated.
**fal.ai:** ESRGAN model, Bearer auth, endpoint /models/fal-ai/esrgan/infer
**Amazon PAAPI:** AWSv4 auth; operations SearchItems, GetItems
**IKEA Frames:** Semi-automated (manual, paid API, Playwright); cron nightly; rate_limit 2 req/min; risk: tos_breakage; fallback: manual curation, paid API Phase 2

---
**Checksum:** sha1:89b78d7fa3