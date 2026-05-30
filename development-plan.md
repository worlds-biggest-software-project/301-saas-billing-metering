# SaaS Billing & Metering — Phased Development Plan

> Project: 301-saas-billing-metering · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased implementation plan. The product is an open, AI-native, usage-based billing and metering engine targeting API-first and AI SaaS companies. It must support high-throughput event ingestion (10,000+ events/sec), hybrid pricing (subscription + usage + add-ons + tiered), invoicing with tax, dunning, ASC 606-aligned audit trails, and a developer-first REST API — deployable both self-hosted and as managed cloud.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | Domain mixes financial/business logic with AI-native features (anomaly detection, NL analytics, pricing recommendations). Python has the strongest LLM/ML ecosystem and excellent decimal/financial libraries, while remaining productive for API work. |
| API framework | **FastAPI** | Async (needed for high-throughput ingestion and webhook fan-out), generates **OpenAPI 3.1** + **JSON Schema 2020-12** automatically (a `standards.md` requirement), Pydantic v2 validation aligns with idempotent, schema-validated billing payloads. |
| Data validation | **Pydantic v2** | Single source of truth for request/response models, config, and event schemas. Powers the CloudEvents envelope validation and OpenAPI generation. |
| Primary database | **PostgreSQL 16** | The canonical billing store. Data model 1 (entity-centric normalized relational) is adopted for referential integrity and structurally-enforced ASC 606 compliance. Money stored as `BIGINT` cents. |
| Time-series layer | **TimescaleDB extension on PostgreSQL** | Adopted from data-model-suggestion-4 for the `usage_events` hypertable + continuous aggregates. Keeps one database engine (no second datastore to operate) while meeting the 10,000+ events/sec MVP throughput target with compression and retention policies. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Mature async ORM; Alembic gives versioned, reviewable migrations (one migration per phase). Raw SQL used for partition/hypertable DDL and aggregation queries. |
| Task queue | **Celery + Redis** | Async workloads are core: invoice generation, dunning retries, webhook delivery, revenue-schedule posting, AI jobs. Redis doubles as broker and rate-limit/dedup cache. Celery Beat handles scheduled billing runs. |
| Cache / dedup / rate limit | **Redis 7** | Sub-second idempotency-key dedup on the ingestion hot path before the DB write; token-bucket rate limiting; meter aggregate cache. |
| Decimal arithmetic | **Python `decimal.Decimal` + integer cents** | All monetary math uses integer cents (per data model 1, key decision 1); `Decimal` used only for rate/percentage intermediate computation, never float. |
| LLM provider | **Provider-agnostic via a thin `LLMClient` abstraction** | AI-native features (anomaly detection narration, NL analytics, contract interpretation) must not lock to one vendor. Default OpenAI-compatible; configurable base URL/model. |
| Payment gateways | **Adapter interface + Stripe and Adyen adapters** | README mandates payment-gateway-agnostic design with ≥2 providers. Adapter pattern isolates provider SDKs behind a `PaymentGateway` protocol. Only tokens stored (PCI scope reduction). |
| Auth | **API keys (hashed) + OAuth 2.1 (PKCE)** | API keys for direct server-to-server integration; OAuth 2.1 for third-party/portal access, per `standards.md`. |
| Frontend (portal/admin) | **Next.js 16 (App Router) + shadcn/ui** | Phase 9 customer self-service portal + admin/pricing UI. Server Components for data-heavy dashboards; talks to the REST API only. Optional — the engine is fully usable headless. |
| SDKs | **Python + TypeScript (generated from OpenAPI)** | Generate clients from the OpenAPI 3.1 spec to keep them in lockstep with the API. Go/Java listed as backlog. |
| Containerisation | **Docker + docker-compose** | Self-hosted is a first-class deployment mode. Compose wires Postgres/TimescaleDB, Redis, API, worker, beat, and (optionally) the portal. |
| Testing | **pytest + pytest-asyncio + testcontainers** | testcontainers spins up real Postgres/Redis for integration tests; `respx`/`responses` mock gateway and LLM HTTP calls. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit** | Fast, single-tool lint/format; strict typing matters in financial code. |
| Package manager | **uv** | Fast, reproducible installs and lockfile; manages the `pyproject.toml` project. |
| Key libraries | `cloudevents`, `httpx`, `tenacity` (retry/backoff), `python-jose` (JWT/OAuth), `weasyprint` (PDF invoices), `babel` (currency/locale), `croniter` (billing schedules) | Each maps to a specific feature: CloudEvents ingestion, async HTTP, dunning/webhook backoff, OAuth tokens, PDF rendering, ISO 4217 formatting, cron-based billing cycles. |

### Project Structure

```
saas-billing-metering/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi/                         # exported OpenAPI 3.1 spec (CI artifact)
│   └── openapi.json
├── migrations/                      # Alembic versions (one+ per phase)
│   └── versions/
├── src/
│   └── billing/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory, router mounting
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db.py                    # async engine/session, base model
│       ├── deps.py                  # FastAPI dependencies (auth, tenant, db)
│       ├── errors.py                # error types + RFC 7807 problem responses
│       ├── money.py                 # cents/Decimal/ISO 4217 helpers
│       ├── models/                  # SQLAlchemy ORM models (by domain)
│       │   ├── tenancy.py
│       │   ├── customer.py
│       │   ├── catalog.py
│       │   ├── subscription.py
│       │   ├── usage.py
│       │   ├── invoice.py
│       │   ├── payment.py
│       │   ├── wallet.py
│       │   ├── entitlement.py
│       │   ├── revenue.py
│       │   ├── metrics.py
│       │   └── audit.py
│       ├── schemas/                 # Pydantic request/response + CloudEvents
│       ├── api/                     # FastAPI routers (one per resource)
│       │   ├── auth.py
│       │   ├── customers.py
│       │   ├── catalog.py
│       │   ├── subscriptions.py
│       │   ├── ingest.py
│       │   ├── usage.py
│       │   ├── invoices.py
│       │   ├── payments.py
│       │   ├── wallets.py
│       │   ├── entitlements.py
│       │   ├── reports.py
│       │   ├── webhooks.py
│       │   └── ai.py
│       ├── services/                # business logic (framework-agnostic)
│       │   ├── metering.py
│       │   ├── pricing.py           # the rating/pricing engine
│       │   ├── subscriptions.py
│       │   ├── invoicing.py
│       │   ├── tax.py
│       │   ├── dunning.py
│       │   ├── wallets.py
│       │   ├── entitlements.py
│       │   ├── revrec.py            # ASC 606 recognition
│       │   ├── metrics.py           # MRR/ARR/churn
│       │   └── audit.py
│       ├── gateways/                # payment gateway adapters
│       │   ├── base.py              # PaymentGateway protocol
│       │   ├── stripe_gw.py
│       │   └── adyen_gw.py
│       ├── ai/                      # AI-native features
│       │   ├── llm.py               # provider-agnostic LLMClient
│       │   ├── anomaly.py
│       │   ├── nl_analytics.py
│       │   └── contract_interp.py
│       ├── workers/                 # Celery tasks + beat schedule
│       │   ├── celery_app.py
│       │   ├── billing_run.py
│       │   ├── dunning_tasks.py
│       │   ├── webhook_delivery.py
│       │   └── revrec_tasks.py
│       └── cli.py                   # admin CLI (Typer): migrate, seed, run-billing
├── portal/                          # Next.js 16 app (Phase 9) — separate workspace
└── tests/
    ├── conftest.py                  # fixtures: db (testcontainers), client, tenant
    ├── fixtures/                    # sample events, plans, invoices (JSON)
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure is grouped by concern (models / schemas / api / services / gateways / ai / workers), so each phase adds files rather than restructuring. The `services/` layer holds all business logic and is independently unit-testable without HTTP or DB where possible.

---

## Phase 1: Foundation — Project Skeleton, Config, Auth & Tenancy

### Purpose
Establish the runnable application skeleton: dependency management, configuration, the async database layer, multi-tenancy, authentication, the error/response contract, and the audit log. Nothing bills yet, but every later phase plugs into these primitives. After this phase a developer can boot the API, authenticate with an API key, and have all requests scoped to a tenant with audit logging.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Create the `uv` project, FastAPI app factory, Docker/compose setup, and CI-ready quality tooling.

**Design**:
- `pyproject.toml` with dependencies from the technology table; tool config for ruff (line length 100), mypy (`strict = true`), pytest.
- `src/billing/main.py` exposes `create_app() -> FastAPI` that mounts routers, registers exception handlers, and adds middleware (request ID, timing). `app = create_app()` for ASGI.
- `src/billing/config.py`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://localhost:6379/0"
      environment: Literal["dev", "test", "prod"] = "dev"
      api_base_url: str = "http://localhost:8000"
      jwt_signing_key: str
      default_currency: str = "USD"
      llm_base_url: str | None = None
      llm_api_key: str | None = None
      llm_model: str = "gpt-4o-mini"
      model_config = SettingsConfigDict(env_prefix="BILLING_", env_file=".env")
  ```
- `docker-compose.yml` services: `db` (timescale/timescaledb-ha:pg16), `redis`, `api`, `worker`, `beat`, `portal` (commented until Phase 9). `Dockerfile` is a multi-stage uv build.
- `src/billing/errors.py`: an `AppError` hierarchy (`NotFoundError`, `ConflictError`, `ValidationError`, `AuthError`, `RateLimitError`) mapped to **RFC 7807** problem+json responses with `type`, `title`, `status`, `detail`, `instance`.

**Testing**:
- `Unit: Settings loads from env with prefix → fields populated, defaults applied`
- `Unit: missing required env (database_url) → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration: unhandled AppError subclass → RFC 7807 body with correct status code`
- `Tooling: ruff check, ruff format --check, mypy src all pass (CI gate)`

#### 1.2 — Async database layer & base model

**What**: SQLAlchemy 2.0 async engine, session dependency, declarative base with shared mixins.

**Design**:
- `db.py`: `engine = create_async_engine(settings.database_url)`, `async_session = async_sessionmaker(...)`, `get_session()` FastAPI dependency yielding a session with commit/rollback handling.
- `Base(DeclarativeBase)` plus mixins:
  ```python
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

  class TenantScopedMixin:
      tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenants.id", ondelete="CASCADE"), index=True)
  ```
- Alembic configured for async; first migration enables `CREATE EXTENSION IF NOT EXISTS timescaledb;` and `pgcrypto` (for `gen_random_uuid`).

**Testing**:
- `Integration (testcontainers): run alembic upgrade head on fresh DB → succeeds, timescaledb extension present`
- `Integration: session dependency commits on success, rolls back on raised exception`
- `Unit: TimestampMixin sets created_at/updated_at on insert/update`

#### 1.3 — Tenancy, users, and API-key authentication

**What**: Implement `tenants`, `users`, `api_keys` tables and API-key auth that scopes every request to a tenant.

**Design**:
- ORM models mirror data-model-1 DDL for `tenants`, `users`, `api_keys` (key stored as SHA-256 `key_hash`, plus `key_prefix` for display, `scopes TEXT[]`).
- API key format: `bk_<env>_<random32>`. On creation return the plaintext once; persist only the hash.
- `deps.py`:
  ```python
  async def current_tenant(authorization: str = Header(...), db=Depends(get_session)) -> Tenant: ...
  def require_scope(scope: str) -> Callable: ...   # dependency factory
  ```
  Auth flow: parse `Authorization: Bearer bk_...` → hash → look up `api_keys` → check `expires_at`, update `last_used_at` → return tenant. Missing/invalid → `AuthError` (401).
- Scopes: `events:write`, `billing:read`, `billing:write`, `admin`.

**Testing**:
- `Unit: api key generation → plaintext returned once, only hash persisted, prefix matches`
- `Integration: valid key → request proceeds, last_used_at updated`
- `Integration: invalid/missing key → 401, RFC 7807 body`
- `Integration: expired key → 401`
- `Integration: key lacking required scope → 403`
- `Integration: tenant A key cannot read tenant B resources (isolation)`

#### 1.4 — Audit log

**What**: Immutable, partitioned `audit_log` plus an `AuditService` invoked by mutating operations.

**Design**:
- `audit_log` per data-model-1 (BIGSERIAL, `RANGE (occurred_at)` partitions, `REVOKE UPDATE, DELETE ... FROM PUBLIC`). Migration creates current + next quarter partitions; a Celery Beat task (Phase 7) rolls partitions forward.
- `AuditService.record(tenant_id, actor, resource_type, resource_id, action, changes: dict, ip)` writes a row; called from service-layer mutations (subscription change, invoice finalize, payment, refund).

**Testing**:
- `Unit: record() serialises changes dict to JSONB, sets actor_type default 'user'`
- `Integration: attempting UPDATE on audit_log row → permission denied (immutability)`
- `Integration: audit row written on a sample mutation, queryable by (resource_type, resource_id)`

---

## Phase 2: Product Catalog & Pricing Configuration

### Purpose
Model what can be sold and how it is priced — products, billable metrics, plans, price components, and tiers — using the relational decomposition from data-model-1. This is the configuration substrate the rating engine (Phase 4) and subscriptions (Phase 3) depend on. After this phase, a tenant can define a hybrid plan (flat base + tiered usage + add-on) entirely through the API.

### Tasks

#### 2.1 — Products & billable metrics

**What**: CRUD for `products` and `billable_metrics`.

**Design**:
- Models per data-model-1. `billable_metrics.aggregation_type ∈ {count, sum, max, min, avg, unique_count, latest}`; `field_name` names the event property to aggregate (null for `count`).
- Endpoints (all tenant-scoped, `billing:write`/`billing:read`):
  - `POST /v1/products`, `GET /v1/products`, `GET/PATCH/DELETE /v1/products/{id}`
  - `POST /v1/metrics`, `GET /v1/metrics`, `GET/PATCH /v1/metrics/{id}` (code immutable once events exist)
- Pydantic `MetricCreate { code, name, description?, aggregation_type, field_name? }` with validator: non-`count` aggregations require `field_name`.

**Testing**:
- `Unit: MetricCreate sum without field_name → ValidationError`
- `Integration: create metric with duplicate (tenant, code) → 409 ConflictError`
- `Integration: create/list/get product happy path → 201/200 with correct shape`

#### 2.2 — Plans, price components & tiers

**What**: CRUD for `plans`, `price_components`, `price_tiers` supporting all pricing models.

**Design**:
- `pricing_model ∈ {flat, per_unit, tiered_graduated, tiered_volume, package, percentage, stairstep}`; `charge_timing ∈ {in_advance, in_arrears}`.
- A plan is created with nested components in one request:
  ```python
  class PriceTierIn(BaseModel):
      up_to: int | None        # null = infinity (last tier)
      unit_amount_cents: int | None
      flat_amount_cents: int | None

  class PriceComponentIn(BaseModel):
      name: str
      billable_metric_code: str | None     # null for pure subscription/flat
      pricing_model: PricingModel
      currency: str = "USD"
      flat_amount_cents: int | None = None
      unit_amount_cents: int | None = None
      package_size: int | None = None
      percentage: Decimal | None = None
      free_units: int = 0
      min_amount_cents: int | None = None
      max_amount_cents: int | None = None
      charge_timing: ChargeTiming = "in_arrears"
      tiers: list[PriceTierIn] = []

  class PlanCreate(BaseModel):
      product_id: UUID
      name: str
      code: str
      billing_interval: Literal["day","week","month","quarter","year"]
      billing_interval_count: int = 1
      base_amount_cents: int = 0
      currency: str = "USD"
      trial_period_days: int = 0
      components: list[PriceComponentIn]
  ```
- **Plan versioning**: editing a plan that has active subscriptions creates a new `version` row (UNIQUE `(tenant, code, version)`); existing subscriptions keep their pinned version. Validators: tiered models require ≥1 tier with exactly one `up_to = null`; tiers must have strictly increasing `up_to`; `percentage` only with `pricing_model='percentage'`.
- Endpoints: `POST /v1/plans`, `GET /v1/plans`, `GET /v1/plans/{id}`, `PATCH /v1/plans/{id}` (→ new version), `POST /v1/plans/{id}/archive`.

**Testing**:
- `Unit: tiered_graduated with non-increasing up_to → ValidationError`
- `Unit: tiered plan with two open-ended (null up_to) tiers → ValidationError`
- `Unit: per_unit component without unit_amount_cents → ValidationError`
- `Integration: create hybrid plan (flat + tiered usage + percentage add-on) → persisted with components/tiers, round-trips on GET`
- `Integration: PATCH plan with active subscription → new version created, old version intact`

---

## Phase 3: Customers & Subscription Lifecycle

### Purpose
Introduce the parties being billed and the contracts they hold. Implements customers, addresses, payment-method references, and the full subscription lifecycle with proration. After this phase, a customer can be subscribed to a plan, upgraded/downgraded with proration, and cancelled — producing the subscription-history records that later feed MRR metrics.

### Tasks

#### 3.1 — Customers, addresses, payment-method records

**What**: CRUD for `customers`, `customer_addresses`, `payment_methods` (token references only).

**Design**:
- Models per data-model-1. `payment_methods` store `provider`, `provider_token`, `card_brand`, `card_last_four`, `card_exp_*` — **never PAN/CVV** (PCI scope reduction noted in `standards.md`).
- `country_code` (ISO 3166 alpha-2) on addresses drives tax jurisdiction (Phase 5). `currency` (ISO 4217) defaults from tenant settings.
- Endpoints: `POST/GET /v1/customers`, `GET/PATCH/DELETE /v1/customers/{id}`, nested `.../addresses`, `.../payment-methods`. `external_id` unique per tenant for upsert-by-external-id.

**Testing**:
- `Unit: payment_method input rejects any field resembling a PAN/CVV (schema has no such fields)`
- `Integration: create customer with external_id, re-POST same external_id → 409 (or idempotent upsert per flag)`
- `Integration: default address/payment-method uniqueness enforced per type`

#### 3.2 — Subscription creation, trials, renewal scheduling

**What**: Create subscriptions with billing-cycle anchoring, trials, and period computation.

**Design**:
- `subscriptions` per data-model-1; `status ∈ {trialing, active, past_due, paused, canceled, expired}`.
- `SubscriptionService.create(customer_id, plan_id, *, start_at=None, trial_override_days=None, payment_method_id=None)`:
  - Compute `billing_cycle_anchor`, `current_period_start/end` from plan `billing_interval` × `billing_interval_count` using `croniter`/`dateutil` relativedelta.
  - If trial: status `trialing`, `trial_start/end` set, first invoice deferred to `trial_end`.
  - Create `subscription_items` from plan components; snapshot the plan `version`.
  - Emit `subscription.created` event (webhook, Phase 8) and a `subscription_history` row (`change_type='new'`, Phase 6).
- Endpoints: `POST /v1/subscriptions`, `GET /v1/subscriptions`, `GET /v1/subscriptions/{id}`.

**Testing**:
- `Unit: monthly plan started 2026-01-15 → period_end 2026-02-15, anchor preserved`
- `Unit: plan with trial_period_days=14 → status trialing, trial_end = start+14d`
- `Integration: create subscription → items created from plan components, history row written`

#### 3.3 — Upgrades, downgrades, cancellation & proration

**What**: Plan changes with proration credits/charges and cancellation flows.

**Design**:
- `SubscriptionService.change_plan(sub_id, new_plan_id, *, proration: Literal["create_prorations","none","always_invoice"]="create_prorations", timing="immediate"|"period_end")`.
- Proration algorithm (in-advance fixed charges):
  ```
  unused = old_base_cents * (seconds_remaining / seconds_in_period)   # credit
  new_charge = new_base_cents * (seconds_remaining / seconds_in_period) # charge
  net = round_half_up(new_charge - unused)
  ```
  Generates `proration_credit` / `proration_charge` line items (applied to next invoice or invoiced immediately).
- `cancel(sub_id, *, at_period_end=True)`: sets `cancel_at_period_end` or `canceled_at`+status `canceled`; writes `subscription_history change_type='churn'`.
- `pause(sub_id)` / `resume(sub_id)` toggle `paused`.
- Endpoints: `POST /v1/subscriptions/{id}/change`, `POST /v1/subscriptions/{id}/cancel`, `POST /v1/subscriptions/{id}/pause`, `POST /v1/subscriptions/{id}/resume`.

**Testing**:
- `Unit: upgrade at 50% through monthly period → credit ≈ half old base, charge ≈ half new base, net correct, half-up rounding`
- `Unit: downgrade timing=period_end → no immediate proration, change effective next period`
- `Unit: cancel at_period_end=True → cancel_at_period_end set, status still active until period end`
- `Integration: change_plan creates proration line items with correct charge_type and period bounds`

---

## Phase 4: Metering Pipeline & Rating Engine (Core Value)

### Purpose
The heart of the product: ingest consumption events at high throughput with deduplication, aggregate them per meter, and rate usage into charges via the pricing engine. This is where the AI-native "real-time, usage-first" positioning is realised. After this phase the system can ingest CloudEvents, compute period aggregates, and price any pricing model into cents.

### Tasks

#### 4.1 — Event ingestion with CloudEvents & idempotent dedup

**What**: High-throughput `POST /v1/events` (single + batch) accepting **CloudEvents 1.0** envelopes, deduplicated and persisted to a TimescaleDB hypertable.

**Design**:
- Adopt TimescaleDB for `usage_events` (from data-model-4): create as a hypertable on `event_timestamp`, with compression after 7 days and a 90-day default retention policy (configurable). Columns per data-model-1 (`tenant_id`, `customer_id`, `metric_code`, `quantity NUMERIC`, `properties JSONB`, `idempotency_key`, `event_timestamp`, `ingested_at`), UNIQUE `(tenant_id, idempotency_key)`.
- Request schema (CloudEvents-aligned):
  ```python
  class UsageEventIn(BaseModel):
      id: str                       # CloudEvents id (→ idempotency_key with source)
      source: str
      type: str                     # maps to metric_code
      subject: str                  # customer external_id or id
      time: datetime
      data: dict                    # must contain quantity + metric properties
  class BatchIngest(BaseModel):
      events: list[UsageEventIn] = Field(max_length=1000)
  ```
- Hot-path dedup: `SETNX` `dedup:{tenant}:{source}:{id}` in Redis (TTL 24h) before insert; DB UNIQUE constraint is the durable backstop. Duplicate → silently accepted (idempotent), counted in response.
- Resolve `subject` → `customer_id`; unknown customer → buffered in a dead-letter set, 202 with `unmatched` count (never 5xx on the hot path).
- Response: `202 {"accepted": N, "duplicates": M, "unmatched": K}`.
- Throughput: batch inserts via `COPY`/`execute_values`; target ≥10,000 events/sec on commodity hardware (load test in 4.4 testing).

**Testing**:
- `Unit: UsageEventIn missing quantity in data → ValidationError`
- `Integration: ingest event → row in hypertable, 202 accepted=1`
- `Integration: same (source,id) twice → second deduplicated, duplicates=1, single row`
- `Integration: batch of 1000 → all inserted; batch of 1001 → 422`
- `Integration: event for unknown subject → unmatched=1, no 5xx`
- `Load (optional, marked slow): 100k events batched → throughput ≥10k/sec asserted`

#### 4.2 — Meter aggregation

**What**: Compute per-customer/metric/period aggregates from raw events into `usage_aggregates`, backed by TimescaleDB continuous aggregates for read performance.

**Design**:
- `MeteringService.aggregate(customer_id, metric_code, period_start, period_end) -> Decimal` dispatches on `aggregation_type`:
  - `sum`/`count`/`max`/`min`/`avg` → SQL aggregate over `quantity`/`properties->>field_name`.
  - `unique_count` → `COUNT(DISTINCT properties->>field_name)`.
  - `latest` → most recent event's value.
- A TimescaleDB **continuous aggregate** maintains hourly rollups per `(tenant, customer, metric_code)`; `usage_aggregates` is the period-aligned materialisation used at invoice time, updated incrementally on ingest (data-model-1 key decision 4).
- `GET /v1/usage` query API: `?customer_id&metric_code&start&end&group_by=hour|day` returns time-bucketed aggregates (real-time usage visibility for the portal).

**Testing**:
- `Unit: sum aggregation over 3 events (10,20,30) → 60`
- `Unit: unique_count over duplicate field values → distinct count`
- `Unit: avg over empty period → 0 (not divide-by-zero)`
- `Integration: ingest then GET /v1/usage grouped by day → correct buckets`
- `Integration: continuous aggregate reflects new events after refresh`

#### 4.3 — Pricing / rating engine

**What**: Pure, deterministic engine that converts a quantity + price component into a charge in cents, for every pricing model.

**Design**:
- `pricing.py` pure functions (no DB, fully unit-testable):
  ```python
  def rate_component(component: PriceComponentSpec, quantity: Decimal) -> RatedCharge: ...
  ```
  Implementations:
  - **flat**: `flat_amount_cents`.
  - **per_unit**: `max(0, quantity - free_units) * unit_amount_cents`.
  - **package**: `ceil(billable_qty / package_size) * unit_amount_cents`.
  - **tiered_graduated**: walk tiers, charge each bracket's portion at its rate + per-tier `flat_amount_cents`.
  - **tiered_volume**: find the single tier containing the total quantity, charge whole quantity at that tier's rate.
  - **stairstep**: charge the matched tier's `flat_amount_cents` (block pricing).
  - **percentage**: `round_half_up(base_amount * percentage / 100)`.
  - Apply `min_amount_cents`/`max_amount_cents` clamps after computation.
- All arithmetic in `Decimal`, final result `round_half_up` to integer cents (banker's rounding configurable per tenant).
- `RatedCharge { amount_cents, breakdown: list[TierBreakdown] }` — breakdown drives invoice line-item transparency.

**Testing** (fixture-driven table tests, the most important suite in the project):
- `Unit: per_unit 100 units @ $0.01, free_units=10 → 90 units → 9000? (compute exact cents)`
- `Unit: tiered_graduated 0–1000@$0.10, 1000+@$0.05, qty=1500 → 1000*10 + 500*5 cents`
- `Unit: tiered_volume same tiers, qty=1500 → 1500 * 5 cents (whole at top tier)`
- `Unit: package size=1000 @ $5, qty=2500 → ceil(2.5)=3 → 1500 cents`
- `Unit: percentage 2.5% of $400.00 → 1000 cents, half-up rounding`
- `Unit: stairstep matched tier flat → exact tier flat amount`
- `Unit: min/max clamps applied (computed below min → min; above max → max)`
- `Unit: free_units exceed quantity → 0`
- `Fixture: golden file pricing_cases.json drives parametrised cases, expected cents exact`

#### 4.4 — Usage-to-charge resolution

**What**: Bridge aggregation and rating: for a subscription's period, produce all usage charges.

**Design**:
- `MeteringService.compute_usage_charges(subscription_id, period_start, period_end) -> list[RatedCharge]`: for each usage `subscription_item`, fetch the aggregate for its metric, call `rate_component`, attach metric/quantity metadata. Returns charges ready for invoice line items (Phase 5).

**Testing**:
- `Integration: subscription with one tiered usage component + ingested events → correct charge with breakdown`
- `Integration: subscription with no usage events → usage charge = 0 (or omitted)`
- `Load (optional): rating 10k subscriptions completes within target window`

---

## Phase 5: Invoicing, Tax & Payments

### Purpose
Turn priced charges into compliant invoices, apply tax and discounts, and collect payment through pluggable gateways. After this phase the system performs the full bill-to-cash flow: generate a draft invoice, finalise it with tax, charge a payment method, and record the result immutably.

### Tasks

#### 5.1 — Invoice generation & lifecycle

**What**: Assemble invoices (`invoices` + `invoice_line_items`) from subscription base charges, usage charges, prorations, and one-offs.

**Design**:
- `invoices.status ∈ {draft, open, paid, void, uncollectible}`; lifecycle `draft → open (finalize) → paid | uncollectible | void`.
- `InvoicingService.generate(subscription_id, period_start, period_end) -> Invoice`:
  1. In-advance fixed charges (base + in_advance components).
  2. In-arrears usage charges via `compute_usage_charges` (4.4).
  3. Proration line items from pending plan changes.
  4. Compute `subtotal_cents`; status `draft`.
- `finalize(invoice_id)`: assigns sequential `invoice_number` (per-tenant counter), applies tax (5.2), discounts/credits (5.3), sets `total_cents`/`amount_due_cents`, status `open`, writes audit + emits `invoice.finalized`. Finalised invoices are immutable (corrections via credit notes).
- Endpoints: `POST /v1/invoices` (manual), `GET /v1/invoices`, `GET /v1/invoices/{id}`, `POST /v1/invoices/{id}/finalize`, `POST /v1/invoices/{id}/void`, `GET /v1/invoices/{id}/pdf`.
- PDF via `weasyprint` from an HTML template (UBL 2.1-aligned line/tax/total structure per data-model-1).

**Testing**:
- `Unit: subtotal = sum(line_items.amount_cents)`
- `Integration: generate invoice for sub with base + usage → draft with correct line items and charge_types`
- `Integration: finalize → invoice_number assigned, status open, immutable thereafter (PATCH rejected)`
- `Integration: GET /pdf → application/pdf, contains invoice_number and total`
- `Integration: invoice_number sequence is gapless per tenant under concurrent finalize`

#### 5.2 — Tax calculation

**What**: Apply tax based on customer jurisdiction and `tax_rates`, producing `invoice_tax_lines`.

**Design**:
- `TaxService.compute(invoice, customer_address) -> list[TaxLine]`: match `tax_rates` by `country_code`/`region_code` effective on invoice date; `tax_type ∈ {vat, gst, sales_tax, hst}`. Support inclusive vs exclusive (`is_inclusive`). `tax_exempt` customers → zero lines. Per-line-item tax to support mixed rates.
- Pluggable `TaxProvider` protocol so an external engine (Avalara/Stripe Tax) can replace the internal rate table later.

**Testing**:
- `Unit: 20% exclusive VAT on 1000 cents → 200 tax, total 1200`
- `Unit: 10% inclusive on 1100 cents → tax 100, taxable 1000`
- `Unit: tax_exempt customer → no tax lines`
- `Unit: no matching jurisdiction → zero tax + warning logged`

#### 5.3 — Discounts, coupons & credit notes

**What**: Apply coupons/discounts and issue credit notes against finalised invoices.

**Design**:
- `coupons`/`customer_coupons` per data-model-1; `discount_type ∈ {fixed_amount, percentage}`, `duration ∈ {forever, once, repeating}`. Discount applied before tax; decrement `times_redeemed`, enforce `max_redemptions`.
- `credit_notes` for post-finalisation corrections: `credit_amount_cents` (applied to balance) and/or `refund_amount_cents` (triggers refund). Endpoint `POST /v1/invoices/{id}/credit-notes`.

**Testing**:
- `Unit: 10% repeating coupon, duration_in_months=3 → applies 3 cycles then stops`
- `Unit: max_redemptions reached → coupon rejected`
- `Integration: credit note with refund_amount → refund record created (gateway mocked)`

#### 5.4 — Payment gateway adapters & charging

**What**: `PaymentGateway` protocol with Stripe and Adyen adapters; charge invoices and record `payments`/`refunds`.

**Design**:
- `gateways/base.py`:
  ```python
  class PaymentGateway(Protocol):
      async def charge(self, *, amount_cents: int, currency: str, token: str, idempotency_key: str) -> ChargeResult: ...
      async def refund(self, *, provider_payment_id: str, amount_cents: int) -> RefundResult: ...
      def verify_webhook(self, payload: bytes, signature: str) -> dict: ...
  ```
- `ChargeResult { status: Literal["succeeded","failed"], provider_payment_id, failure_code?, failure_message? }`.
- `PaymentService.pay_invoice(invoice_id)`: pick default payment method, call gateway with an idempotency key (`pay:{invoice_id}:{attempt}`), record `payments` row (status mapped), on success set invoice `paid`/`amount_paid_cents`, on failure leave `open` and hand off to dunning (Phase 7).
- Inbound gateway webhooks (`POST /v1/gateways/{provider}/webhook`) verify signatures, reconcile async charge results.

**Testing**:
- `Integration (mocked Stripe via respx): successful charge → payment succeeded, invoice paid`
- `Integration (mocked): card_declined → payment failed with failure_code, invoice stays open`
- `Integration: same idempotency_key charged twice → single provider call / single payment`
- `Integration: gateway webhook with invalid signature → 401, no state change`
- `Integration: Adyen adapter conforms to PaymentGateway protocol (contract test reused across adapters)`

---

## Phase 6: SaaS Metrics & Revenue Recognition (ASC 606)

### Purpose
Make the engine the authoritative source for financial truth: MRR/ARR/churn movement tracking and ASC 606/IFRS 15-aligned revenue recognition with immutable journal entries. After this phase, finance/RevOps teams get accurate recurring-revenue metrics and auditable revenue schedules — a key differentiator versus usage-only tools.

### Tasks

#### 6.1 — Subscription history & MRR movements

**What**: SCD Type 2 `subscription_history` and `mrr_movements` fact table, populated on every subscription change.

**Design**:
- On create/upgrade/downgrade/cancel/reactivate, `MetricsService` closes the current `subscription_history` row (`effective_to`, `is_current=false`) and opens a new one with computed `mrr_amount_cents` (normalise plan price to monthly). It writes an `mrr_movements` row with `movement_type ∈ {new, expansion, contraction, churn, reactivation}` and `mrr_delta_cents`.
- MRR normalisation: yearly→/12, quarterly→/3, weekly→×52/12, etc., rounded half-up.

**Testing**:
- `Unit: $1200/yr plan → MRR 10000 cents`
- `Unit: upgrade $100→$150/mo → expansion movement +5000`
- `Unit: cancel → churn movement = -current MRR`
- `Integration: change_plan writes exactly one history close + one open + one movement`

#### 6.2 — MRR snapshots & reporting endpoints

**What**: Daily `mrr_snapshots` rollup and reporting API for ARR/MRR/NRR/churn and revenue waterfall.

**Design**:
- Celery Beat nightly task aggregates `mrr_movements` into `mrr_snapshots` (total + new/expansion/contraction/churn/reactivation, active subs/customers).
- `ReportsService` computes ARR (= MRR×12), gross/net churn, NRR over a window.
- Endpoints: `GET /v1/reports/mrr?from&to`, `/reports/arr`, `/reports/waterfall?from&to` (beginning → new → expansion → contraction → churn → ending), `GET /v1/reports/export?format=csv|pdf`.

**Testing**:
- `Unit: NRR = (start_mrr + expansion - contraction - churn) / start_mrr`
- `Integration: waterfall components sum from beginning to ending balance`
- `Integration: export?format=csv → valid CSV with expected columns`

#### 6.3 — Revenue recognition (ASC 606 / IFRS 15)

**What**: Contracts, performance obligations, recognition schedules, and immutable journal entries.

**Design**:
- Tables per data-model-1 (`contracts`, `performance_obligations`, `recognition_schedules`, `revenue_journal_entries` with UPDATE/DELETE revoked).
- On subscription create, `RevRecService.create_contract(subscription)` builds a contract + obligations:
  - Subscription base → `over_time`, `straight_line` recognition across the term.
  - Usage → `usage_based`, recognised as consumed.
- `generate_schedule(obligation)` produces monthly `recognition_schedules` rows; a Beat task `recognize_due()` marks due periods recognised and posts balanced `revenue_journal_entries` (deferred revenue → recognised revenue).
- **Predictive ASC 606 risk** (AI-native hook, full impl Phase 8): flag obligations where billing pattern diverges from recognition schedule.
- Endpoints: `GET /v1/revenue/schedule?from&to`, `GET /v1/revenue/contracts/{id}`.

**Testing**:
- `Unit: $1200 annual straight-line → 12 monthly schedule rows of 100 each, remainder on final period`
- `Unit: journal entries balance (debit total == credit total)`
- `Integration: recognize_due posts entries only for elapsed periods; future periods untouched`
- `Integration: UPDATE/DELETE on revenue_journal_entries → permission denied`

---

## Phase 7: Async Engine — Billing Runs, Dunning & Scheduling

### Purpose
Automate the recurring machinery: scheduled billing runs that close periods and generate invoices, intelligent dunning for failed payments, and partition/snapshot maintenance. After this phase the system runs unattended — invoices are produced on schedule and failed payments are retried with backoff.

### Tasks

#### 7.1 — Celery infrastructure & billing run

**What**: Celery app, Beat schedule, and the periodic billing run that invoices due subscriptions.

**Design**:
- `workers/celery_app.py` (Redis broker/result backend). Beat schedule: `run_billing_cycle` hourly (idempotent — processes only subscriptions whose `current_period_end <= now`), `roll_partitions` daily, `snapshot_mrr` nightly, `recognize_revenue` nightly, `process_dunning` every 15 min.
- `run_billing_cycle`: for each due subscription → `InvoicingService.generate` → `finalize` → if `charge_automatically`, enqueue `pay_invoice`; advance `current_period_start/end`; emit events. Wrapped in a per-subscription advisory lock to prevent double-billing.

**Testing**:
- `Integration: subscription past period_end → billing run generates+finalizes invoice, advances period`
- `Integration: billing run run twice → no duplicate invoice for the same period (idempotent)`
- `Integration: subscription not yet due → skipped`

#### 7.2 — Dunning & retry logic

**What**: `dunning_attempts`-driven retry of failed payments with configurable backoff.

**Design**:
- On payment failure, `DunningService.schedule(invoice)` creates `dunning_attempts` with `scheduled_at` per a retry policy (e.g. +1d, +3d, +5d, +7d; configurable per tenant). `process_dunning` task picks due attempts, re-charges, records `result`, sets `next_retry_at` or marks invoice `uncollectible` after final attempt → subscription `past_due`.
- **Intelligent dunning hook**: an optional `LLMClient`/heuristic chooses retry timing from customer history (full AI version Phase 8); deterministic fallback by default.

**Testing**:
- `Unit: retry policy [1,3,5,7] → next_retry_at offsets correct`
- `Integration: failed payment → dunning attempt scheduled; process retries; success → invoice paid, remaining attempts cancelled`
- `Integration: all attempts exhausted → invoice uncollectible, subscription past_due`

#### 7.3 — Maintenance tasks

**What**: Partition rolling, snapshot generation, retention.

**Design**:
- `roll_partitions`: ensures next-quarter partitions exist for `audit_log`; TimescaleDB manages `usage_events` chunks via retention/compression policies.
- Wire `snapshot_mrr`, `recognize_revenue` (Phase 6) into Beat.

**Testing**:
- `Integration: roll_partitions creates upcoming partition idempotently`
- `Integration: retention policy drops chunks older than configured window (test with short window)`

---

## Phase 8: Webhooks & AI-Native Features

### Purpose
Deliver the outbound integration surface (webhooks) and the AI-native differentiators that set this product apart: anomaly detection, natural-language analytics, AI-assisted contract interpretation, and predictive revenue-recognition risk. After this phase the platform actively surfaces insight, not just records.

### Tasks

#### 8.1 — Webhook delivery

**What**: `webhook_endpoints`/`webhook_deliveries` with HMAC-signed, retried delivery.

**Design**:
- Tables per data-model-1. Event taxonomy: `invoice.finalized`, `invoice.paid`, `payment.failed`, `subscription.created|updated|canceled`, `usage.anomaly_detected`, etc.
- `deliver_webhook` Celery task: POST payload with `X-Billing-Signature: sha256=HMAC(secret, body)`; retry with exponential backoff via `tenacity` up to N attempts; record status/attempts.
- Endpoints: CRUD `/v1/webhook-endpoints`; `POST /v1/webhook-endpoints/{id}/test`.

**Testing**:
- `Unit: signature = HMAC-SHA256(secret, body); verifiable by receiver`
- `Integration (mock receiver): event → delivery succeeds, status delivered`
- `Integration: receiver 500 → retried with backoff, marked failed after max attempts`

#### 8.2 — Anomaly detection

**What**: Detect usage spikes, billing errors, and potential fraud before invoicing.

**Design**:
- `ai/anomaly.py`: statistical baseline per `(customer, metric)` (rolling mean/stddev, MAD); flag points beyond threshold (e.g. z-score > 3) or sudden >X% deltas. Optional `LLMClient` narration of why a spike is suspicious.
- Runs in the billing run pre-finalisation and on a schedule; flags emit `usage.anomaly_detected` webhooks and create alert records.
- Endpoint: `GET /v1/anomalies?customer_id&status`.

**Testing**:
- `Unit: stable series + one 10x spike → spike flagged, others not`
- `Unit: gradual ramp within threshold → no false positive`
- `Integration: anomaly during billing run → webhook emitted, invoice flagged for review`

#### 8.3 — Natural-language analytics

**What**: Conversational querying of billing/revenue data without SQL.

**Design**:
- `ai/nl_analytics.py`: `ask(tenant_id, question: str) -> AnswerWithData`. LLM translates the question into a **constrained, parameterised** query over a whitelisted set of metrics/views (never raw SQL execution from the model — the model selects from a catalog of safe query templates + filters). Returns the answer plus the structured data used (auditability).
- Endpoint: `POST /v1/ai/ask { question }`. Strict tenant scoping on every generated query.

**Testing**:
- `Unit: "What was MRR last month?" → maps to mrr snapshot template with correct date range`
- `Unit: question requesting cross-tenant data → refused / scoped to caller tenant only`
- `Integration (mocked LLM): returns answer + the structured rows it was derived from`

#### 8.4 — AI-assisted contract interpretation & predictive rev-rec

**What**: Map free-text deal terms into billing config; flag ASC 606 risks in real time.

**Design**:
- `ai/contract_interp.py`: `interpret(text) -> ProposedPlanConfig` — LLM extracts terms (base fee, included units, overage rates, minimums, term) into a `PlanCreate` draft returned for human review (never auto-applied). 
- Predictive rev-rec: extend 6.3's risk flag with an LLM/heuristic that, as usage arrives, projects whether recognised revenue will diverge from the schedule and emits `revenue.risk_flagged`.
- Endpoints: `POST /v1/ai/interpret-contract { text } -> draft plan`, surfaced risks in `GET /v1/revenue/risks`.

**Testing**:
- `Unit (mocked LLM): sample contract text → ProposedPlanConfig with extracted base + overage tier`
- `Unit: malformed/low-confidence extraction → flagged for review, never auto-created`
- `Integration: usage diverging from schedule → revenue.risk_flagged webhook`

---

## Phase 9: Entitlements, Wallets & Customer/Admin Portal

### Purpose
Round out v1.1 capabilities: prepaid credit wallets (key for AI/token pricing), entitlement management (feature gating separate from billing), and the Next.js portal for self-service plus admin/pricing configuration. After this phase non-engineers can configure pricing and customers can self-serve.

### Tasks

#### 9.1 — Credit wallets

**What**: `wallets`/`wallet_transactions` with top-ups, consumption, expiration, rollover.

**Design**:
- Tables per data-model-1. `wallet_transactions` record `balance_before`/`balance_after` (self-auditing chain). Consumption on usage rating deducts from wallets by `priority` before generating cash charges; `source ∈ {purchased, granted, usage_deduction, voided, expired, refund}`.
- Expiration Beat task voids expired credits with an `expired` transaction.
- Endpoints: `POST /v1/wallets`, `POST /v1/wallets/{id}/top-up`, `GET /v1/wallets/{id}/transactions`.

**Testing**:
- `Unit: top-up 1000 then consume 300 → balance 700, balance_before/after chain consistent`
- `Unit: consume more than balance → partial deduction + cash charge for remainder`
- `Integration: expired credits → expired transaction, balance reduced`

#### 9.2 — Entitlements

**What**: `features`/`plan_entitlements`/`customer_entitlement_overrides`/`entitlement_usage` with a fast check endpoint.

**Design**:
- Tables per data-model-1; `feature_type ∈ {boolean, numeric_limit, metered}`. Resolution order: customer override → plan entitlement → default deny.
- `GET /v1/entitlements/check?customer_id&feature_key` → `{ has_access, limit?, used?, remaining? }`, Redis-cached for sub-10ms checks. Metered features increment `entitlement_usage` and enforce/soft-warn at limit.

**Testing**:
- `Unit: override grants access where plan denies → has_access true`
- `Unit: numeric_limit at limit → remaining 0, soft vs hard limit honored`
- `Integration: check endpoint reflects plan entitlements and cache invalidation on plan change`

#### 9.3 — Customer & admin portal (Next.js 16)

**What**: Self-service portal (usage, invoices, plan changes) + admin UI (no-code pricing config, dashboards, NL analytics).

**Design**:
- Next.js 16 App Router + shadcn/ui in `portal/`, talking only to the REST API. Server Components fetch dashboards (MRR, usage, anomalies); OAuth 2.1 for customer login.
- Pages: customer (`/usage`, `/invoices`, `/plan`), admin (`/pricing` no-code plan builder mapping to PlanCreate, `/customers`, `/reports`, `/ask` NL analytics chat, `/anomalies`).

**Testing**:
- `Component: plan builder produces a valid PlanCreate payload`
- `E2E (Playwright, mocked API): customer views usage chart and downloads invoice PDF`
- `E2E: admin creates a tiered plan via no-code builder → POST /v1/plans called with correct body`

---

## Phase 10: Hardening — SDKs, OpenAPI, Compliance & Deployment

### Purpose
Make the platform production- and integration-ready: published OpenAPI spec and generated SDKs, security/compliance posture (PCI scope, GDPR, OAuth 2.1, rate limiting), observability, and finalized self-hosted + managed deployment artifacts.

### Tasks

#### 10.1 — OpenAPI spec & generated SDKs

**What**: Export the OpenAPI 3.1 document and generate Python + TypeScript clients.

**Design**:
- CI step dumps `app.openapi()` to `openapi/openapi.json`; generate clients (openapi-generator / `openapi-ts`) into `sdks/python` and `sdks/typescript`. Contract test asserts the spec validates against OpenAPI 3.1 and JSON Schema 2020-12.

**Testing**:
- `Integration: exported spec is valid OpenAPI 3.1 (schema validation)`
- `Integration: generated Python SDK ingests an event against a running API (smoke)`

#### 10.2 — Security, compliance & rate limiting

**What**: OAuth 2.1 flows, Redis token-bucket rate limiting, GDPR data export/erasure, encryption-at-rest config.

**Design**:
- OAuth 2.1 authorization-code + PKCE for third-party/portal; scopes mirror API-key scopes.
- Rate limiter dependency (token bucket per API key; ingestion limit configurable, default 10k/sec) returning `429` + `Retry-After` on exhaustion.
- GDPR: `GET /v1/customers/{id}/export` (portable JSON) and `DELETE /v1/customers/{id}` (erasure with billing-record retention rules). All traffic HTTPS-only; DB encryption-at-rest documented in deployment.

**Testing**:
- `Integration: OAuth code+PKCE flow issues a scoped token; missing scope → 403`
- `Integration: exceed rate limit → 429 with Retry-After`
- `Integration: customer export returns complete portable record; erasure removes PII while retaining immutable financial records`

#### 10.3 — Observability & deployment

**What**: Structured logging, OpenTelemetry traces/metrics, finalized Docker/compose + deployment docs.

**Design**:
- Structured JSON logs with request/tenant IDs; OpenTelemetry instrumentation for API + workers (ingestion throughput, billing-run duration, gateway latency). `/metrics` Prometheus endpoint.
- Finalized multi-service `docker-compose.yml` (db, redis, api, worker, beat, portal) and a managed-cloud deployment note (managed Postgres/TimescaleDB + Redis).

**Testing**:
- `Integration: docker compose up → all services healthy, end-to-end ingest→invoice→pay smoke passes`
- `Integration: /metrics exposes ingestion and billing-run counters`
- `E2E: full lifecycle on real Postgres/Redis (testcontainers): define plan → subscribe → ingest usage → run billing → finalize → pay → metrics/revrec updated`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, tenancy, db, audit)        ─── required by everything
    │
Phase 2: Catalog & Pricing Config                     ─── requires 1
    │
Phase 3: Customers & Subscriptions                    ─── requires 2
    │
Phase 4: Metering Pipeline & Rating Engine (CORE)     ─── requires 2, 3
    │
Phase 5: Invoicing, Tax & Payments                    ─── requires 4
    ├── Phase 6: SaaS Metrics & Rev-Rec (ASC 606)     ─── requires 5; can parallel with 7
    └── Phase 7: Async Engine (billing runs, dunning) ─── requires 5; can parallel with 6
         │
Phase 8: Webhooks & AI-Native Features                ─── requires 5, 6, 7
    ├── Phase 9: Entitlements, Wallets & Portal       ─── requires 5 (wallets need rating from 4); can parallel with 8
    └── Phase 10: Hardening, SDKs, Compliance, Deploy ─── requires all; finalize last
```

**Parallelism opportunities:**
- After Phase 5: **Phase 6 (metrics/rev-rec)** and **Phase 7 (async/dunning)** can be built concurrently — they share no code beyond the Celery app introduced in 7.1 (build 7.1 first, then split).
- After Phase 7: **Phase 8 (webhooks/AI)** and **Phase 9 (entitlements/wallets/portal)** can be built concurrently by separate developers.
- Within Phase 4, the **rating engine (4.3)** is pure and can be developed in parallel with the **ingestion pipeline (4.1/4.2)**.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented as specified.
2. All unit and integration tests for the phase pass; the cumulative suite remains green.
3. `ruff check` and `ruff format --check` pass with no violations.
4. `mypy src` passes in strict mode (no new ignores without justification).
5. `alembic upgrade head` applies cleanly on a fresh database, and a matching downgrade exists.
6. The Docker image builds and `docker compose up` brings the affected services to a healthy state.
7. The phase's feature works end-to-end against a real Postgres/Redis (testcontainers e2e where applicable).
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec with request/response schemas.
9. New configuration options are added to `.env.example` and documented.
10. Mutating operations write audit-log entries; financial records remain immutable where required.
11. All monetary calculations use integer cents / `Decimal` (no floats) and are covered by exact-value tests.
```
