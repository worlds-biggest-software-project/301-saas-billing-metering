# Data Model Suggestion 4: Time-Series / Analytics-First

> Project: SaaS Billing & Metering · Created: 2026-05-22

## Philosophy

This model optimizes for the highest-throughput requirement in a billing platform: usage event ingestion and aggregation. It uses TimescaleDB hypertables for time-series usage data, continuous aggregates for real-time billing summaries, and partitioned fact tables for SaaS metrics — all running on PostgreSQL with the TimescaleDB extension. The relational tables for subscriptions, invoices, and payments are conventional; the time-series layer handles the metering pipeline.

The key insight is that a billing platform's performance bottleneck is always in metering, not in subscription CRUD. At 10,000+ events per second, raw usage data grows by hundreds of millions of rows per month. Conventional relational tables with B-tree indexes degrade; monthly aggregation queries slow from seconds to minutes. TimescaleDB solves this by automatically partitioning data into time-based chunks, compressing old chunks by 90-95%, and providing continuous aggregates that maintain pre-computed rollups in real time.

This approach is inspired by OpenMeter (which uses ClickHouse for aggregation) and Orb (which processes 250,000+ events/sec). The difference is that this model keeps everything in PostgreSQL + TimescaleDB, avoiding the operational complexity of a separate analytics database while still achieving the throughput needed for real-time billing.

**Best for:** Platforms handling high-volume usage metering (10,000+ events/sec), where real-time aggregation performance, compression, and retention management are critical. Ideal for AI/API companies billing on tokens, requests, or compute time.

**Trade-offs:**
- **Pro:** Native time-series optimization — automatic chunking, compression, and retention without manual partition management
- **Pro:** Continuous aggregates provide real-time billing summaries without re-scanning raw events
- **Pro:** 90-95% compression on historical event data reduces storage costs dramatically
- **Pro:** Stays in PostgreSQL — no separate analytics database, no data synchronization
- **Con:** Requires TimescaleDB extension (not vanilla PostgreSQL)
- **Con:** Hypertables have limitations: no unique constraints across chunks (dedup requires application logic or separate table)
- **Con:** Continuous aggregates add write overhead (maintained incrementally on each INSERT)
- **Con:** Complex time-series queries may still require careful index design

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0 | Usage events follow CloudEvents structure; type maps to meter code, subject to customer ID |
| ASC 606 / IFRS 15 | Revenue recognition tables are standard relational with immutable journal entries |
| ISO 4217 | Currency codes on all monetary fields |
| ISO 3166 | Country codes for tax jurisdiction in customer records |
| OpenMetrics / Prometheus | Metric naming conventions for usage meters (namespace.metric_name.unit) |
| PCI DSS | Payment tokens only; no PANs stored |
| GAAP SaaS Metrics | MRR fact table as a time-partitioned series for trend analysis |

---

## Usage Metering (TimescaleDB Hypertables)

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ============================================================
-- RAW USAGE EVENTS (high-throughput hypertable)
-- ============================================================

CREATE TABLE usage_events (
    event_timestamp TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    metric_code VARCHAR(100) NOT NULL,
    quantity NUMERIC NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    idempotency_key VARCHAR(255) NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT create_hypertable('usage_events', 'event_timestamp',
    chunk_time_interval => INTERVAL '1 day');

-- Space partitioning for multi-tenant workloads
SELECT add_dimension('usage_events', 'tenant_id', number_partitions => 4);

CREATE INDEX idx_usage_customer ON usage_events(customer_id, metric_code, event_timestamp DESC);
CREATE INDEX idx_usage_tenant ON usage_events(tenant_id, event_timestamp DESC);
CREATE INDEX idx_usage_properties ON usage_events USING GIN (properties jsonb_path_ops);

-- Deduplication table (hypertables can't enforce cross-chunk uniqueness)
CREATE TABLE usage_dedup_keys (
    tenant_id UUID NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL,
    PRIMARY KEY(tenant_id, idempotency_key)
);

-- Compression policy: compress chunks older than 7 days (90-95% size reduction)
SELECT add_compression_policy('usage_events', INTERVAL '7 days');

ALTER TABLE usage_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, customer_id, metric_code',
    timescaledb.compress_orderby = 'event_timestamp'
);

-- Retention policy: drop raw events older than 1 year (aggregates are retained)
SELECT add_retention_policy('usage_events', INTERVAL '1 year');

-- ============================================================
-- CONTINUOUS AGGREGATES (real-time billing summaries)
-- ============================================================

-- Hourly aggregation for near-real-time billing
CREATE MATERIALIZED VIEW usage_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', event_timestamp) AS bucket,
    tenant_id,
    customer_id,
    metric_code,
    COUNT(*) AS event_count,
    SUM(quantity) AS total_quantity,
    AVG(quantity) AS avg_quantity,
    MAX(quantity) AS max_quantity,
    MIN(quantity) AS min_quantity
FROM usage_events
GROUP BY bucket, tenant_id, customer_id, metric_code
WITH NO DATA;

SELECT add_continuous_aggregate_policy('usage_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Daily aggregation for invoice generation
CREATE MATERIALIZED VIEW usage_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', event_timestamp) AS bucket,
    tenant_id,
    customer_id,
    metric_code,
    COUNT(*) AS event_count,
    SUM(quantity) AS total_quantity,
    AVG(quantity) AS avg_quantity,
    MAX(quantity) AS max_quantity
FROM usage_events
GROUP BY bucket, tenant_id, customer_id, metric_code
WITH NO DATA;

SELECT add_continuous_aggregate_policy('usage_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- Monthly aggregation for billing periods
CREATE MATERIALIZED VIEW usage_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', event_timestamp) AS bucket,
    tenant_id,
    customer_id,
    metric_code,
    COUNT(*) AS event_count,
    SUM(quantity) AS total_quantity,
    AVG(quantity) AS avg_quantity,
    MAX(quantity) AS max_quantity
FROM usage_events
GROUP BY bucket, tenant_id, customer_id, metric_code
WITH NO DATA;

SELECT add_continuous_aggregate_policy('usage_monthly',
    start_offset => INTERVAL '3 months',
    end_offset => INTERVAL '1 month',
    schedule_interval => INTERVAL '1 day');

-- ============================================================
-- GROUPED AGGREGATION (e.g., usage by AI model)
-- ============================================================

CREATE MATERIALIZED VIEW usage_daily_by_model
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', event_timestamp) AS bucket,
    tenant_id,
    customer_id,
    metric_code,
    properties->>'model' AS model,
    COUNT(*) AS event_count,
    SUM(quantity) AS total_quantity
FROM usage_events
WHERE metric_code = 'ai_tokens'
GROUP BY bucket, tenant_id, customer_id, metric_code, properties->>'model'
WITH NO DATA;
```

---

## Billing Tables (Relational Core)

```sql
-- ============================================================
-- TENANCY & CUSTOMERS
-- ============================================================

CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(100) NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id VARCHAR(255),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    balance_cents BIGINT NOT NULL DEFAULT 0,
    tax_config JSONB NOT NULL DEFAULT '{}',
    billing_address JSONB,
    payment_methods JSONB NOT NULL DEFAULT '[]',
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, external_id)
);

-- ============================================================
-- CATALOG & PRICING
-- ============================================================

CREATE TABLE billable_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    code VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    aggregation_type VARCHAR(30) NOT NULL,
    field_name VARCHAR(100),
    group_keys TEXT[],
    filters JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NOT NULL,
    billing_interval VARCHAR(20) NOT NULL,
    base_amount_cents BIGINT NOT NULL DEFAULT 0,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    trial_period_days INTEGER NOT NULL DEFAULT 0,
    version INTEGER NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT true,
    charges JSONB NOT NULL DEFAULT '[]',
    entitlements JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code, version)
);

-- ============================================================
-- SUBSCRIPTIONS
-- ============================================================

CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    plan_id UUID NOT NULL REFERENCES plans(id),
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    billing_cycle_anchor TIMESTAMPTZ NOT NULL,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end TIMESTAMPTZ NOT NULL,
    trial_end TIMESTAMPTZ,
    canceled_at TIMESTAMPTZ,
    plan_snapshot JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('trialing','active','past_due','paused','canceled','expired'))
);

CREATE INDEX idx_subs_customer ON subscriptions(customer_id, status);

-- ============================================================
-- INVOICING
-- ============================================================

CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    invoice_number VARCHAR(50) UNIQUE,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    currency CHAR(3) NOT NULL,
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    tax_total_cents BIGINT NOT NULL DEFAULT 0,
    credits_applied_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_due_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT NOT NULL DEFAULT 0,
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    due_date DATE,
    finalized_at TIMESTAMPTZ,
    paid_at TIMESTAMPTZ,
    line_items JSONB NOT NULL DEFAULT '[]',
    tax_breakdown JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('draft','open','paid','void','uncollectible'))
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id, status);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    amount_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    provider VARCHAR(50) NOT NULL,
    provider_payment_id VARCHAR(255),
    failure_info JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('pending','processing','succeeded','failed','refunded'))
);

-- ============================================================
-- CREDIT WALLETS
-- ============================================================

CREATE TABLE wallets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    balance NUMERIC NOT NULL DEFAULT 0,
    expiration_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE wallet_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id UUID NOT NULL REFERENCES wallets(id),
    transaction_type VARCHAR(20) NOT NULL,
    source VARCHAR(50) NOT NULL,
    credit_amount NUMERIC NOT NULL,
    balance_before NUMERIC NOT NULL,
    balance_after NUMERIC NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

REVOKE UPDATE, DELETE ON wallet_transactions FROM PUBLIC;
```

---

## SaaS Metrics (Time-Partitioned)

```sql
-- ============================================================
-- MRR MOVEMENTS (time-series fact table)
-- ============================================================

CREATE TABLE mrr_movements (
    movement_date DATE NOT NULL,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    subscription_id UUID NOT NULL,
    movement_type VARCHAR(20) NOT NULL,
    mrr_delta_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (movement_type IN ('new','expansion','contraction','churn','reactivation'))
);

SELECT create_hypertable('mrr_movements', 'movement_date',
    chunk_time_interval => INTERVAL '1 month');

-- MRR waterfall as a continuous aggregate
CREATE MATERIALIZED VIEW mrr_waterfall_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', movement_date) AS month,
    tenant_id,
    SUM(mrr_delta_cents) FILTER (WHERE movement_type = 'new') AS new_mrr_cents,
    SUM(mrr_delta_cents) FILTER (WHERE movement_type = 'expansion') AS expansion_mrr_cents,
    SUM(ABS(mrr_delta_cents)) FILTER (WHERE movement_type = 'contraction') AS contraction_mrr_cents,
    SUM(ABS(mrr_delta_cents)) FILTER (WHERE movement_type = 'churn') AS churned_mrr_cents,
    SUM(mrr_delta_cents) FILTER (WHERE movement_type = 'reactivation') AS reactivation_mrr_cents
FROM mrr_movements
GROUP BY month, tenant_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('mrr_waterfall_monthly',
    start_offset => INTERVAL '3 months',
    end_offset => INTERVAL '1 month',
    schedule_interval => INTERVAL '1 day');

-- ============================================================
-- REVENUE RECOGNITION
-- ============================================================

CREATE TABLE revenue_journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    contract_id UUID,
    period DATE NOT NULL,
    debit_account VARCHAR(100) NOT NULL,
    credit_account VARCHAR(100) NOT NULL,
    amount_cents BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    posted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('pending','posted','reversed'))
);

REVOKE UPDATE, DELETE ON revenue_journal_entries FROM PUBLIC;

-- ============================================================
-- AUDIT LOG (time-partitioned)
-- ============================================================

CREATE TABLE audit_log (
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    actor_id UUID,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}'
);

SELECT create_hypertable('audit_log', 'occurred_at',
    chunk_time_interval => INTERVAL '1 month');

SELECT add_compression_policy('audit_log', INTERVAL '30 days');
SELECT add_retention_policy('audit_log', INTERVAL '2 years');
```

---

## Example Queries

```sql
-- Real-time usage for current billing period (uses continuous aggregate)
SELECT
    metric_code,
    SUM(total_quantity) AS total_usage,
    SUM(event_count) AS total_events
FROM usage_daily
WHERE customer_id = 'customer-uuid'
  AND bucket >= '2026-05-01'
  AND bucket < '2026-06-01'
GROUP BY metric_code;

-- Usage trend over the last 7 days (hourly granularity)
SELECT
    bucket,
    total_quantity,
    event_count
FROM usage_hourly
WHERE customer_id = 'customer-uuid'
  AND metric_code = 'api_calls'
  AND bucket >= NOW() - INTERVAL '7 days'
ORDER BY bucket;

-- AI token usage by model for billing
SELECT
    model,
    SUM(total_quantity) AS total_tokens,
    SUM(event_count) AS total_requests
FROM usage_daily_by_model
WHERE customer_id = 'customer-uuid'
  AND bucket >= '2026-05-01'
  AND bucket < '2026-06-01'
GROUP BY model
ORDER BY total_tokens DESC;

-- Anomaly detection: customers with 3x their usual daily usage
WITH recent_avg AS (
    SELECT customer_id, metric_code,
           AVG(total_quantity) AS avg_daily
    FROM usage_daily
    WHERE bucket >= NOW() - INTERVAL '30 days'
      AND bucket < NOW() - INTERVAL '1 day'
    GROUP BY customer_id, metric_code
),
today AS (
    SELECT customer_id, metric_code, total_quantity
    FROM usage_daily
    WHERE bucket = date_trunc('day', NOW())
)
SELECT t.customer_id, t.metric_code,
       t.total_quantity AS today_usage,
       ra.avg_daily AS avg_daily_usage,
       t.total_quantity / NULLIF(ra.avg_daily, 0) AS ratio
FROM today t
JOIN recent_avg ra USING (customer_id, metric_code)
WHERE t.total_quantity > ra.avg_daily * 3
ORDER BY ratio DESC;

-- MRR waterfall for the last 12 months
SELECT month, new_mrr_cents, expansion_mrr_cents,
       contraction_mrr_cents, churned_mrr_cents, reactivation_mrr_cents
FROM mrr_waterfall_monthly
WHERE tenant_id = 'tenant-uuid'
ORDER BY month DESC
LIMIT 12;

-- Compression stats
SELECT
    hypertable_name,
    pg_size_pretty(before_compression_total_bytes) AS before,
    pg_size_pretty(after_compression_total_bytes) AS after,
    round(1 - (after_compression_total_bytes::numeric / before_compression_total_bytes), 2) AS ratio
FROM hypertable_compression_stats('usage_events');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Customers | 2 | tenants, customers |
| Catalog & Pricing | 2 | billable_metrics, plans |
| Subscriptions | 1 | subscriptions |
| Usage Metering | 2 | usage_events (hypertable), usage_dedup_keys |
| Continuous Aggregates | 4 | usage_hourly, usage_daily, usage_monthly, usage_daily_by_model |
| Invoicing | 2 | invoices (line items as JSONB), payments |
| Wallets | 2 | wallets, wallet_transactions |
| SaaS Metrics | 1+1 | mrr_movements (hypertable) + mrr_waterfall_monthly (aggregate) |
| Revenue & Audit | 2 | revenue_journal_entries, audit_log (hypertable) |
| **Total** | **18 tables + 5 continuous aggregates** | Optimized for metering throughput |

---

## Key Design Decisions

1. **TimescaleDB hypertables for usage events** with 1-day chunk intervals. This provides automatic time-based partitioning without manual `CREATE TABLE ... PARTITION OF` statements. Space partitioning by `tenant_id` (4 partitions) ensures large tenants don't create hot spots. Chunk intervals are chosen for 10-100M rows per chunk at 10K events/sec.

2. **Compression after 7 days** with `segmentby = 'tenant_id, customer_id, metric_code'` and `orderby = 'event_timestamp'`. This means queries filtering by customer and metric decompress only relevant segments. Typical compression ratios are 90-95%, reducing storage from terabytes to hundreds of gigabytes.

3. **Separate dedup_keys table** because TimescaleDB hypertables cannot enforce unique constraints across chunks. The application does `INSERT INTO usage_dedup_keys ... ON CONFLICT DO NOTHING` first; if the insert succeeds, the event is new and gets inserted into `usage_events`. This is a two-step process but guarantees exactly-once semantics.

4. **Three levels of continuous aggregates** (hourly, daily, monthly) provide pre-computed rollups at different granularities. Invoice generation queries `usage_monthly`; dashboard charts query `usage_hourly`; billing period summaries query `usage_daily`. Each aggregate is maintained incrementally — TimescaleDB only processes new chunks, not the entire dataset.

5. **Model-specific continuous aggregate** (`usage_daily_by_model`) demonstrates dimension-aware aggregation for AI billing. Rather than grouping all token usage together, this splits by `properties->>'model'` to enable per-model billing (GPT-4 at one rate, Claude at another).

6. **MRR movements as a hypertable** rather than a regular table. While MRR movements are low-volume compared to usage events, the hypertable provides automatic chunking, compression, and the ability to use continuous aggregates for the MRR waterfall — a core SaaS dashboard component.

7. **Audit log as a hypertable with compression and retention.** Audit events are append-only, compressed after 30 days, and dropped after 2 years. This provides compliant audit retention with automatic lifecycle management, no manual partition creation.

8. **Invoice line items stored as JSONB** within the invoice row rather than a separate table. In this model, the heavy lifting (aggregation, trend analysis) happens in the time-series layer; the invoice is a finalized output document. Storing line items inline simplifies invoice retrieval to a single-row read. The trade-off is that querying across line items (e.g., "total usage charges across all invoices") requires JSONB extraction, but this is a reporting query that can use the continuous aggregates instead.
