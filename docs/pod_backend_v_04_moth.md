#MOTH:repo
meta:{project:POD-Backend;version:0.5;date:2025-10-26;author:John}

[GOAL]
Reusable backend for multi-tenant print-on-demand art storefronts with Printify fulfillment, Stripe payments, frame-size registry, vector search, AI upscaling, and robust automation.

[FEATURES.mvp]
modules:TenantAuth;Catalog;FrameRegistry;Orders;Providers;AI;VectorSearch;CRM;Automation;Security;Monitoring
providers:{primary=Printify;future=Staples,Amazon}
auth:{supabase=true;google=true;apple=true;webhook_hmac=true;rls=true}
payments:{stripe=true;stripe_tax=true}
products:{poster=true;canvas=true}
image_pipeline:{fal_ai_async=true;upscale_threshold=300dpi}
vector_embeddings:{pgvector=true;clip=true;indexing=ivfflat}
refund_policy:{lenient=true;auto_approve_days=30}
order_hold:{enabled=true;duration=15min}

[FEATURES.future]
ph2:AmazonStoreConnector;LocalPickup;VisionWallMatch
ph3:ARWallPreview;ArtistRoyalties;ExtraProviders

[NON_FUNCTIONAL]
security:{webhook_hmac=true;pii_encryption=true;tenant_isolation=RLS;gdpr_compliance=true}
performance:{search_latency<500ms;max_concurrent_checkouts=1000;uptime>=99.9}
compliance:{gdpr=true;ai_art_license_verification=true}

[RISK_MITIGATION]
webhook_failure:{retry=3;poll_interval=5min;fallback=GET_orders_status}
fal_ai_cost_cap:{max_jobs_per_tenant=1000;alert_threshold=80%}
ikea_fallback:{manual_curation=true;phase2_paid_api=true}

[TESTING]
unit:{dpi_calc;webhook_parser;payment_flow}
integration:{printify_webhook_mock;fal_ai_async_job;stripe_refund}
load:{pgvector_query_latency;order_submission_under_load}
validation:{ikea_source_hash_consistency;data_schema_diff}

[METRICS]
KPIs:{order_success_rate>99%;webhook_success_rate>98%;avg_response_time<500ms}
monitoring_tools:{sentry;prometheus;supabase_analytics}
logging:{structured_json=true;trace_id_per_request=true}

[SCHEMAS]
tenants:{id;name;primary_domain;settings_json}
roles:{user_id;tenant_id;role}
artworks:{id;tenant_id;title;creator;source_type;license_type;storage_paths;status}
products:{id;artwork_id;title;category}
product_variants:{id;product_id;sku;substrate;size_id;base_cost_cache;markup_rule_id}
frame_types:{id;label;region}
frame_sizes:{id;frame_type_id;label;width_in;height_in;width_mm;height_mm;has_mat;mat_window_w;mat_window_h}
provider_sizes:{id;provider;provider_product_id;width_in;height_in}
size_reconciliations:{frame_size_id;provider_size_id;fit_mode;dpi_result;status}
affiliate_frames:{size_id;retailer;asin_or_sku;url;region;api_provider}
orders:{id;tenant_id;user_id;stripe_intent_id;order_type;status;totals_json;provider_orders_json;shipping_address_json;refund_status}
fulfillments:{id;order_id;provider_order_id;tracking_json}
ai_jobs:{id;artwork_id;job_type;status;webhook_token}
ai_results:{id;job_id;result_url;metadata_json}
vector_embeddings:{artwork_id;embedding_vector}
crm_contacts:{id;user_id;tenant_id;email;name;phone}
crm_tickets:{id;contact_id;order_id;tenant_id;status;subject}
crm_interactions:{id;ticket_id;user_id;message;type}

[API.frame_registry]
GET:/v1/frame-types
GET:/v1/frame-types/{id}/sizes
POST:/v1/dpi/calc → {pixels_w,pixels_h,print_w_in,print_h_in}
GET:/v1/reconcile?frame_size_id=&provider=
GET:/v1/affiliates/frames?size_id=&region=
policy:{ok>=300;warn=200-299;block<200}

[API.print_provider]
get_product_catalog();get_variants(id);get_shipping_quote(variant_id,address);submit_order(payload);get_order_status(id);parse_webhook(payload)
provider:{printify=true;staples=false}

[PIPELINES.ai]
validate_aspect_ratio();if dpi<threshold→queue fal.ai(job_type=upscale)
webhook:on_complete→update ai_results,status=complete
auth:Bearer;storage:supabase

[PIPELINES.vector]
on_upload:generate_clip_embedding→store_pgvector
endpoints:{/v1/search/similar;/v1/search/curated}

[PAYMENTS]
stripe_tax:true;merchant_of_record:owner
lifecycle:{checkout;payment_intent;status:on_hold;15min_job→submit_order}
refunds:{auto_approve:true;window=30d}

[CRM]
auto_ticket:{payment_fail;shipping_issue;refund}
notify_channels:{email;webhook}

[AUTOMATION.jobs]
nightly:{provider_catalog_sync;affiliate_refresh;ikea_harvest;low_dpi_report}
minute:{order_hold_release}
event:{fal_ai_webhook;crm_notifications}
idempotent:true;use_pgcron:true;failure_topic:DLQ;retry_logic:true

[DEPLOYMENT]
containerized:true;platform:Render/Fly/AWS;db:Postgres+pgvector;auth:Supabase;monitoring:Prometheus/Sentry

[APPENDIX.printify_api]
endpoints:{catalog,variants,shipping,orders,webhooks}
auth:Bearer
notes:{send_to_production:false→hold;webhook:order:updated}

[APPENDIX.fal_ai]
model:ESRGAN;auth:Bearer
flow:{async_job;webhook_callback;update_ai_results}
endpoint:/models/fal-ai/esrgan/infer

[APPENDIX.amazon_paapi]
auth:AWSv4;operations:{SearchItems,GetItems};purpose:affiliate_frames_refresh

[APPENDIX.ikea_frames]
method:semi_automated;strategy:{manual,paid_api,playwright}
tables:{ikea_frames,frame_size_alias}
cron:nightly;rate_limit:2req/min
risk:{tos_breakage,regional_variance}
fallback:{manual_curation:true;paid_api_phase2:true}

[SCHEMAS.ikea_frames]
columns:{id;series;product_name;article_number;region;country_code;product_url;inner_opening_w_mm;inner_opening_h_mm;mat_window_w_mm;mat_window_h_mm;outer_w_mm;outer_h_mm;color_finish;has_mat;included_mat_sizes_json;price_minor;currency;availability_status;images_json;source_timestamp;last_verified_at;source_hash}

[END]
checksum:sha1:89b78d7fa3

