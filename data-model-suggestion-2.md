# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: SaaS Billing & Metering · Created: 2026-05-22

## Philosophy

This model treats the immutable event log as the single source of truth for every billing action. Every state change — customer created, subscription started, usage event ingested, invoice finalized, payment received, credit applied, revenue recognized — is recorded as an append-only event. Current state is derived by replaying events or reading from pre-computed materialised views (CQRS pattern).

Billing systems have a natural affinity for event sourcing because billing is fundamentally a ledger: money flows in, charges accumulate, credits are consumed, and revenue is recognized over time. Every flow must be auditable, immutable, and reconstructible. Traditional mutable-state billing systems achieve auditability by bolting on audit log tables; an event-sourced system provides it structurally — the audit trail IS the data model.

This approach satisfies ASC 606/IFRS 15 requirements natively: every revenue-affecting event is timestamped and immutable, making it possible to reconstruct the revenue state at any historical point. It also enables powerful analytics: AI models can consume the event stream to detect anomalies, predict churn, and recommend pricing changes based on the complete history of every customer interaction.

**Best for:** Platforms where auditability, temporal queries, and AI-driven analytics over billing history are primary requirements. Ideal for companies targeting SOC 2, PCI DSS, and ASC 606 compliance from day one.

**Trade-offs:**
- **Pro:** Complete, immutable audit trail — every billing action is recorded and cannot be modified
- **Pro:** Temporal queries are native: "what was this customer's MRR on March 1?" requires no bi-temporal hacks
- **Pro:** AI/ML analytics benefit from structured event streams for churn prediction, anomaly detection, and pricing optimization
- **Pro:** Revenue recognition compliance is structural, not bolted on
- **Con:** Higher write amplification — every state change writes to the event store AND updates read models
- **Con:** Eventual consistency between event store and read models (typically milliseconds)
- **Con:** Read models require explicit maintenance; schema changes may require event replay
- **Con:** Higher storage growth; requires retention policies for high-volume usage events

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Every revenue-affecting event is immutable; revenue state reconstructible at any point in time |
| CloudEvents v1.0 | Usage events and billing events follow CloudEvents envelope (specversion, id, source, type, time, data) |
| ISO 4217 | Currency codes on every monetary event |
| ISO 3166 | Country codes in customer and address events |
| PCI DSS | Payment events store only tokens; no PANs in the event store |
| SOC 2 Type II | Immutable event store satisfies "evidence of actions" requirements |
| GAAP SaaS Metrics | MRR movements derived from subscription lifecycle events |
| Double-Entry Bookkeeping | Ledger events always balance (debits = credits) |

---

## Event Store (Source of Truth)

```sql
CREATE TABLE event_store (
    sequence_id BIGSERIAL NOT NULL,
    event_id UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_type VARCHAR(100) NOT NULL,
    stream_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    event_version INTEGER NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    tenant_id UUID NOT NULL,
    actor_id UUID,
    actor_type VARCHAR(50) NOT NULL DEFAULT 'system',
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (sequence_id)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

ALTER TABLE event_store ADD CONSTRAINT uq_stream_version
    UNIQUE (stream_id, event_version);

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_tenant ON event_store(tenant_id, occurred_at DESC);

REVOKE UPDATE, DELETE ON event_store FROM PUBLIC;

-- Event type registry
INSERT INTO event_type_registry (event_type, stream_type, description) VALUES
-- Customer lifecycle
('customer.created', 'customer', 'New customer registered'),
('customer.updated', 'customer', 'Customer details changed'),
('customer.address_added', 'customer', 'Billing/shipping address added'),
('customer.payment_method_added', 'customer', 'Payment method tokenized and stored'),
-- Plan & pricing
('plan.created', 'plan', 'New pricing plan defined'),
('plan.updated', 'plan', 'Plan configuration changed (creates new version)'),
('plan.archived', 'plan', 'Plan deactivated'),
('price_component.created', 'plan', 'Price component added to plan'),
('price_component.updated', 'plan', 'Price component configuration changed'),
-- Subscription lifecycle
('subscription.created', 'subscription', 'New subscription started'),
('subscription.trial_started', 'subscription', 'Trial period began'),
('subscription.activated', 'subscription', 'Subscription moved to active'),
('subscription.upgraded', 'subscription', 'Plan changed to higher tier'),
('subscription.downgraded', 'subscription', 'Plan changed to lower tier'),
('subscription.paused', 'subscription', 'Subscription paused'),
('subscription.resumed', 'subscription', 'Subscription resumed from pause'),
('subscription.canceled', 'subscription', 'Subscription canceled'),
('subscription.expired', 'subscription', 'Subscription term ended'),
('subscription.renewed', 'subscription', 'Subscription renewed for new term'),
-- Usage metering
('usage.event_ingested', 'usage', 'Raw usage event received'),
('usage.aggregated', 'usage', 'Usage aggregated for billing period'),
('usage.anomaly_detected', 'usage', 'Unusual usage pattern flagged'),
-- Invoicing
('invoice.drafted', 'invoice', 'Draft invoice created'),
('invoice.line_item_added', 'invoice', 'Line item added to draft invoice'),
('invoice.finalized', 'invoice', 'Invoice finalized and sent'),
('invoice.payment_attempted', 'invoice', 'Payment collection attempted'),
('invoice.paid', 'invoice', 'Invoice fully paid'),
('invoice.voided', 'invoice', 'Invoice voided'),
('invoice.marked_uncollectible', 'invoice', 'Invoice marked as uncollectible'),
-- Payments
('payment.succeeded', 'payment', 'Payment processed successfully'),
('payment.failed', 'payment', 'Payment attempt failed'),
('payment.refunded', 'payment', 'Payment refunded'),
-- Credits & wallets
('wallet.created', 'wallet', 'Credit wallet created'),
('wallet.credits_added', 'wallet', 'Credits purchased or granted'),
('wallet.credits_consumed', 'wallet', 'Credits consumed for usage'),
('wallet.credits_expired', 'wallet', 'Credits expired'),
('wallet.credits_voided', 'wallet', 'Credits voided'),
-- Revenue recognition
('revenue.obligation_created', 'contract', 'Performance obligation identified'),
('revenue.scheduled', 'contract', 'Revenue recognition scheduled'),
('revenue.recognized', 'contract', 'Revenue recognized for period'),
('revenue.deferred', 'contract', 'Revenue deferred'),
('revenue.journal_posted', 'contract', 'Journal entry posted'),
-- Entitlements
('entitlement.granted', 'entitlement', 'Feature access granted'),
('entitlement.revoked', 'entitlement', 'Feature access revoked'),
('entitlement.limit_reached', 'entitlement', 'Usage limit reached'),
-- Dunning
('dunning.scheduled', 'dunning', 'Retry attempt scheduled'),
('dunning.attempted', 'dunning', 'Retry attempt executed'),
('dunning.exhausted', 'dunning', 'All retry attempts exhausted');

CREATE TABLE event_type_registry (
    event_type VARCHAR(255) PRIMARY KEY,
    stream_type VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    schema_version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(100) NOT NULL,
    snapshot_data JSONB NOT NULL,
    event_version INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id, event_version)
);
```

---

## Usage Event Ingestion (High-Throughput Stream)

```sql
-- Separate high-throughput table for raw usage events
-- These flow through to the event_store but are also queryable directly
CREATE TABLE usage_event_stream (
    sequence_id BIGSERIAL NOT NULL,
    event_id UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    customer_external_id VARCHAR(255) NOT NULL,
    metric_code VARCHAR(100) NOT NULL,
    quantity NUMERIC NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    idempotency_key VARCHAR(255) NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (sequence_id)
) PARTITION BY RANGE (event_timestamp);

CREATE TABLE usage_stream_2026_q2 PARTITION OF usage_event_stream
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE usage_stream_2026_q3 PARTITION OF usage_event_stream
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

CREATE UNIQUE INDEX idx_usage_dedup ON usage_event_stream(tenant_id, idempotency_key);
CREATE INDEX idx_usage_customer ON usage_event_stream(tenant_id, customer_external_id, metric_code, event_timestamp);

REVOKE UPDATE, DELETE ON usage_event_stream FROM PUBLIC;
```

---

## Ledger (Double-Entry for Financial Events)

```sql
-- Every monetary event creates balanced ledger entries
CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    transaction_id UUID NOT NULL,
    account_type VARCHAR(50) NOT NULL,
    account_name VARCHAR(100) NOT NULL,
    direction VARCHAR(6) NOT NULL,
    amount_cents BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    reference_type VARCHAR(50),
    reference_id UUID,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (direction IN ('debit','credit')),
    CHECK (account_type IN ('asset','liability','equity','revenue','expense'))
);

CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account_name, occurred_at);
CREATE INDEX idx_ledger_tenant ON ledger_entries(tenant_id, occurred_at DESC);

REVOKE UPDATE, DELETE ON ledger_entries FROM PUBLIC;
```

---

## Read Models (CQRS Projections)

```sql
-- ============================================================
-- PROJECTION: CUSTOMERS
-- ============================================================

CREATE TABLE rm_customers (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    external_id VARCHAR(255),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    balance_cents BIGINT NOT NULL DEFAULT 0,
    tax_id VARCHAR(100),
    tax_exempt BOOLEAN NOT NULL DEFAULT false,
    billing_address JSONB,
    payment_methods JSONB NOT NULL DEFAULT '[]',
    updated_at TIMESTAMPTZ NOT NULL,
    UNIQUE(tenant_id, external_id)
);

-- ============================================================
-- PROJECTION: PLANS & PRICING
-- ============================================================

CREATE TABLE rm_plans (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NOT NULL,
    billing_interval VARCHAR(20) NOT NULL,
    base_amount_cents BIGINT NOT NULL DEFAULT 0,
    currency CHAR(3) NOT NULL,
    version INTEGER NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    price_components JSONB NOT NULL DEFAULT '[]',
    entitlements JSONB NOT NULL DEFAULT '[]',
    updated_at TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: SUBSCRIPTIONS
-- ============================================================

CREATE TABLE rm_subscriptions (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    plan_id UUID NOT NULL,
    status VARCHAR(30) NOT NULL,
    current_period_start TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    trial_end TIMESTAMPTZ,
    canceled_at TIMESTAMPTZ,
    mrr_cents BIGINT NOT NULL DEFAULT 0,
    items JSONB NOT NULL DEFAULT '[]',
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_subs_customer ON rm_subscriptions(customer_id, status);
CREATE INDEX idx_rm_subs_tenant ON rm_subscriptions(tenant_id, status);

-- ============================================================
-- PROJECTION: INVOICES
-- ============================================================

CREATE TABLE rm_invoices (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    subscription_id UUID,
    invoice_number VARCHAR(50) UNIQUE,
    status VARCHAR(30) NOT NULL,
    currency CHAR(3) NOT NULL,
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    tax_total_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_due_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT NOT NULL DEFAULT 0,
    line_items JSONB NOT NULL DEFAULT '[]',
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    due_date DATE,
    finalized_at TIMESTAMPTZ,
    paid_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_invoices_customer ON rm_invoices(customer_id, status);

-- ============================================================
-- PROJECTION: WALLETS
-- ============================================================

CREATE TABLE rm_wallets (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    currency CHAR(3) NOT NULL,
    balance NUMERIC NOT NULL DEFAULT 0,
    paid_credits NUMERIC NOT NULL DEFAULT 0,
    granted_credits NUMERIC NOT NULL DEFAULT 0,
    consumed_credits NUMERIC NOT NULL DEFAULT 0,
    expiration_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: USAGE AGGREGATES
-- ============================================================

CREATE TABLE rm_usage_aggregates (
    customer_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    metric_code VARCHAR(100) NOT NULL,
    subscription_id UUID NOT NULL,
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    aggregated_value NUMERIC NOT NULL DEFAULT 0,
    event_count BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY(customer_id, metric_code, subscription_id, period_start)
);

-- ============================================================
-- PROJECTION: ENTITLEMENTS
-- ============================================================

CREATE TABLE rm_entitlements (
    customer_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    feature_key VARCHAR(100) NOT NULL,
    has_access BOOLEAN NOT NULL DEFAULT true,
    usage_limit BIGINT,
    current_usage BIGINT NOT NULL DEFAULT 0,
    reset_period VARCHAR(20),
    source VARCHAR(50) NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY(customer_id, feature_key)
);

-- ============================================================
-- PROJECTION: REVENUE RECOGNITION
-- ============================================================

CREATE TABLE rm_revenue_schedules (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    contract_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    obligation_description TEXT NOT NULL,
    allocated_price_cents BIGINT NOT NULL,
    recognition_method VARCHAR(20) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    amount_cents BIGINT NOT NULL,
    recognized BOOLEAN NOT NULL DEFAULT false,
    recognized_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_deferred_revenue (
    tenant_id UUID NOT NULL,
    period DATE NOT NULL,
    opening_balance_cents BIGINT NOT NULL,
    cash_received_cents BIGINT NOT NULL DEFAULT 0,
    revenue_recognized_cents BIGINT NOT NULL DEFAULT 0,
    closing_balance_cents BIGINT NOT NULL,
    PRIMARY KEY(tenant_id, period)
);

-- ============================================================
-- PROJECTION: SAAS METRICS
-- ============================================================

CREATE TABLE rm_mrr_snapshots (
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

-- ============================================================
-- PROJECTION TRACKING
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_sequence_id BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_rebuilding BOOLEAN NOT NULL DEFAULT false,
    error_message TEXT
);
```

---

## Temporal Queries

```sql
-- What was this customer's subscription state on March 1?
SELECT event_type, event_data, occurred_at
FROM event_store
WHERE stream_type = 'subscription'
  AND stream_id = 'subscription-uuid'
  AND occurred_at <= '2026-03-01'::timestamptz
ORDER BY event_version;

-- All MRR-changing events in the last 30 days
SELECT event_type, event_data, occurred_at
FROM event_store
WHERE event_type IN ('subscription.created','subscription.upgraded',
                     'subscription.downgraded','subscription.canceled')
  AND tenant_id = 'tenant-uuid'
  AND occurred_at >= NOW() - INTERVAL '30 days'
ORDER BY occurred_at;

-- Revenue recognition audit trail for a contract
SELECT event_type, event_data, occurred_at, actor_id
FROM event_store
WHERE stream_type = 'contract'
  AND stream_id = 'contract-uuid'
ORDER BY event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store (partitioned), event_type_registry |
| Snapshots | 1 | stream_snapshots |
| Usage Stream | 1 | usage_event_stream (partitioned, high-throughput) |
| Ledger | 1 | ledger_entries (double-entry) |
| Read Model: Customers | 1 | rm_customers |
| Read Model: Catalog | 1 | rm_plans |
| Read Model: Subscriptions | 1 | rm_subscriptions |
| Read Model: Invoices | 1 | rm_invoices |
| Read Model: Wallets | 1 | rm_wallets |
| Read Model: Usage | 1 | rm_usage_aggregates |
| Read Model: Entitlements | 1 | rm_entitlements |
| Read Model: Revenue | 2 | rm_revenue_schedules, rm_deferred_revenue |
| Read Model: Metrics | 1 | rm_mrr_snapshots |
| Projection Tracking | 1 | projection_checkpoints |
| **Total** | **17** | Event store is source of truth; read models are disposable |

---

## Key Design Decisions

1. **Separate usage_event_stream from the general event_store.** Usage events arrive at 10,000+/sec; mixing them with business events would bloat the main event store and degrade stream replay performance. The usage stream is append-only, partitioned, and optimized for high throughput. Business events (subscription changes, invoice actions) go to the main event store.

2. **Double-entry ledger_entries table** for every monetary flow. When a payment is received: debit `asset:cash`, credit `liability:deferred_revenue`. When revenue is recognized: debit `liability:deferred_revenue`, credit `revenue:recognized`. The invariant `SUM(debits) = SUM(credits)` per transaction_id provides self-auditing financial integrity without external reconciliation.

3. **Read models store denormalized JSONB** for nested structures. `rm_plans.price_components` is a JSONB array of the full pricing configuration; `rm_invoices.line_items` contains all line items inline. This eliminates joins for the most common read operations (display a plan, show an invoice) while the event store preserves the normalized history.

4. **Revenue recognition as a first-class event stream.** Revenue events (`revenue.obligation_created`, `revenue.scheduled`, `revenue.recognized`, `revenue.journal_posted`) are in the event store alongside billing events. This makes it possible to replay the revenue state at any historical point, satisfying ASC 606 audit requirements without a separate audit system.

5. **REVOKE UPDATE/DELETE on all source-of-truth tables** (event_store, usage_event_stream, ledger_entries). Immutability is enforced at the database level, not the application level. This satisfies SOC 2 and PCI DSS requirements for tamper-proof records.

6. **MRR snapshots derived from subscription events.** Rather than storing MRR movements as a separate fact table, the event-sourced model derives MRR from subscription lifecycle events (created, upgraded, downgraded, canceled). The `rm_mrr_snapshots` read model materializes daily aggregations for dashboard performance.

7. **Projection checkpoints enable incremental processing.** Each projection tracks the last processed `sequence_id`, enabling catch-up after downtime without full replay. The `is_rebuilding` flag supports concurrent rebuild operations for new projections or schema changes.

8. **Event versioning per stream** (`stream_id + event_version` unique constraint) prevents duplicate events and enables optimistic concurrency. Writers read the current version, increment, and INSERT — a concurrent write to the same stream fails the uniqueness check and retries.
