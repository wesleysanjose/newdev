# Domain Knowledge Scan: menumind-api

**Scan Date:** 2026-02-15
**Scope:** `~/dev/menumind-api` - Full codebase (13+ microservices, shared libraries, tests, scripts, infrastructure)

---

## Summary

| Layer | Count | Confidence |
|-------|-------|------------|
| Physical/Mathematical | 18 | high |
| Operational/Tradeoff | 32 | high |
| Strategic/Political | 14 | medium |
| Historical/Path-Dependent | 9 | medium |
| Ambiguous | 5 | -- |

---

## Layer 1: Physical / Mathematical

These are non-negotiable invariants derived from math, formal rules, data integrity, or hard regulatory requirements. AI can safely own, test, refactor, and optimize these.

### 1.1 Points Ledger Integrity
- **Location**: `shared/services/points/service.py:208`
- **What**: Balance = SUM(amount) across all transactions. No separate balance table. Positive = earn, negative = spend.
- **Evidence**: Pure ledger design with `pg_advisory_xact_lock` for atomic redemption (line 319-320). SYSTEM_UUID = `"00000000-0000-0000-0000-000000000000"` for automated operations.
- **AI Actionable**: Yes -- AI can safely test, refactor, optimize this. The math is deterministic.

### 1.2 Fernet Encryption (PBKDF2 480K iterations)
- **Location**: `shared/utils/encryption.py:66-84`
- **What**: API key encryption using PBKDF2-SHA256 with 480,000 iterations and Fernet cipher (AES-128-CBC + HMAC).
- **Evidence**: `iterations=480000` follows NIST 2025+ recommendation. Authenticated encryption prevents tampering.
- **AI Actionable**: Yes -- cryptographic operations are deterministic. Tests can verify encrypt/decrypt round-trips.

### 1.3 HMAC-SHA256 URL Signing
- **Location**: `shared/utils/cdn_url_generator.py:55-84`
- **What**: CDN URLs signed with HMAC-SHA256. Message format: `"path:expires"` or `"path:expires:user_id"`. Constant-time comparison via `hmac.compare_digest()`.
- **Evidence**: Timing attack prevention (line 89), optional user-ID binding prevents cross-user URL sharing.
- **AI Actionable**: Yes -- signature generation/verification is pure math.

### 1.4 Apple JWS Verification (ES256)
- **Location**: `services/subscription_server/app/utils/jws_verify.py:109-168`
- **What**: Verifies Apple App Store Server Notifications using ES256 public key from Apple's JWKS endpoint. Certificate chain validation.
- **Evidence**: Fetches JWKS from `https://appleid.apple.com/auth/keys`, caches 24h, auto-refreshes on key rotation.
- **AI Actionable**: Yes -- cryptographic verification is deterministic.

### 1.5 Rating Constraint (1-5)
- **Location**: `shared/models/feedback.py:56, 77`
- **What**: Database CHECK constraint enforcing rating between 1 and 5.
- **Evidence**: PostgreSQL CHECK constraint at schema level, cannot be bypassed by application code.
- **AI Actionable**: Yes -- schema-level invariant.

### 1.6 Soft Delete Enforcement
- **Location**: `tests/regression/test_auditability.py`, `scripts/lint_migrations.py:378-531`
- **What**: All entities must have `deleted_at`/`deleted_by` columns. No hard deletes. Migration linter enforces via 10 rules (AUDIT001-005, FK001-002, TRG001, DEL001, IDX001).
- **Evidence**: Regression tests verify trigger functions, audit columns, and soft delete pattern. Grandfather revision = 16 (pre-spec migrations exempted).
- **AI Actionable**: Yes -- structural constraint enforced by linter and tests.

### 1.7 Audit Column Enforcement
- **Location**: `scripts/lint_migrations.py:55-56`, `shared/utils/audit_context.py`
- **What**: Every table requires `created_at`, `updated_at`, `created_by`, `updated_by`. Trigger function `update_updated_at_column()` auto-sets `updated_at=NOW()` on UPDATE.
- **Evidence**: Migration linter errors (AUDIT001-004) block non-compliant migrations. ContextVar-based actor propagation ensures actor_id is always set.
- **AI Actionable**: Yes -- enforced by tooling.

### 1.8 Unique Referral Constraint
- **Location**: `shared/models/referral.py:62`
- **What**: `referee_id` has unique constraint -- each user can only be referred once. Self-referral prevention also enforced.
- **Evidence**: Database-level unique constraint, application-level self-referral check.
- **AI Actionable**: Yes -- data integrity constraint.

### 1.9 Image Processing Pipeline
- **Location**: `shared/utils/image_optimizer.py:14-16`, `services/menu_parsing/app/services/image_resize_service.py:22-24`
- **What**: MAX_DIMENSION=1536px, JPEG_QUALITY=85, LANCZOS resampling, 1.1x sharpening. Progressive quality reduction sequence: [75, 70, 65, 60] if still too large.
- **Evidence**: Deterministic image transformation pipeline. RGBA/LA/P mode conversion to RGB with white background.
- **AI Actionable**: Yes -- image processing math is deterministic.

### 1.10 Dish Name Normalization Pipeline
- **Location**: `shared/services/dish/normalization.py:53-125`, `shared/utils/dish_normalizer.py:76-119`
- **What**: 5-step normalization: lowercase, remove diacritics (NFD), remove prices/parenthetical, expand abbreviations, collapse whitespace. Canonical mapping from JSON dictionary.
- **Evidence**: Idempotent transformation. Base64-encoded normalized name for cache keys.
- **AI Actionable**: Yes -- pure string transformation.

### 1.11 Period Key Format
- **Location**: `shared/models/activity.py:53`
- **What**: Monthly usage bucketing uses period_key format `"YYYY-MM"` (e.g., `"2025-12"`).
- **Evidence**: Consistent across activity recorder and usage tracking.
- **AI Actionable**: Yes -- date formatting invariant.

### 1.12-1.18 Additional Invariants
- **App-user uniqueness**: `uq_subscription_customers_app_user` constraint (customer.py:55)
- **Transaction ID uniqueness**: Per-transaction dedup (transaction.py:28)
- **One active menu per restaurant**: Deactivate-all-before-activate pattern (menu_service.py:250-251)
- **Only one verified claim per restaurant**: Prevents duplicate ownership (claim_service.py:113-132)
- **Product type constraints**: CheckConstraint for 5 product types (product.py:49)
- **Package type constraints**: CheckConstraint for 9 package types (offering.py:74)
- **Entitlement app-local uniqueness**: `uq_subscription_entitlements_app_identifier` (entitlement.py:51)

---

## Layer 2: Operational / Tradeoff

Logic shaped by incidents, performance constraints, cost, scalability, or reliability concerns. AI can measure impact, but humans decide tolerance.

### 2.1 Circuit Breaker Configuration
- **Location**: `shared/config/settings.py:250-254`, `services/image_search/app/providers/manager.py:22-23`
- **What**: Subscription service circuit breaker with threshold=5 failures, timeout=30s. Image search circuit breaker with threshold=3, recovery=60s.
- **Current Value**: 5 failures / 30s (subscription), 3 failures / 60s (image search)
- **Evidence**: State machine: CLOSED -> (N failures) -> OPEN -> (timeout) -> HALF_OPEN -> (success) -> CLOSED
- **Question for domain expert**: What incident drove these specific thresholds? Has the subscription service's failure rate changed since these were set?

### 2.2 Grace Period (72 hours)
- **Location**: `shared/services/tier_service.py:133-134`
- **What**: When subscription server is unavailable, cached tier validity persists for 72 hours before defaulting to FREE.
- **Current Value**: `GRACE_PERIOD_HOURS = 72`
- **Evidence**: Comment: "3 days, like RevenueCat". Tests verify 24h=pass, 96h=fail boundary.
- **Question for domain expert**: Is 72 hours the right balance between user experience (avoiding false paywalls) and revenue protection?

### 2.3 Subscription Cache TTL
- **Location**: `shared/config/settings.py` (300s), `services/menu_parsing/app/services/config_loader.py:36`
- **What**: LLM config, image processing config, and provider config all cached for 300 seconds (5 minutes).
- **Current Value**: 300s for configs, 86400s (24h) for image search cache, 604800s (7d) for dish existence cache
- **Evidence**: Multiple cache layers with different TTLs based on data volatility.
- **Question for domain expert**: Have stale configs ever caused production issues? Should config cache be shorter?

### 2.4 HTTP Retry Configuration
- **Location**: `shared/utils/http_retry.py`, `shared/config/settings.py`
- **What**: 3 retries, 1.0s initial delay, 10.0s max delay, 2.0s jitter. Only retries ConnectError/ConnectTimeout.
- **Current Value**: max_retries=3, initial_delay=1.0s, max_delay=10.0s, jitter=2.0s
- **Evidence**: Exponential backoff with jitter for inter-service HTTP calls.
- **Question for domain expert**: Are 3 retries sufficient for the observed failure patterns? Has jitter been tuned?

### 2.5 Webhook Forwarding Retry
- **Location**: `services/subscription_server/app/services/webhook_forwarder.py:62-64`
- **What**: 3 retries with exponential backoff (1s, 2s, 4s). 10s timeout per attempt.
- **Current Value**: MAX_RETRIES=3, TIMEOUT=10s, BACKOFF_BASE=1s
- **Evidence**: `2^attempt` formula for delay calculation.
- **Question for domain expert**: What happens when all 3 retries fail? Is there a dead-letter queue?

### 2.6 Subscription Retry Queue Backoff
- **Location**: `shared/services/subscription_audit.py:464-466`
- **What**: Exponential backoff for subscription operation retries: `2^retry_count` minutes (1, 2, 4, 8, 16 min).
- **Current Value**: max_retries=5, retention=90 days
- **Evidence**: Persistent retry queue with database-backed state.
- **Question for domain expert**: Has the 90-day retention been validated against compliance requirements?

### 2.7 Database Connection Pool
- **Location**: `.env.example:18-19`
- **What**: PostgreSQL connection pool size=10, max overflow=20.
- **Current Value**: pool_size=10, max_overflow=20, pool_recycle=3600s
- **Evidence**: Standard asyncpg pool configuration.
- **Question for domain expert**: Is pool_size=10 sufficient for peak load? Has connection exhaustion occurred?

### 2.8 JWT Token Expiry
- **Location**: `.env.example:32-33`
- **What**: Access token expires in 1 hour, refresh token in 30 days.
- **Current Value**: access=1h, refresh=30d
- **Evidence**: HS256 JWT signing with configurable expiry.
- **Question for domain expert**: Is 1-hour access token appropriate for mobile app UX (background refresh)?

### 2.9 Rate Limiting Configuration
- **Location**: `.env.example:25-27, 155-158`
- **What**: General: 100 requests/hour. Detailed: daily=100, hourly=20, monthly=3000, burst=10.
- **Current Value**: Multiple rate limit tiers
- **Evidence**: Per-user rate limiting with custom overrides possible.
- **Question for domain expert**: Are these limits based on actual abuse patterns or theoretical?

### 2.10 AI Provider Timeout & Retries
- **Location**: `.env.example:63-65`
- **What**: Provider timeout=30s, max retries=3, failover enabled.
- **Current Value**: 30s timeout, 3 retries, failover=true
- **Evidence**: Multi-provider failover: OpenAI -> Anthropic -> Google.
- **Question for domain expert**: Has the 30s timeout caused premature failures for complex menu images?

### 2.11 Image Search Provider Rate Limits
- **Location**: Various provider files in `services/image_search/app/providers/`
- **What**: Provider-specific rate limits vary significantly.
- **Current Value**: Google Custom Search: 100/day, 10/min. Bing: 1000/day, 30/min. Unsplash: 5000/day, 50/min. Pexels: 200/day, 20/min. Google Scraper: 10000/day, 20/min.
- **Evidence**: Conservative estimates to avoid API key revocation.
- **Question for domain expert**: Are these actual API limits or self-imposed? Have we hit rate limits in production?

### 2.12 Streaming & Polling Timeouts
- **Location**: `services/api_gateway/app/services/menu_streaming_client.py:20, 27`
- **What**: SSE streaming timeout=300s (5 min), non-streaming=10s. Polling: 60s timeout, 2s interval, max 60 polls.
- **Current Value**: streaming=300s, sync=10s, poll_interval=2s
- **Evidence**: Long-running SSE connections for real-time menu parsing.
- **Question for domain expert**: Has 5-minute streaming timeout been validated against actual parsing times?

### 2.13 CDN Cache Configuration
- **Location**: `docker-compose.yml`, `.env.example:134-139`
- **What**: Cache max size=100MB, URL expiry=86400s (24h) in docker-compose but 14400s (4h) in code.
- **Current Value**: 100MB cache, 4h-24h URL expiry (inconsistent)
- **Evidence**: LRU eviction with size tracking.
- **Question for domain expert**: Why is URL expiry inconsistent between config and code (4h vs 24h)?

### 2.14 Health Check Intervals
- **Location**: `docker-compose.yml`
- **What**: PostgreSQL: 10s interval, 5s timeout, 5 retries. Services: 30s interval, 10s timeout, 3 retries, 30-60s start period.
- **Current Value**: Various per service type
- **Evidence**: Docker HEALTHCHECK directives.
- **Question for domain expert**: Are start periods sufficient for cold starts under load?

### 2.15 Prometheus TSDB Retention
- **Location**: `docker-compose.yml:515`
- **What**: 200 hours (~8.3 days) metric retention.
- **Current Value**: 200 hours
- **Evidence**: `--storage.tsdb.retention.time=200h`
- **Question for domain expert**: Is 8 days sufficient for incident investigation?

### 2.16 OpenTelemetry Histogram Buckets
- **Location**: `shared/utils/otel.py:168-184`
- **What**: Operation-specific histogram boundaries tuned per service type.
- **Current Value**: Redis: 0.5ms-1s, S3: 10ms-10s, AI: 0.5s-120s, HTTP: 5ms-60s, GenAI tokens: 100-128K
- **Evidence**: 8 custom bucket configurations with 9 specialized metric classes.
- **Question for domain expert**: Are these bucket boundaries aligned with SLO targets?

### 2.17 Temporal Workflow Timeouts
- **Location**: `.env.example:168-169`
- **What**: Workflow timeout=300s (5 min), activity timeout=60s.
- **Current Value**: workflow=300s, activity=60s
- **Evidence**: Temporal.io orchestration for menu parsing.
- **Question for domain expert**: What happens when a workflow times out? Is there recovery/retry?

### 2.18 Photo Freshness TTL
- **Location**: `shared/services/restaurant/restaurant_service.py:37-38`
- **What**: Public photos cached for 180 days before refetch.
- **Current Value**: `PUBLIC_PHOTOS_TTL_DAYS = 180`
- **Evidence**: Prevents stale restaurant photos while managing API costs.
- **Question for domain expert**: Is 180 days appropriate? Do restaurants change photos more frequently?

### 2.19 Image Processing for AI vs Display
- **Location**: `services/image_processing/app/services/image_service.py:32-34, 133`
- **What**: Menu images: max 2048px, quality 90 (high for AI). Display variants: 800, 400, 200, 100px in JPEG+WebP.
- **Current Value**: AI=2048px/q90, display=800-100px variants
- **Evidence**: Different optimization targets for different use cases.
- **Question for domain expert**: Has the 2048px limit affected AI parsing accuracy for large/detailed menus?

### 2.20 Subscription Health Check
- **Location**: `shared/jobs/subscription_health_check.py`
- **What**: Runs every 5 minutes, batch_size=100, max_users=500, 7-day expiration cutoff.
- **Current Value**: 5min interval, 100/batch, 500 max, 7d cutoff
- **Evidence**: Prevents runaway execution with max_users cap.
- **Question for domain expert**: Has 500 users/run been sufficient? Are anomalies caught in time?

### 2.21-2.32 Additional Operational Items
- Retry processor: every minute, batch=100, retention=90 days
- Stats aggregation: daily at 2 AM, 1 day default window
- Google scraper anti-detection: User-Agent spoofing, 3 retries, 1s base delay, exponential backoff
- Connection limits: 100 total, 10 per host, 30s keepalive
- Over-fetch multiplier: 2.0x target images (request double for resilience)
- Target images per dish: 5 (configurable)
- Menu parsing request timeout: 60s sync
- Image search timeout: 30s
- Verification code: 6 digits, 15-minute expiry, 5 max attempts
- Gemini capture: temperature=0.1, max_tokens=8192
- Toxiproxy chaos thresholds: 5KB connection reset, 2s latency, 1s jitter, 3s freeze

---

## Layer 3: Strategic / Political

Decisions shaped by business strategy, OKRs, priorities, or market conditions. Humans must own these.

### 3.1 Tier Limits (FREE=5, BASIC=20, PRO=unlimited)
- **Location**: `shared/models/activity.py:18-31`
- **What**: Three-tier subscription model defining monthly scan limits.
- **Evidence**: `TIER_LIMITS` dict mapping tier to action limits. Tests enforce exact values.
- **Unknown**: Why 5/20/unlimited specifically? Based on user research or market positioning?
- **Decision owner**: Product team / pricing strategy owner

### 3.2 Dish Confidence Scoring Weights
- **Location**: `shared/services/dish/confidence.py`
- **What**: Weighted blend: AI=0.35, source_count=0.25, consistency=0.25, quality=0.15. Source quality: owner=1.0, public=0.7, user_scan=0.5.
- **Evidence**: Weights sum to 1.0. Source count has diminishing returns (1=0.5, 2=0.75, 3+=0.9).
- **Unknown**: How were these weights determined? A/B tested or expert judgment?
- **Decision owner**: ML/AI team + product team

### 3.3 Fuzzy Matching Thresholds
- **Location**: `shared/services/dish/matching.py:34-36`
- **What**: DEFAULT_THRESHOLD=60.0 (general matching), STRICT_THRESHOLD=85.0 (deduplication). RapidFuzz 0-100 scale.
- **Evidence**: Methods: exact, partial ratio, token set ratio.
- **Unknown**: What false positive/negative rates do these thresholds produce?
- **Decision owner**: Data quality team

### 3.4 Source Priority (owner > public > user_scan)
- **Location**: `shared/services/dish/dish_service.py:41-54`
- **What**: Source priority for dish data: owner=1 (highest), public=2, user_scan=3. Higher priority source overwrites lower.
- **Evidence**: `SOURCE_PRIORITY` dict determines merge behavior.
- **Unknown**: Should user-contributed data ever override public data if more recent?
- **Decision owner**: Product team

### 3.5 Points Economy
- **Location**: `shared/models/points.py:26-44`
- **What**: 16 point actions with configurable points, daily/lifetime limits. Examples: dish_photo=10pts (daily limit 5), set_origin=2pts (lifetime limit 1), redeem_scan=100pts cost.
- **Evidence**: Admin-configurable via PointConfiguration table.
- **Unknown**: How were point values calibrated? What's the target earn-to-redeem ratio?
- **Decision owner**: Product/growth team

### 3.6 Quest Completion Bonus
- **Location**: `shared/services/quest.py:291`
- **What**: 10 points bonus for completing a daily quest. 48-hour streak grace period.
- **Evidence**: Hardcoded 10-point bonus, 48h window for streak continuity.
- **Unknown**: Does the 10-point bonus effectively drive daily engagement?
- **Decision owner**: Product/growth team

### 3.7 Premium Entitlements Definition
- **Location**: `shared/services/tier_service.py:136-137`
- **What**: `PREMIUM_ENTITLEMENTS = ("pro", "premium")` -- these entitlement identifiers grant premium access.
- **Evidence**: Used in tier determination cascade.
- **Unknown**: Why both "pro" and "premium"? Are they different entitlement levels?
- **Decision owner**: Product team

### 3.8 Admin Tier Override Duration
- **Location**: `services/subscription_server/app/routers/admin.py:1227`
- **What**: Manual tier override expires after 365 days for basic/pro.
- **Evidence**: `expiration_date = now + 365 days` for non-free tiers.
- **Unknown**: Is 365 days appropriate for all override scenarios (customer support, testing)?
- **Decision owner**: Customer support / ops team

### 3.9 Webhook Event Type Mapping
- **Location**: `shared/services/webhook_adapters.py:106-128`
- **What**: Active events (INITIAL_PURCHASE, RENEWAL, REACTIVATION) -> tier="pro". Inactive events (EXPIRATION, CANCELLATION, REFUND) -> tier="free".
- **Evidence**: Unknown event types default to EXPIRATION (safe default).
- **Unknown**: Should unknown events default to EXPIRATION? Could this incorrectly downgrade users?
- **Decision owner**: Subscription/payments team

### 3.10 Image Search Query Building
- **Location**: `services/image_search/app/services/provider_config_service.py:165-196`
- **What**: Per-provider query suffixes (Google: "food dish", Bing: "food dish recipe", Pexels: "food", Unsplash: "food photography").
- **Evidence**: Query source options: AI-optimized query or raw dish name.
- **Unknown**: Have these suffixes been A/B tested for image quality?
- **Decision owner**: Product/design team

### 3.11 Content Personalization Features
- **Location**: `shared/models/streaming_events.py:107-109`
- **What**: Optional enrichment fields: culturalContext, localHistory, pairingSuggestion (controlled by user preferences).
- **Evidence**: Toggle per user via taste profile: show_cultural_context, show_local_history, show_pairing_suggestions.
- **Unknown**: What percentage of users enable each? Impact on parsing latency?
- **Decision owner**: Product team

### 3.12 Restaurant Feedback Model (One-Tap vs Star Rating)
- **Location**: `shared/models/feedback.py:98-108`
- **What**: Simplified one-tap recommendation (RECOMMENDED/NOT_RECOMMENDED/NEUTRAL) replacing traditional 5-star rating.
- **Evidence**: Legacy `overall_rating` and `tags` fields kept nullable for backward compatibility.
- **Unknown**: Has conversion rate improved with one-tap model?
- **Decision owner**: Product/UX team

### 3.13 LLM Provider Regional Routing
- **Location**: `shared/models/llm_config.py:25-39`
- **What**: Region-based provider selection: US, EU, CN, APAC, DEFAULT. Priority-based failover within each region.
- **Evidence**: Supports OpenAI, Anthropic, Google, Azure, Local providers per region.
- **Unknown**: Are regional choices driven by data residency requirements or latency?
- **Decision owner**: Engineering/legal team

### 3.14 Daily Restaurant Review Limit
- **Location**: `tests/unit/test_feedback_service.py`
- **What**: 1 restaurant review per user per day.
- **Evidence**: `DailyLimitExceededError` raised on second review same UTC day.
- **Unknown**: Does this limit frustrate power users who visit multiple restaurants daily?
- **Decision owner**: Product team

---

## Layer 4: Historical / Path-Dependent

Constraints that exist because of legacy systems, backward compatibility, migration cost, or external integration lock-in.

### 4.1 RevenueCat SDK Compatibility Layer
- **Location**: `services/subscription_server/app/schemas/customer_info.py`, `offerings.py`
- **What**: Response schemas mimic RevenueCat API format (SubscriberResponse, PurchasesPackage, PurchasesOffering). Package identifiers use `$rc_` prefix. All timestamps in both ISO string AND milliseconds.
- **Evidence**: Comments reference "forked RevenueCat SDK", package type identifiers match `$rc_monthly`, `$rc_annual` format.
- **Risk if changed**: iOS app depends on exact response format. Breaking changes require app update.
- **Question for domain expert**: Is the RevenueCat SDK fork still maintained? Can we migrate to a native format?

### 4.2 ERROR_MAPPINGS Legacy Shim
- **Location**: `shared/errors/business_errors.py:98-120`
- **What**: Maps legacy error codes to new business error codes. Maintains backward compatibility during error code migration.
- **Evidence**: Dict mapping old error type strings to new BusinessErrorCode enum values.
- **Risk if changed**: Older API clients may depend on legacy error format.
- **Question for domain expert**: Are there still clients using the legacy error format? Can we sunset this?

### 4.3 Fixed Encryption Salt
- **Location**: `shared/utils/encryption.py:75`
- **What**: `salt = b'menumind_llm_config_salt_v1'` -- fixed salt for PBKDF2 key derivation.
- **Evidence**: TODO comment suggests production should use unique salt per key.
- **Risk if changed**: All existing encrypted API keys would need re-encryption with new salt.
- **Question for domain expert**: Is the fixed salt a security risk given the high iteration count? Is a migration planned?

### 4.4 Hardcoded Dev Secret Key
- **Location**: `shared/utils/cdn_url_generator.py:33`
- **What**: `CDN_SIMULATOR_SECRET` defaults to `"dev-secret-key-change-in-prod"`.
- **Evidence**: TODO comment explicitly calls out production security requirement.
- **Risk if changed**: None in production (should be changed). Risk is in forgetting to change it.
- **Question for domain expert**: Is there a deployment check that prevents this default from reaching production?

### 4.5 Facebook API v18.0 Hardcoded
- **Location**: `services/api_gateway/app/auth/oidc_providers.py:105, 125, 141`
- **What**: Facebook Graph API version `v18.0` hardcoded in three places (discovery, token, user info URLs).
- **Evidence**: API version string embedded in URL literals.
- **Risk if changed**: Facebook may deprecate v18.0; need to update all three references.
- **Question for domain expert**: What's the Facebook API deprecation timeline? Should this be configurable?

### 4.6 Mock OIDC Provider
- **Location**: `services/api_gateway/app/auth/oidc_providers.py:224-291`
- **What**: MockOIDCProvider with hardcoded base URL (`http://localhost:8000/api/v1/mock-oidc`), client ID (`bigbite-ios-app`), secret (`mock-client-secret`).
- **Evidence**: Silent fallback to hardcoded mock profile on any JWT decode failure (line 282).
- **Risk if changed**: Development/testing workflow depends on mock provider.
- **Question for domain expert**: Can this be conditionally excluded from production builds?

### 4.7 Premium User Migration Script
- **Location**: `scripts/migrate_premium_users.py:29-31`
- **What**: Hardcoded UUIDs: APP_ID=`8b5875b2...`, PRO_ENTITLEMENT_ID=`d7c701e2...`, PRO_MONTHLY_PRODUCT_ID=`a7521884...`. Sets expiration 365 days from migration.
- **Evidence**: One-time migration script for is_paid=true users to subscription server entitlements.
- **Risk if changed**: Script is likely already run; UUIDs are database references.
- **Question for domain expert**: Has this migration been completed? Can the script be archived?

### 4.8 Deprecated photo_url Fields
- **Location**: `shared/models/menu.py:52-53`, `shared/services/consumption/service.py:132-133`
- **What**: `image_url` and `photo_url` columns kept for backward compatibility, replaced by S3 path (s3_bucket, s3_key).
- **Evidence**: Comments indicate these are deprecated. Migration script verifies backfill.
- **Risk if changed**: Older code paths or external integrations may still read these fields.
- **Question for domain expert**: Are there still consumers of the deprecated URL fields?

### 4.9 Apple offerType Magic Numbers
- **Location**: `services/subscription_server/app/services/customer_service.py:254-259`
- **What**: Hard-coded Apple offer type codes: 1=intro, 2=trial. No enum or constant definition.
- **Evidence**: Apple's convention, but undocumented in our codebase.
- **Risk if changed**: Breaking Apple integration. But should be documented/enumerated.
- **Question for domain expert**: Should these be extracted to named constants for clarity?

---

## Ambiguous

Items that could belong to multiple layers and need clarification.

### A.1 Interval Inference from Product Name
- **Location**: `services/subscription_server/app/services/customer_service.py:529-534`
- **What**: Heuristic that infers subscription interval from product_identifier keywords: "annual"/"year" -> "year", "week" -> "week", default "month".
- **Possible layers**: Historical (naming convention lock-in) or Strategic (product naming strategy)
- **Evidence needed**: Is there a formal product naming convention that guarantees these keywords?

### A.2 Google Scraper Anti-Detection Headers
- **Location**: `services/image_search/app/providers/google_scraper.py:30-43`
- **What**: Browser-like User-Agent, security fetch headers, cookie jar for session continuity.
- **Possible layers**: Operational (reliability technique) or Historical (evolved from blocked requests)
- **Evidence needed**: Is Google scraping officially sanctioned? What's the legal/TOS status?

### A.3 WeChat OIDC Integration
- **Location**: `services/api_gateway/app/auth/oidc_providers.py:154-222`
- **What**: WeChat OAuth provider (Chinese market). Email NOT available from WeChat API.
- **Possible layers**: Strategic (market expansion) or Historical (built for specific customer)
- **Evidence needed**: Is Chinese market expansion still a priority? Does this affect GDPR compliance?

### A.4 Subscription Server "store_transaction_id" Empty
- **Location**: `services/subscription_server/app/services/customer_service.py:166`
- **What**: `store_transaction_id` hardcoded empty string with TODO comment.
- **Possible layers**: Historical (not yet implemented) or Operational (intentionally deferred)
- **Evidence needed**: Is linking to Apple's transaction table planned?

### A.5 Beverage Exclusion from Image Search
- **Location**: `shared/models/menu.py:108`
- **What**: `is_drink` boolean flag skips image search for beverages.
- **Possible layers**: Strategic (product scope decision) or Operational (cost optimization)
- **Evidence needed**: Is this a permanent product decision or temporary cost measure?

---

## Recommendations

### 1. Safe for AI
- **Cryptographic operations**: Encryption, HMAC signing, JWS verification -- pure math, well-tested
- **Points ledger math**: Balance calculations, limit enforcement, atomic redemption
- **Schema validation**: Migration linter, audit column enforcement, soft delete pattern
- **Image processing pipeline**: Deterministic transformations, compression, resizing
- **Dish normalization**: String transformations, cache key generation
- **Test expansion**: Adding unit tests for existing invariants (especially boundary cases)
- **Error code mapping**: Business error code -> HTTP status mapping
- **Data model refactoring**: Non-behavioral changes to models that preserve invariants

### 2. Needs Human Input Before Changing
- **Tier limits** (5/20/unlimited) -- pricing/business decision
- **Confidence scoring weights** -- affects data quality perception
- **Grace period duration** (72h) -- revenue vs UX tradeoff
- **Points economy values** -- affects user engagement and cost
- **Source priority hierarchy** -- affects data trust model
- **Rate limits** -- affects user experience and abuse prevention
- **Circuit breaker thresholds** -- affects availability vs reliability tradeoff
- **RevenueCat compatibility** -- affects iOS app release cycle

### 3. Suggested Questions for Domain Expert

1. **[Tier Limits]** Why specifically 5/20/unlimited scans? Based on user research, competitive analysis, or revenue modeling?

2. **[Grace Period]** The 72-hour grace period matches RevenueCat's default. Has this been validated against actual subscription server downtime patterns? Has a user ever been incorrectly blocked or incorrectly given access due to grace period timing?

3. **[Fixed Encryption Salt]** `menumind_llm_config_salt_v1` is a fixed salt for PBKDF2. Given the 480K iteration count, is this an acceptable security tradeoff? Is there a plan to migrate to per-key salts?

4. **[RevenueCat SDK Fork]** The subscription server maintains exact RevenueCat API compatibility. Is the forked iOS SDK still maintained? What's the migration path to a native API format?

5. **[Circuit Breaker]** The subscription circuit breaker (5 failures/30s) and image search circuit breaker (3 failures/60s) have different configurations. What incidents or load patterns determined these values?

6. **[Confidence Weights]** Dish confidence uses AI=0.35, source_count=0.25, consistency=0.25, quality=0.15. Were these weights determined empirically or by expert judgment? Has their effectiveness been measured?

7. **[CDN URL Expiry Mismatch]** Docker-compose sets URL expiry to 86400s (24h) but the code uses 14400s (4h). Which is the intended production value?

8. **[Google Scraping]** The Google image scraper uses anti-detection headers and browser spoofing. What's the legal/TOS status? Is there a risk of IP blocking in production?

9. **[Points Economy]** How were point values calibrated (e.g., dish_photo=10pts, redeem_scan=100pts)? What's the target earn-to-redeem ratio? Has the economy been modeled for sustainability?

10. **[Deprecated Fields]** Several models have deprecated fields (photo_url, image_url, overall_rating). Are there still active consumers? Can a deprecation timeline be established?

---

## Appendix: Architecture Overview

### Service Portfolio
| Service | Port | Purpose |
|---------|------|---------|
| API Gateway | 8000 | Auth, rate limiting, CORS, proxy |
| Menu Parsing | 8001 | AI-powered menu image parsing |
| Image Search | 8002 | Multi-provider dish image search |
| Image Processing | 8003 | Image optimization & variants |
| CDN Simulator | 8005 | Signed URL serving & edge cache |
| Admin Dashboard | 8006 | Management UI & API |
| Subscription Server | 8007 | Apple IAP, entitlements, webhooks |
| Identity Service | 8010 | Auth, OIDC, user management |
| User Profile | 8011 | Profiles, preferences, devices |
| Usage Service | 8012 | Activity tracking, limits |
| Geo Service | 8013 | Location-based features |

### Key Magic Numbers Summary
| Parameter | Value | Location |
|-----------|-------|----------|
| Free tier scans/month | 5 | activity.py:18-31 |
| Basic tier scans/month | 20 | activity.py:18-31 |
| Grace period | 72 hours | tier_service.py:133 |
| Circuit breaker threshold | 5 failures | settings.py |
| Circuit breaker timeout | 30 seconds | settings.py |
| PBKDF2 iterations | 480,000 | encryption.py:75 |
| CDN URL expiry | 4 hours (code) / 24h (config) | cdn_url_generator.py / docker-compose |
| Image max dimension | 1536 px | image_optimizer.py:14 |
| JPEG quality | 85 | image_optimizer.py:15 |
| Fuzzy match default | 60.0 / 100 | matching.py:34 |
| Fuzzy match strict | 85.0 / 100 | matching.py:35 |
| Confidence weight AI | 0.35 | confidence.py |
| Verification code length | 6 digits | claim_service.py:65 |
| Verification code expiry | 15 minutes | claim_service.py:66 |
| Max verification attempts | 5 | claim_service.py:67 |
| Quest completion bonus | 10 points | quest.py:291 |
| Streak grace period | 48 hours | quest.py:256 |
| Photo freshness TTL | 180 days | restaurant_service.py:37 |
| Webhook retry | 3 attempts (1s, 2s, 4s) | webhook_forwarder.py:62-64 |
| Admin tier override | 365 days | admin.py:1227 |
