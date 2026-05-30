# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: SaaS Billing & Metering · Created: 2026-05-22

## Philosophy

This model uses relational tables for structural entities (tenants, customers, subscriptions, invoices, payments) and JSONB columns for variable, type-specific, or rapidly-evolving data (pricing configurations, usage event properties, meter definitions, entitlement rules). The core skeleton is relational with full foreign keys; the flesh is JSONB with GIN indexes.

A billing platform must support an ever-growing variety of pricing models (flat, per-unit, tiered graduated, tiered volume, package, percentage, stairstep, matrix/dimensional), usage event shapes (API calls with model and token counts, storage with GB and region, compute with CPU-hours and instance type), and entitlement configurations (boolean flags, numeric limits, metered features). A fully normalized model requires schema migrations for every new pricing variant; a fully document-oriented model loses the structural guarantees needed for financial integrity. The hybrid approach provides relational integrity for the money path (invoices, payments, ledger) with JSONB flexibility for the configuration path (plans, pricing, events).

This is the approach used by Lago (open-source billing) and Orb, where plan charges store their pricing configuration as structured JSON while customer-invoice relationships are strictly relational. It enables the fastest path to MVP while preserving the ability to enforce critical financial constraints.

**Best for:** Rapid MVP development, platforms supporting many pricing models without schema migrations, teams wanting relational integrity for financial data with flexibility for pricing configuration.

**Trade-offs:**
- **Pro:** Adding a new pricing model requires no schema migration — just a new JSONB shape validated by JSON Schema
- **Pro:** Usage event properties are flexible — bill on any dimension without pre-defining columns
- **Pro:** Fewer tables than the normalized model (~22 vs ~41), reducing migration complexity
- **Pro:** Financial tables (invoices, payments, ledger) have full referential integrity
- **Con:** Pricing configuration within JSONB cannot have foreign key constraints
- **Con:** Complex JSONB queries can be slower than equivalent relational joins
- **Con:** JSONB schema evolution requires application-level migration logic
- **Con:** Less self-documenting than a fully normalized schema for pricing rules

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Revenue recognition tables are fully relational with immutable journal entries |
| CloudEvents v1.0 | Usage event structure follows CloudEvents envelope; properties stored as JSONB data field |
| ISO 4217 | Currency codes as CHAR(3) on all monetary fields |
| ISO 3166 | Country codes in customer address JSONB |
| UBL 2.1 | Invoice structure aligns with UBL patterns; line items are relational |
| PCI DSS | Payment methods store only tokens in relational columns |
| JSON Schema 2020-12 | JSONB pricing configurations validated against registered JSON Schemas |
| GAAP SaaS Metrics | MRR movements as relational fact table for accurate metric computation |

---

## Multi-Tenancy & Customers

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings: {"max_customers": 1000, "features": ["pricing_experiments", "revenue_recognition"],
    --            "webhook_signing_secret": "whsec_...", "default_currency": "USD"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'member',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, email)
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
    -- tax_config: {"tax_id": "DE123456789", "tax_exempt": false, "vat_number": "DE123456789",
    --              "reverse_charge": false, "nexus_jurisdictions": ["US-CA", "US-NY", "DE"]}
    billing_address JSONB,
    -- billing_address: {"line1": "123 Main St", "city": "San Francisco", "state": "CA",
    --                   "postal_code": "94105", "country_code": "US"}
    shipping_address JSONB,
    payment_methods JSONB NOT NULL DEFAULT '[]',
    -- payment_methods: [{"id": "pm_...", "type": "card", "provider": "stripe",
    --                    "token": "tok_...", "brand": "visa", "last_four": "4242",
    --                    "exp_month": 12, "exp_year": 2028, "is_default": true}]
    metadata JSONB NOT NULL DEFAULT '{}',
    auto_collection BOOLEAN NOT NULL DEFAULT true,
    net_term_days INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);

ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_customers ON customers
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

---

## Product Catalog & Pricing (JSONB-Flexible)

```sql
CREATE TABLE billable_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    code VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    aggregation_type VARCHAR(30) NOT NULL,
    field_name VARCHAR(100),
    group_keys TEXT[],
    filters JSONB NOT NULL DEFAULT '{}',
    -- filters: {"model": ["gpt-4", "claude-3"], "region": ["us-east-1", "eu-west-1"]}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code),
    CHECK (aggregation_type IN ('count','sum','max','min','avg','unique_count','latest','weighted_sum'))
);

CREATE TABLE plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NOT NULL,
    description TEXT,
    billing_interval VARCHAR(20) NOT NULL,
    billing_interval_count INTEGER NOT NULL DEFAULT 1,
    base_amount_cents BIGINT NOT NULL DEFAULT 0,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    trial_period_days INTEGER NOT NULL DEFAULT 0,
    version INTEGER NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT true,
    charges JSONB NOT NULL DEFAULT '[]',
    /*
        charges stores all pricing components as a JSONB array:
        [
            {
                "metric_code": "api_calls",
                "charge_model": "tiered_graduated",
                "charge_timing": "in_arrears",
                "min_amount_cents": 0,
                "tiers": [
                    {"up_to": 1000, "unit_amount_cents": 0, "flat_amount_cents": 0},
                    {"up_to": 10000, "unit_amount_cents": 5, "flat_amount_cents": 0},
                    {"up_to": null, "unit_amount_cents": 3, "flat_amount_cents": 0}
                ]
            },
            {
                "metric_code": "storage_gb",
                "charge_model": "per_unit",
                "charge_timing": "in_arrears",
                "unit_amount_cents": 25,
                "free_units": 10
            },
            {
                "metric_code": null,
                "charge_model": "flat",
                "charge_timing": "in_advance",
                "flat_amount_cents": 9900
            },
            {
                "metric_code": "data_transfer_gb",
                "charge_model": "percentage",
                "charge_timing": "in_arrears",
                "percentage": 2.5,
                "fixed_amount_cents": 50,
                "free_units": 100
            },
            {
                "metric_code": "ai_tokens",
                "charge_model": "package",
                "charge_timing": "in_arrears",
                "package_size": 1000,
                "package_amount_cents": 100
            }
        ]
    */
    entitlements JSONB NOT NULL DEFAULT '[]',
    /*
        entitlements: [
            {"feature_key": "advanced_analytics", "type": "boolean", "has_access": true},
            {"feature_key": "api_rate_limit", "type": "numeric_limit", "limit": 10000, "reset_period": "monthly"},
            {"feature_key": "seats", "type": "numeric_limit", "limit": 50, "soft_limit": false}
        ]
    */
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code, version),
    CHECK (billing_interval IN ('day','week','month','quarter','year'))
);
```

---

## Subscriptions

```sql
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    plan_id UUID NOT NULL REFERENCES plans(id),
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    billing_cycle_anchor TIMESTAMPTZ NOT NULL,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end TIMESTAMPTZ NOT NULL,
    trial_start TIMESTAMPTZ,
    trial_end TIMESTAMPTZ,
    canceled_at TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
    auto_renew BOOLEAN NOT NULL DEFAULT true,
    plan_snapshot JSONB NOT NULL,
    -- Snapshot of the plan + charges at subscription creation time
    -- so price changes don't retroactively affect existing subscriptions
    proration_behavior VARCHAR(30) NOT NULL DEFAULT 'create_prorations',
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('trialing','active','past_due','paused','canceled','expired')),
    CHECK (proration_behavior IN ('create_prorations','none','always_invoice'))
);

CREATE INDEX idx_subscriptions_customer ON subscriptions(customer_id, status);
CREATE INDEX idx_subscriptions_tenant ON subscriptions(tenant_id, status);
CREATE INDEX idx_subscriptions_period ON subscriptions(current_period_end) WHERE status = 'active';

ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_subscriptions ON subscriptions
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

---

## Usage Metering

```sql
CREATE TABLE usage_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    metric_code VARCHAR(100) NOT NULL,
    quantity NUMERIC NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties carries arbitrary dimensions for flexible billing:
    -- {"model": "gpt-4", "tokens_input": 500, "tokens_output": 1500, "region": "us-east-1"}
    -- {"storage_type": "hot", "gb": 42.5, "bucket": "prod-assets"}
    -- {"api_endpoint": "/v1/chat/completions", "status_code": 200, "latency_ms": 230}
    idempotency_key VARCHAR(255) NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, idempotency_key)
) PARTITION BY RANGE (event_timestamp);

CREATE TABLE usage_events_2026_q2 PARTITION OF usage_events
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE usage_events_2026_q3 PARTITION OF usage_events
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

CREATE INDEX idx_usage_customer_metric ON usage_events(customer_id, metric_code, event_timestamp);
CREATE INDEX idx_usage_properties ON usage_events USING GIN (properties jsonb_path_ops);

-- Pre-computed aggregates for billing periods
CREATE TABLE usage_aggregates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    metric_code VARCHAR(100) NOT NULL,
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    aggregated_value NUMERIC NOT NULL DEFAULT 0,
    event_count BIGINT NOT NULL DEFAULT 0,
    group_values JSONB NOT NULL DEFAULT '{}',
    -- group_values: {"model": "gpt-4"} for per-group aggregation
    last_aggregated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(customer_id, metric_code, subscription_id, period_start, group_values)
);
```

---

## Invoicing & Payments

```sql
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
    discount_total_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_due_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT NOT NULL DEFAULT 0,
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    due_date DATE,
    finalized_at TIMESTAMPTZ,
    paid_at TIMESTAMPTZ,
    voided_at TIMESTAMPTZ,
    collection_method VARCHAR(30) NOT NULL DEFAULT 'charge_automatically',
    tax_breakdown JSONB NOT NULL DEFAULT '[]',
    -- tax_breakdown: [{"jurisdiction": "US-CA", "tax_type": "sales_tax", "rate": 8.25,
    --                  "taxable_cents": 5000, "tax_cents": 413}]
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('draft','open','paid','void','uncollectible'))
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id, status);
CREATE INDEX idx_invoices_tenant ON invoices(tenant_id, status);

CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    charge_type VARCHAR(30) NOT NULL,
    description TEXT,
    metric_code VARCHAR(100),
    quantity NUMERIC,
    unit_amount_cents BIGINT,
    amount_cents BIGINT NOT NULL,
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    pricing_details JSONB NOT NULL DEFAULT '{}',
    -- pricing_details: {"charge_model": "tiered_graduated", "tiers_applied": [
    --     {"tier": 1, "up_to": 1000, "quantity": 1000, "unit_amount_cents": 0, "amount_cents": 0},
    --     {"tier": 2, "up_to": 10000, "quantity": 5000, "unit_amount_cents": 5, "amount_cents": 25000}
    -- ]}
    sort_order INTEGER NOT NULL DEFAULT 0,
    CHECK (charge_type IN ('subscription','usage','one_time','proration_credit','proration_charge','credit'))
);

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
    -- failure_info: {"code": "card_declined", "message": "Insufficient funds",
    --               "decline_code": "insufficient_funds", "attempt_number": 2}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('pending','processing','succeeded','failed','refunded'))
);

CREATE TABLE credit_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    reason VARCHAR(100) NOT NULL,
    credit_amount_cents BIGINT NOT NULL DEFAULT 0,
    refund_amount_cents BIGINT NOT NULL DEFAULT 0,
    items JSONB NOT NULL DEFAULT '[]',
    -- items: [{"line_item_id": "uuid", "amount_cents": 2500, "quantity": 1}]
    status VARCHAR(20) NOT NULL DEFAULT 'issued',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Credit Wallets

```sql
CREATE TABLE wallets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    name VARCHAR(255) NOT NULL DEFAULT 'Default',
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    rate_amount NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    balance NUMERIC NOT NULL DEFAULT 0,
    paid_credits NUMERIC NOT NULL DEFAULT 0,
    granted_credits NUMERIC NOT NULL DEFAULT 0,
    consumed_credits NUMERIC NOT NULL DEFAULT 0,
    priority INTEGER NOT NULL DEFAULT 1,
    expiration_at TIMESTAMPTZ,
    applies_to JSONB NOT NULL DEFAULT '{}',
    -- applies_to: {"metric_codes": ["api_calls", "ai_tokens"]} or {} for all charges
    top_up_config JSONB,
    -- top_up_config: {"threshold": 100, "amount": 1000, "auto": true}
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE wallet_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id UUID NOT NULL REFERENCES wallets(id),
    transaction_type VARCHAR(20) NOT NULL,
    source VARCHAR(50) NOT NULL,
    credit_amount NUMERIC NOT NULL,
    balance_before NUMERIC NOT NULL,
    balance_after NUMERIC NOT NULL,
    invoice_id UUID REFERENCES invoices(id),
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (transaction_type IN ('inbound','outbound')),
    CHECK (source IN ('purchased','granted','usage_deduction','voided','expired','refund'))
);

REVOKE UPDATE, DELETE ON wallet_transactions FROM PUBLIC;
```

---

## Revenue Recognition & SaaS Metrics

```sql
CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    total_value_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,
    obligations JSONB NOT NULL DEFAULT '[]',
    -- obligations: [
    --   {"id": "uuid", "description": "Pro Plan Access", "type": "over_time",
    --    "standalone_price_cents": 11880, "allocated_price_cents": 10692,
    --    "recognition_method": "straight_line", "start": "2026-01-01", "end": "2026-12-31"},
    --   {"id": "uuid", "description": "Onboarding", "type": "point_in_time",
    --    "standalone_price_cents": 2000, "allocated_price_cents": 1800,
    --    "recognition_method": "milestone", "start": "2026-01-15", "end": "2026-01-15"}
    -- ]
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE recognition_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id UUID NOT NULL REFERENCES contracts(id),
    obligation_id UUID NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    amount_cents BIGINT NOT NULL,
    recognized BOOLEAN NOT NULL DEFAULT false,
    recognized_at TIMESTAMPTZ,
    UNIQUE(contract_id, obligation_id, period_start)
);

CREATE TABLE revenue_journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    schedule_id UUID NOT NULL REFERENCES recognition_schedules(id),
    period DATE NOT NULL,
    debit_account VARCHAR(100) NOT NULL,
    credit_account VARCHAR(100) NOT NULL,
    amount_cents BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    posted_at TIMESTAMPTZ,
    CHECK (status IN ('pending','posted','reversed'))
);

REVOKE UPDATE, DELETE ON revenue_journal_entries FROM PUBLIC;

CREATE TABLE mrr_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    subscription_id UUID NOT NULL,
    movement_date DATE NOT NULL,
    movement_type VARCHAR(20) NOT NULL,
    mrr_delta_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (movement_type IN ('new','expansion','contraction','churn','reactivation'))
);

CREATE INDEX idx_mrr_date ON mrr_movements(tenant_id, movement_date);

CREATE TABLE mrr_snapshots (
    snapshot_date DATE NOT NULL,
    tenant_id UUID NOT NULL,
    total_mrr_cents BIGINT NOT NULL,
    new_mrr_cents BIGINT NOT NULL DEFAULT 0,
    expansion_mrr_cents BIGINT NOT NULL DEFAULT 0,
    contraction_mrr_cents BIGINT NOT NULL DEFAULT 0,
    churned_mrr_cents BIGINT NOT NULL DEFAULT 0,
    reactivation_mrr_cents BIGINT NOT NULL DEFAULT 0,
    active_subscriptions INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY(snapshot_date, tenant_id)
);
```

---

## Coupons, Dunning & Webhooks

```sql
CREATE TABLE coupons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    discount_config JSONB NOT NULL,
    -- discount_config: {"type": "percentage", "percent_off": 20, "duration": "repeating",
    --                   "duration_in_months": 3, "applies_to": {"metric_codes": ["api_calls"]}}
    -- OR: {"type": "fixed_amount", "amount_off_cents": 5000, "currency": "USD", "duration": "once"}
    max_redemptions INTEGER,
    times_redeemed INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE customer_coupons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    coupon_id UUID NOT NULL REFERENCES coupons(id),
    subscription_id UUID REFERENCES subscriptions(id),
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    months_remaining INTEGER
);

CREATE TABLE dunning_attempts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    attempt_number INTEGER NOT NULL,
    scheduled_at TIMESTAMPTZ NOT NULL,
    attempted_at TIMESTAMPTZ,
    result VARCHAR(20),
    next_retry_at TIMESTAMPTZ
);

CREATE TABLE webhook_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    secret_hash VARCHAR(255) NOT NULL,
    events TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES webhook_endpoints(id),
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    attempts INTEGER NOT NULL DEFAULT 0,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    actor_id UUID,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);

REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;
```

---

## Example Queries

```sql
-- Find all plans with per-unit pricing under $0.01 for api_calls
SELECT id, name, code, charges
FROM plans
WHERE tenant_id = 'tenant-uuid'
  AND is_active = true
  AND charges @> '[{"metric_code": "api_calls", "charge_model": "per_unit"}]'::jsonb;

-- Get usage by model for AI billing
SELECT
    properties->>'model' AS model,
    SUM(quantity) AS total_tokens
FROM usage_events
WHERE customer_id = 'customer-uuid'
  AND metric_code = 'ai_tokens'
  AND event_timestamp BETWEEN '2026-05-01' AND '2026-06-01'
GROUP BY properties->>'model';

-- MRR waterfall for a tenant
SELECT * FROM mrr_snapshots
WHERE tenant_id = 'tenant-uuid'
ORDER BY snapshot_date DESC
LIMIT 12;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 2 | tenants, users |
| Customers | 1 | customers (addresses + payment methods as JSONB) |
| Catalog & Pricing | 2 | billable_metrics, plans (charges + entitlements as JSONB) |
| Subscriptions | 1 | subscriptions (plan snapshot as JSONB) |
| Usage Metering | 2 | usage_events (partitioned), usage_aggregates |
| Invoicing | 4 | invoices, invoice_line_items, payments, credit_notes |
| Wallets | 2 | wallets, wallet_transactions |
| Revenue | 4 | contracts (obligations as JSONB), recognition_schedules, revenue_journal_entries, mrr_movements + mrr_snapshots |
| Coupons & Dunning | 4 | coupons, customer_coupons, dunning_attempts, webhook_endpoints + webhook_deliveries |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **25** | Compact schema; JSONB absorbs pricing/entitlement variability |

---

## Key Design Decisions

1. **Plan charges as a single JSONB array** rather than separate price_components + price_tiers tables. The full pricing configuration (charge model, tiers, free units, percentages) lives in `plans.charges`. This is the Lago pattern: plans are loaded and saved as a unit, matching how pricing teams think about plan configuration. A `plan_snapshot` on subscriptions captures the exact pricing at subscription time.

2. **Customer addresses and payment methods as JSONB** rather than separate tables. Most billing operations need the full customer record (name, email, billing address, default payment method) in a single query. JSONB keeps this in one row. Payment method tokens are still stored — never PANs — but the structure is flexible enough to support cards, bank accounts, and digital wallets without schema changes.

3. **Usage event properties as JSONB** with GIN index. Events carry arbitrary dimensions (model name, region, endpoint, status code) that enable flexible billing: "bill on total tokens, grouped by model." The `jsonb_path_ops` GIN index supports containment queries for filtering by property values.

4. **Invoice line items are relational** despite other entities using JSONB. The money path (invoices → line items → payments → journal entries) keeps full relational integrity. Line items have a `pricing_details` JSONB column that records the exact calculation (which tiers were applied, how many units per tier) for audit transparency without separate calculation tables.

5. **Revenue recognition contracts store obligations as JSONB** but recognition schedules and journal entries are relational. The obligation definitions (SSP, allocation, recognition method) change infrequently and load as a unit. The schedules and journal entries are queried independently for period-close processing and audit reporting.

6. **Row-Level Security on customer-facing tables.** Customers, subscriptions, invoices, and wallets have RLS policies enforcing tenant isolation. The application sets `app.tenant_id` per transaction. Internal tables (mrr_movements, audit_log) use explicit tenant_id filtering.

7. **Wallet transactions are immutable** (REVOKE UPDATE/DELETE). Every credit movement records balance_before and balance_after for self-auditing. The wallet's cached balance can be recomputed from transactions. The `applies_to` JSONB on wallets enables scoping credits to specific metrics (e.g., "these credits only apply to AI token charges").

8. **Coupon discount configuration as JSONB** rather than separate amount_off/percent_off columns. This enables rich discount rules (applies to specific metrics, percentage with maximum cap, combined fixed + percentage) without schema changes as discount models evolve.
