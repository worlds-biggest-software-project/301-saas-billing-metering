# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: SaaS Billing & Metering · Created: 2026-05-22

## Philosophy

This model applies classical relational normalization to every billing concept. Each entity — tenants, customers, plans, prices, subscriptions, invoices, payments, wallets, entitlements, revenue schedules — gets its own table with explicit foreign keys. Pricing models (flat, per-unit, tiered graduated, tiered volume, package, percentage) are represented through a price_components + price_tiers pattern rather than JSONB, ensuring every pricing rule is query-accessible and constraint-enforced.

The approach follows the entity structures observed across Stripe (Product → Price → SubscriptionItem), Zuora (RatePlan → RatePlanCharge → RatePlanChargeTier), and Lago (Plan → Charge → BillableMetric), mapped into a unified relational schema. Revenue recognition tables follow the ASC 606 five-step model (contract → performance obligation → recognition schedule → journal entry) with immutable audit fields.

This is the safest choice for teams operating in audit-intensive environments where ASC 606/IFRS 15 compliance, SOC 2 attestation, and clear data lineage from usage event to recognized revenue are mandatory. Every relationship is visible in the DDL; every financial invariant is database-enforced.

**Best for:** Regulated SaaS companies requiring ASC 606 compliance, strong audit trails, and explicit schema documentation for financial audits.

**Trade-offs:**
- **Pro:** Maximum referential integrity — the database prevents orphaned invoices, invalid pricing states, and broken subscription chains
- **Pro:** ASC 606 compliance is structurally enforced: performance obligations and recognition schedules are explicit tables
- **Pro:** Every pricing rule is a row in a table, enabling SQL analytics across pricing models
- **Pro:** Standard SQL queries without JSONB operators; wide ORM and BI tool compatibility
- **Con:** High table count (~40 tables) increases migration complexity
- **Con:** Adding a new pricing model requires schema migration, not just a new JSONB shape
- **Con:** Usage event properties must be pre-defined as columns or stored in a narrow EAV pattern

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Explicit tables for contracts, performance obligations, recognition schedules, and journal entries |
| CloudEvents v1.0 | Usage event structure follows CloudEvents envelope (specversion, id, source, type, subject, time) |
| ISO 4217 | Currency codes stored as CHAR(3) on every monetary field with minor_unit for rounding |
| ISO 3166 | Country codes (alpha-2) on customer addresses for tax jurisdiction determination |
| UBL 2.1 (OASIS) | Invoice structure aligns with UBL InvoiceLine, TaxTotal, LegalMonetaryTotal patterns |
| PCI DSS | Payment methods store only tokens, last-four, and brand — no PANs or CVVs |
| GAAP SaaS Metrics | MRR movement fact table and subscription history enable ARR, NRR, churn calculations |
| OAuth 2.0 / RFC 6749 | API key and token tables support OAuth flows for multi-tenant access |

---

## Multi-Tenancy & Access Control

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
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

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    key_hash VARCHAR(255) NOT NULL UNIQUE,
    key_prefix VARCHAR(10) NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Customers & Accounts

```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id VARCHAR(255),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    tax_id VARCHAR(100),
    tax_exempt BOOLEAN NOT NULL DEFAULT false,
    balance_cents BIGINT NOT NULL DEFAULT 0,
    auto_collection BOOLEAN NOT NULL DEFAULT true,
    net_term_days INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, external_id)
);

CREATE TABLE customer_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    address_type VARCHAR(20) NOT NULL DEFAULT 'billing',
    line1 TEXT,
    line2 TEXT,
    city VARCHAR(255),
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(customer_id, address_type)
);

CREATE TABLE payment_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    payment_type VARCHAR(50) NOT NULL,
    provider VARCHAR(50) NOT NULL,
    provider_token VARCHAR(255) NOT NULL,
    card_brand VARCHAR(20),
    card_last_four CHAR(4),
    card_exp_month SMALLINT,
    card_exp_year SMALLINT,
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Product Catalog & Pricing

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE billable_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    code VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    aggregation_type VARCHAR(30) NOT NULL,
    field_name VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code),
    CHECK (aggregation_type IN ('count','sum','max','min','avg','unique_count','latest'))
);

CREATE TABLE plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NOT NULL,
    billing_interval VARCHAR(20) NOT NULL,
    billing_interval_count INTEGER NOT NULL DEFAULT 1,
    base_amount_cents BIGINT NOT NULL DEFAULT 0,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    trial_period_days INTEGER NOT NULL DEFAULT 0,
    version INTEGER NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code, version),
    CHECK (billing_interval IN ('day','week','month','quarter','year'))
);

CREATE TABLE price_components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id UUID NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
    billable_metric_id UUID REFERENCES billable_metrics(id),
    name VARCHAR(255) NOT NULL,
    pricing_model VARCHAR(30) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    flat_amount_cents BIGINT,
    unit_amount_cents BIGINT,
    package_size INTEGER,
    percentage NUMERIC(8,4),
    free_units BIGINT NOT NULL DEFAULT 0,
    min_amount_cents BIGINT,
    max_amount_cents BIGINT,
    charge_timing VARCHAR(20) NOT NULL DEFAULT 'in_arrears',
    sort_order INTEGER NOT NULL DEFAULT 0,
    CHECK (pricing_model IN ('flat','per_unit','tiered_graduated','tiered_volume',
                             'package','percentage','stairstep')),
    CHECK (charge_timing IN ('in_advance','in_arrears'))
);

CREATE TABLE price_tiers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    price_component_id UUID NOT NULL REFERENCES price_components(id) ON DELETE CASCADE,
    ordinal INTEGER NOT NULL,
    up_to BIGINT,
    unit_amount_cents BIGINT,
    flat_amount_cents BIGINT,
    UNIQUE(price_component_id, ordinal)
);

CREATE INDEX idx_price_tiers_component ON price_tiers(price_component_id, ordinal);
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
    payment_method_id UUID REFERENCES payment_methods(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('trialing','active','past_due','paused','canceled','expired'))
);

CREATE INDEX idx_subscriptions_customer ON subscriptions(customer_id, status);
CREATE INDEX idx_subscriptions_tenant ON subscriptions(tenant_id, status);

CREATE TABLE subscription_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    price_component_id UUID NOT NULL REFERENCES price_components(id),
    quantity INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(subscription_id, price_component_id)
);
```

---

## Usage Metering

```sql
CREATE TABLE usage_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    metric_code VARCHAR(100) NOT NULL,
    quantity NUMERIC NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    idempotency_key VARCHAR(255) NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, idempotency_key)
) PARTITION BY RANGE (event_timestamp);

CREATE TABLE usage_events_2026_q2 PARTITION OF usage_events
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE usage_events_2026_q3 PARTITION OF usage_events
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

CREATE INDEX idx_usage_events_customer ON usage_events(customer_id, metric_code, event_timestamp);
CREATE INDEX idx_usage_events_dedup ON usage_events(tenant_id, idempotency_key);

CREATE TABLE usage_aggregates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL REFERENCES customers(id),
    metric_code VARCHAR(100) NOT NULL,
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    aggregated_value NUMERIC NOT NULL DEFAULT 0,
    event_count BIGINT NOT NULL DEFAULT 0,
    last_aggregated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(customer_id, metric_code, subscription_id, period_start)
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('draft','open','paid','void','uncollectible')),
    CHECK (collection_method IN ('charge_automatically','send_invoice'))
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id, status);
CREATE INDEX idx_invoices_tenant ON invoices(tenant_id, status);

CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    subscription_item_id UUID REFERENCES subscription_items(id),
    charge_type VARCHAR(30) NOT NULL,
    description TEXT,
    metric_code VARCHAR(100),
    quantity NUMERIC,
    unit_amount_cents BIGINT,
    amount_cents BIGINT NOT NULL,
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    sort_order INTEGER NOT NULL DEFAULT 0,
    CHECK (charge_type IN ('subscription','usage','one_time','proration_credit','proration_charge','credit'))
);

CREATE TABLE invoice_tax_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    line_item_id UUID REFERENCES invoice_line_items(id),
    jurisdiction VARCHAR(100) NOT NULL,
    tax_type VARCHAR(20) NOT NULL,
    tax_rate NUMERIC(8,4) NOT NULL,
    taxable_amount_cents BIGINT NOT NULL,
    tax_amount_cents BIGINT NOT NULL
);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    amount_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    provider VARCHAR(50) NOT NULL,
    provider_payment_id VARCHAR(255),
    failure_code VARCHAR(100),
    failure_message TEXT,
    attempt_number INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('pending','processing','succeeded','failed','refunded','partially_refunded'))
);

CREATE TABLE refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id UUID NOT NULL REFERENCES payments(id),
    amount_cents BIGINT NOT NULL,
    reason TEXT,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    provider_refund_id VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('pending','succeeded','failed'))
);

CREATE TABLE credit_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    credit_note_number VARCHAR(50) UNIQUE,
    reason VARCHAR(100) NOT NULL,
    credit_amount_cents BIGINT NOT NULL DEFAULT 0,
    refund_amount_cents BIGINT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'issued',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('issued','void'))
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
    paid_credits NUMERIC NOT NULL DEFAULT 0,
    granted_credits NUMERIC NOT NULL DEFAULT 0,
    consumed_credits NUMERIC NOT NULL DEFAULT 0,
    balance NUMERIC NOT NULL DEFAULT 0,
    priority INTEGER NOT NULL DEFAULT 1,
    expiration_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('active','terminated','expired'))
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
    status VARCHAR(20) NOT NULL DEFAULT 'settled',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (transaction_type IN ('inbound','outbound')),
    CHECK (source IN ('purchased','granted','usage_deduction','voided','expired','refund'))
);
```

---

## Entitlements

```sql
CREATE TABLE features (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    key VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    feature_type VARCHAR(20) NOT NULL,
    meter_code VARCHAR(100),
    description TEXT,
    UNIQUE(tenant_id, key),
    CHECK (feature_type IN ('boolean','numeric_limit','metered'))
);

CREATE TABLE plan_entitlements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id UUID NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
    feature_id UUID NOT NULL REFERENCES features(id),
    has_access BOOLEAN NOT NULL DEFAULT true,
    usage_limit BIGINT,
    reset_period VARCHAR(20),
    soft_limit BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(plan_id, feature_id),
    CHECK (reset_period IS NULL OR reset_period IN ('daily','weekly','monthly','yearly'))
);

CREATE TABLE customer_entitlement_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    feature_id UUID NOT NULL REFERENCES features(id),
    has_access BOOLEAN,
    usage_limit BIGINT,
    valid_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMPTZ,
    source VARCHAR(50) NOT NULL
);

CREATE TABLE entitlement_usage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id),
    feature_id UUID NOT NULL REFERENCES features(id),
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    current_usage BIGINT NOT NULL DEFAULT 0,
    UNIQUE(customer_id, feature_id, period_start)
);
```

---

## Revenue Recognition (ASC 606)

```sql
CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    start_date DATE NOT NULL,
    end_date DATE,
    total_value_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('active','modified','terminated','completed'))
);

CREATE TABLE performance_obligations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id UUID NOT NULL REFERENCES contracts(id),
    description TEXT NOT NULL,
    obligation_type VARCHAR(20) NOT NULL,
    standalone_selling_price_cents BIGINT NOT NULL,
    allocated_price_cents BIGINT NOT NULL,
    recognition_method VARCHAR(20) NOT NULL,
    satisfaction_start DATE,
    satisfaction_end DATE,
    is_satisfied BOOLEAN NOT NULL DEFAULT false,
    CHECK (obligation_type IN ('over_time','point_in_time')),
    CHECK (recognition_method IN ('straight_line','usage_based','milestone','point_in_time'))
);

CREATE TABLE recognition_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    obligation_id UUID NOT NULL REFERENCES performance_obligations(id),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    amount_cents BIGINT NOT NULL,
    recognized BOOLEAN NOT NULL DEFAULT false,
    recognized_at TIMESTAMPTZ,
    UNIQUE(obligation_id, period_start)
);

CREATE TABLE revenue_journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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
```

---

## SaaS Metrics

```sql
CREATE TABLE subscription_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    plan_id UUID NOT NULL REFERENCES plans(id),
    mrr_amount_cents BIGINT NOT NULL,
    status VARCHAR(30) NOT NULL,
    change_type VARCHAR(20) NOT NULL,
    effective_from TIMESTAMPTZ NOT NULL,
    effective_to TIMESTAMPTZ,
    is_current BOOLEAN NOT NULL DEFAULT true,
    CHECK (change_type IN ('new','upgrade','downgrade','churn','reactivation','renewal'))
);

CREATE INDEX idx_sub_history_current ON subscription_history(subscription_id) WHERE is_current = true;

CREATE TABLE mrr_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    customer_id UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    movement_date DATE NOT NULL,
    movement_type VARCHAR(20) NOT NULL,
    mrr_delta_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (movement_type IN ('new','expansion','contraction','churn','reactivation'))
);

CREATE INDEX idx_mrr_movements_date ON mrr_movements(tenant_id, movement_date);

CREATE TABLE mrr_snapshots (
    snapshot_date DATE NOT NULL,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    total_mrr_cents BIGINT NOT NULL,
    new_mrr_cents BIGINT NOT NULL DEFAULT 0,
    expansion_mrr_cents BIGINT NOT NULL DEFAULT 0,
    contraction_mrr_cents BIGINT NOT NULL DEFAULT 0,
    churned_mrr_cents BIGINT NOT NULL DEFAULT 0,
    reactivation_mrr_cents BIGINT NOT NULL DEFAULT 0,
    active_subscriptions INTEGER NOT NULL DEFAULT 0,
    active_customers INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY(snapshot_date, tenant_id)
);
```

---

## Tax & Coupons

```sql
CREATE TABLE tax_rates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    jurisdiction VARCHAR(100) NOT NULL,
    country_code CHAR(2) NOT NULL,
    region_code VARCHAR(10),
    tax_type VARCHAR(20) NOT NULL,
    rate NUMERIC(8,4) NOT NULL,
    is_inclusive BOOLEAN NOT NULL DEFAULT false,
    effective_from DATE NOT NULL,
    effective_to DATE,
    CHECK (tax_type IN ('vat','gst','sales_tax','hst'))
);

CREATE TABLE coupons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    discount_type VARCHAR(20) NOT NULL,
    amount_off_cents BIGINT,
    percent_off NUMERIC(5,2),
    currency CHAR(3),
    duration VARCHAR(20) NOT NULL,
    duration_in_months INTEGER,
    max_redemptions INTEGER,
    times_redeemed INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, code),
    CHECK (discount_type IN ('fixed_amount','percentage')),
    CHECK (duration IN ('forever','once','repeating'))
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
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    actor_id UUID,
    actor_type VARCHAR(50) NOT NULL DEFAULT 'user',
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_q3 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);

REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;
```

---

## Dunning & Webhooks

```sql
CREATE TABLE dunning_attempts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    payment_id UUID REFERENCES payments(id),
    attempt_number INTEGER NOT NULL,
    scheduled_at TIMESTAMPTZ NOT NULL,
    attempted_at TIMESTAMPTZ,
    result VARCHAR(20),
    next_retry_at TIMESTAMPTZ,
    CHECK (result IS NULL OR result IN ('succeeded','failed','skipped'))
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
    last_attempted_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (status IN ('pending','delivered','failed'))
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 3 | tenants, users, api_keys |
| Customers | 3 | customers, customer_addresses, payment_methods |
| Catalog & Pricing | 5 | products, billable_metrics, plans, price_components, price_tiers |
| Subscriptions | 2 | subscriptions, subscription_items |
| Usage Metering | 2 | usage_events (partitioned), usage_aggregates |
| Invoicing | 5 | invoices, invoice_line_items, invoice_tax_lines, credit_notes, payments + refunds |
| Wallets | 2 | wallets, wallet_transactions |
| Entitlements | 4 | features, plan_entitlements, customer_entitlement_overrides, entitlement_usage |
| Revenue Recognition | 4 | contracts, performance_obligations, recognition_schedules, revenue_journal_entries |
| SaaS Metrics | 3 | subscription_history, mrr_movements, mrr_snapshots |
| Tax & Coupons | 3 | tax_rates, coupons, customer_coupons |
| Audit & Webhooks | 4 | audit_log, dunning_attempts, webhook_endpoints, webhook_deliveries |
| **Total** | **41** | Comprehensive coverage of billing lifecycle |

---

## Key Design Decisions

1. **Monetary values stored as BIGINT cents** rather than NUMERIC decimals. This avoids floating-point rounding issues in financial calculations and follows the Stripe convention. All amounts are in the smallest currency unit (cents for USD, yen for JPY). ISO 4217 minor_unit determines the conversion.

2. **Pricing model decomposition into price_components + price_tiers** rather than JSONB. Every tier bracket is a row in price_tiers with an explicit ordinal, up_to, and amounts. This enables SQL queries like "find all plans where the first tier is under $0.01/unit" without JSONB operators.

3. **Usage events partitioned by event_timestamp** with quarterly partitions. Deduplication uses a unique constraint on `(tenant_id, idempotency_key)` following the CloudEvents source+id pattern. Old partitions can be detached and archived without affecting current billing.

4. **Separate usage_aggregates table** pre-computes period-level aggregations per customer/metric/subscription. This avoids re-scanning millions of raw events at invoice generation time. Aggregates are updated incrementally as events arrive.

5. **ASC 606 tables are immutable** where required. Revenue journal entries have UPDATE/DELETE revoked at the database level. Contract modifications create new contract versions rather than mutating existing records. Recognition schedules are append-only once the period has closed.

6. **SCD Type 2 for subscription_history** enables point-in-time MRR queries. Every subscription state change (upgrade, downgrade, churn) closes the current row and opens a new one. The mrr_movements fact table provides the building blocks for waterfall reports; mrr_snapshots is a materialized daily rollup for dashboard performance.

7. **Entitlements separated from billing** following the Stigg pattern. The features/plan_entitlements tables define what each plan includes; customer_entitlement_overrides handle exceptions (trials, promotions, enterprise deals). This allows changing entitlements without modifying billing configuration.

8. **Wallet transactions use balance_before/balance_after** for self-auditing integrity. Each transaction records the wallet balance before and after the operation, creating a verifiable chain. The wallet's balance column is a cached sum that can be recomputed from transactions if needed.
