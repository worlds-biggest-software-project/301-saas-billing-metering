# Standards & API Reference

> Project: SaaS Billing & Metering · Generated: 2026-05-03

## Industry Standards & Specifications

### Revenue Recognition Standards

| Standard | Authority | Relevance | URL |
|----------|-----------|-----------|-----|
| ASC 606 | FASB (US GAAP) | Revenue Recognition from Contracts with Customers; prescribes five-step model for recognizing revenue when performance obligations are satisfied. Billing engines must produce immutable audit trails supporting ASC 606 compliance. | https://www.fasb.org/standards/ASC-606 |
| IFRS 15 | IASB (International) | International equivalent to ASC 606; effective Jan 1, 2018. Requires revenue recognition to reflect transfer of control, not billing activity. Billing platforms must support both standards. | https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/ |

### Compliance and Security Standards

| Standard | Authority | Relevance | URL |
|----------|-----------|-----------|-----|
| PCI DSS | PCI Security Standards Council | Payment Card Industry Data Security Standard; required for any billing system storing, processing, or transmitting cardholder data. Defines encryption, access control, and audit logging requirements. | https://www.pcisecuritystandards.org/standards/pci-dss/ |
| SOC 2 Type II | AICPA | Security, availability, and confidentiality controls; critical for B2B SaaS platforms handling customer billing data. Requires annual audits and ongoing compliance. | https://www.aicpa.org/interestareas/informationsystems/resources/soc-2-reporting-service-organizations |
| GDPR (Article 6, 21) | EU Regulation | Data processing lawfulness and privacy; requires consent for billing data collection and processing. Relevant for EU-based customers or operations. | https://gdpr-info.eu/ |

### API and Data Standards

| Standard | Description | Relevance | URL |
|----------|-------------|-----------|-----|
| REST API Principles | HTTP/1.1 (RFC 7231) + JSON | Industry-standard for billing APIs; all surveyed platforms expose REST APIs accepting JSON and returning structured responses. | https://datatracker.ietf.org/doc/html/rfc7231 |
| OpenAPI 3.1 | Specification | Emerging standard for API documentation; enables automated client generation and integration testing. | https://spec.openapis.org/oas/v3.1.0.html |
| JSON Schema 2020-12 | Meta-schema specification | Data validation for billing configurations, pricing models, and usage event definitions. Integrated with OpenAPI 3.1+. | https://json-schema.org/draft/2020-12/json-schema-validation |
| OAuth 2.0 | IETF RFC 6749 | Authentication and authorization for multi-tenant SaaS billing APIs. Industry standard for third-party access. | https://datatracker.ietf.org/doc/rfc6749/ |
| OAuth 2.1 (draft) | IETF | Modern OAuth addressing security improvements (PKCE mandatory, legacy flows removed). Recommended for new implementations. | https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ |
| CloudEvents | CNCF Specification | Event format specification for usage event ingestion; emerging standard for billing event schemas. OpenMeter uses CloudEvents; others define proprietary event formats. | https://cloudevents.io/ |
| AsyncAPI | Event-driven API specification | Specification for asynchronous messaging and event streaming in billing systems (webhook delivery, event subscriptions). | https://www.asyncapi.com/ |

### SaaS Metrics Standards

| Standard | Description | Relevance | URL |
|----------|-------------|-----------|-----|
| GAAP SaaS Metrics | Accounting Standards | Authoritative definitions of ARR, MRR, NRR, churn, CAC, LTV. Billing engines must accurately calculate these metrics for financial reporting. | https://www.investopedia.com/terms/r/recurring-revenue.asp |
| Revenue Waterfall | Financial Reporting | Standard format for visualizing revenue movement (beginning balance → new customers → churn → expansion → ending balance). Billing platforms should support waterfall reporting. | Industry practice (no central standard) |

---

## Similar Products — Developer Documentation & APIs

### Stripe Billing

- **Description:** Integrated billing platform with subscription, usage-based, and hybrid pricing models. Part of Stripe payments ecosystem with ~$1B Metronome acquisition (Jan 2026) for advanced contract handling.
- **API Documentation:** https://docs.stripe.com/billing
- **Billing Meter API:** https://docs.stripe.com/api/billing/meter
- **Usage Recording:** https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage
- **SDKs/Libraries:** JavaScript, Python, Go, Java, Ruby, .NET
- **Developer Guide:** https://docs.stripe.com/get-started/use-cases/usage-based-billing
- **Standards:** REST API with JSON payloads; OpenAPI documentation available. Integrates with Stripe payments and accounting systems.
- **Authentication:** API Keys (restricted and unrestricted); webhooks use HMAC signatures. OAuth 2.0 for third-party integrations.
- **Rate Limits:** Meter Events API: 1,000 calls/sec standard (v2 supports 10,000/sec for high-throughput).

### Orb

- **Description:** Real-time usage metering and billing engine purpose-built for consumption-based pricing. Specializes in pricing experimentation and flexible metrics.
- **API Documentation:** https://docs.withorb.com/overview
- **REST API Reference:** https://docs.withorb.com/reference/api-overview
- **Metering API:** https://docs.withorb.com/reference/metering
- **Pricing Simulations:** https://docs.withorb.com/guides/pricing-simulations
- **SDKs/Libraries:** Python, JavaScript, Java, Go, Ruby
- **Developer Guide:** https://docs.withorb.com/guides/getting-started
- **Standards:** REST API with JSON; custom event format (not CloudEvents). OpenAPI spec available.
- **Authentication:** API Keys with role-based access control (RBAC). OAuth 2.0 for customer portal SSO.
- **Rate Limits & Throughput:** 250,000+ events/sec for standard metering; custom higher-throughput configurations available.

### Chargebee

- **Description:** Comprehensive subscription billing platform with usage add-ons, dunning, and revenue recognition. 2026 update added AI monetization features (tokens, API calls).
- **API Documentation:** https://apidocs.chargebee.com/docs/api/getting-started
- **Subscriptions API:** https://apidocs.chargebee.com/docs/api/subscriptions
- **Usages API (Metered Billing):** https://apidocs.chargebee.com/docs/api/usages
- **SDKs/Libraries:** Node, Python, PHP, Java, Go, Ruby, .NET; framework adapters (Laravel, Next.js)
- **Developer Guide:** https://www.chargebee.com/tutorials/quickstart/
- **Standards:** REST API with form-encoded requests and JSON responses. HTTP Basic Auth for API keys. Webhooks for event delivery.
- **Authentication:** HTTP Basic Auth (API key as username). OAuth 2.0 for hosted checkout and integrations.
- **Rate Limits & Throughput:** Not publicly documented; request-rate limits implied through pricing tiers.

### Zuora

- **Description:** Enterprise subscription billing and revenue management platform. Emphasis on complex contracts, multi-entity billing, and ASC 606/IFRS 15 compliance.
- **API Documentation:** https://docs.zuora.com/en/zuora-platform/integration/apis/rest-api
- **REST API Reference:** https://developer.zuora.com/v1-api-reference/
- **Billing API:** https://docs.zuora.com/en/zuora-billing/set-up-zuora-billing
- **Revenue API:** https://developer.zuora.com/other-api/revenue
- **SDKs/Libraries:** Python, Java, JavaScript, Ruby, Go
- **Developer Guide:** https://developer.zuora.com/
- **Standards:** REST API v1 with JSON payloads. OpenAPI spec available. Webhooks for event notifications.
- **Authentication:** OAuth 2.0 (recommended). Basic Auth for legacy integrations.
- **Rate Limits & Throughput:** Not publicly documented; enterprise-tier based.

### Lago

- **Description:** Open-source Apache 2.0 billing platform for usage-based, subscription, and hybrid pricing. Self-hosted or managed cloud deployment.
- **API Documentation:** https://docs.getlago.com/
- **GitHub Repository:** https://github.com/getlago/lago
- **REST API Reference:** https://docs.getlago.com/api-reference
- **SDKs/Libraries:** Python, JavaScript, Node.js; community contributions for other languages
- **Developer Guide:** https://docs.getlago.com/guide/introduction/welcome-to-lago
- **Standards:** REST API with JSON payloads. Custom event format (not CloudEvents). OpenAPI spec included.
- **Authentication:** API Keys (Bearer token). JWT for webhook signatures.
- **Rate Limits & Throughput:** Self-hosted: unlimited (depends on infrastructure). Cloud: tiered by plan (millions of events/month).

### Flexprice

- **Description:** No-code usage-based pricing platform with credit wallet management and entitlement features. Open-source (Apache 2.0) self-hosted option and managed cloud.
- **API Documentation:** https://flexprice.io/docs
- **GitHub Repository:** https://github.com/flexprice/flexprice
- **REST API Reference:** https://flexprice.io/api
- **SDKs/Libraries:** Python, JavaScript, Node.js
- **Developer Guide:** Getting Started docs in repository and main site
- **Standards:** REST API with JSON; custom event format. OpenAPI spec available.
- **Authentication:** API Keys. OAuth 2.0 for managed cloud.
- **Rate Limits & Throughput:** Up to 60,000 events/sec for real-time metering; scales with infrastructure for self-hosted.

### Stigg

- **Description:** Entitlement management and pricing layer sitting above billing systems (not standalone). Manages feature gating, usage limits, and subscription lifecycle.
- **API Documentation:** https://docs.stigg.io/
- **REST API Reference:** https://docs.stigg.io/reference/api-overview
- **Entitlements API:** https://docs.stigg.io/docs/entitlements
- **Metering API:** https://docs.stigg.io/docs/metering
- **SDKs/Libraries:** JavaScript, Python, Go; React components for UI
- **Developer Guide:** https://docs.stigg.io/getting-started
- **Standards:** REST API with JSON. Custom event format (not CloudEvents). OpenAPI spec available.
- **Authentication:** API Keys with environment separation (dev/prod). OAuth 2.0 for customer portal.
- **Rate Limits & Throughput:** Not publicly documented; request-rate limits by plan.

### Maxio

- **Description:** Billing and financial operations platform for B2B SaaS. Emphasis on revenue recognition, SaaS metrics, and integrations with GL/ERP systems.
- **API Documentation:** https://www.maxio.com/developers
- **REST API Reference:** Proprietary; requires developer account
- **Subscriptions API:** Manage subscriptions, billing, and customer data
- **Usage API:** Record and query usage for metered billing
- **SDKs/Libraries:** JavaScript, Python, Ruby, .NET
- **Developer Guide:** Getting started via developer portal
- **Standards:** REST API with JSON. Proprietary event format.
- **Authentication:** API Keys (Bearer token). OAuth 2.0 for third-party integrations.
- **Rate Limits & Throughput:** Enterprise-tiered; not publicly documented.

### OpenMeter

- **Description:** Open-source Apache 2.0 real-time metering engine. CloudEvents-native for event ingestion. Unopinionated architecture; integrates with any billing system.
- **API Documentation:** https://openmeter.io/docs
- **GitHub Repository:** https://github.com/openmeterio/openmeter
- **REST API Reference:** https://openmeter.io/docs/api
- **Event Ingestion:** https://openmeter.io/docs/metering/events
- **SDKs/Libraries:** Python, JavaScript, Go, Java; community contributions for others
- **Developer Guide:** https://openmeter.io/docs/getting-started/event-ingestion
- **Standards:** CloudEvents format (CNCF spec) for event ingestion. REST API with JSON for meter queries.
- **Authentication:** API Keys. JWT for webhook signatures.
- **Rate Limits & Throughput:** Depends on deployment; self-hosted scales to millions of events/day; cloud tiered by plan.

---

## Notes

### Emerging Standards and Future Directions

1. **CloudEvents for Metering**: OpenMeter champions CNCF CloudEvents as the standard event format for usage event ingestion. Other platforms (Stripe, Orb, Chargebee) use proprietary formats; alignment on CloudEvents could reduce integration friction.

2. **AI-Native Billing Schemas**: As AI companies adopt token-based, model-based, and agentic pricing, new event schemas are emerging (e.g., LLM tokens, model API calls, agent task execution). No standardized schema yet; opportunity for CNCF-led specification.

3. **Entitlement Standards**: Entitlement management (feature gating, usage limits) is fragmented across platforms. OpenFeature (CNCF) offers feature flag standards; entitlement standards are emerging but not standardized.

4. **Revenue Recognition Automation**: ASC 606/IFRS 15 compliance is critical for public SaaS; no standard machine-readable format yet exists for revenue schedule specifications. Opportunity for common schema.

5. **Pricing Experimentation APIs**: Orb and others offer pricing experiment APIs; no standard format for experiment definitions, hypotheses, or results. Common format could enable cross-platform experiment portability.

### Compliance and Regulatory Alignment

- **ASC 606 / IFRS 15**: All billing engines should produce immutable audit trails sufficient for external audits. Revenue recognition must be deferred until performance obligation satisfaction.
- **PCI DSS Level 1**: Any platform storing cardholder data must achieve Level 1 PCI DSS certification (annual audit, quarterly scans). Many billing platforms use tokenization to avoid direct PCI scope.
- **GDPR/CCPA**: Billing systems process personal data (customer names, usage, payment info); must comply with data privacy regulations, including right to erasure and data portability.
- **SOC 2 Type II**: B2B SaaS platforms should target SOC 2 Type II attestation for security, availability, and confidentiality controls.

### Integration Patterns and Best Practices

1. **Idempotent API Design**: Billing events must be idempotent (same event sent twice = same result). Critical for handling network retries and duplicate prevention.
2. **Event Deduplication**: Platforms must deduplicate usage events by unique identifier (source + event_id) to prevent duplicate charges.
3. **Webhook Delivery**: Standard pattern for asynchronous billing notifications (invoice created, payment succeeded, subscription updated). HMAC signatures for webhook authenticity.
4. **Rate Limiting**: High-throughput platforms must clearly document rate limits (events/sec, API calls/sec) and provide graceful degradation (backpressure, queuing).
5. **Currency and Localization**: Global SaaS requires multi-currency support with real-time exchange rates and region-specific tax handling.

### Security Best Practices for Billing APIs

- **Always use HTTPS**: No billing data over HTTP; encryption in transit is mandatory.
- **API Key Management**: Implement key rotation, scoping (restricted keys), and audit logging for key usage.
- **OAuth 2.0 / OAuth 2.1**: Preferred for third-party integrations; enables granular scope-based permissions.
- **HMAC Webhook Signatures**: All webhook deliveries should include HMAC signatures for authenticity verification.
- **Audit Logging**: Immutable audit trail of all billing changes (invoice issued, payment recorded, refund processed) for compliance and dispute resolution.
- **Data Encryption at Rest**: Customer billing data must be encrypted at rest (AES-256 or equivalent).

---

## Recommended Alignment with Standards

For project 301 (SaaS Billing & Metering):

1. **Revenue Recognition**: Implement ASC 606 and IFRS 15 compliance with immutable audit trails and deferral logic for performance obligations.
2. **API Design**: Expose REST API with JSON payloads following OpenAPI 3.1 specification. Consider CloudEvents for usage event ingestion.
3. **Authentication**: Implement OAuth 2.0 (or 2.1) for API access and third-party integrations. Use API Keys for direct integration scenarios.
4. **Security**: Achieve PCI DSS compliance (or use tokenization to reduce scope). Target SOC 2 Type II attestation.
5. **Compliance**: Support GDPR data portability and erasure. Implement right-to-audit and detailed invoice records for regulatory alignment.
6. **Metering**: Design for idempotent event ingestion with deduplication. Support real-time meter aggregations (SUM, AVG, MIN, MAX).
7. **Integrations**: Provide webhooks for async billing notifications and integrations with accounting systems (QuickBooks, Xero, NetSuite).
8. **Reporting**: Support standard SaaS metrics (ARR, MRR, NRR, churn) calculated from billing engine data.

---

## Sources

- [ASC 606 — Revenue Recognition](https://www.fasb.org/standards/ASC-606)
- [IFRS 15 — Revenue from Contracts](https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/)
- [PCI DSS — Payment Card Security](https://www.pcisecuritystandards.org/standards/pci-dss/)
- [Stripe Billing API](https://docs.stripe.com/billing)
- [Orb Billing Platform](https://docs.withorb.com)
- [Chargebee API Documentation](https://apidocs.chargebee.com)
- [Zuora REST API](https://developer.zuora.com)
- [Lago Open-Source Billing](https://docs.getlago.com)
- [Flexprice Platform](https://flexprice.io)
- [Stigg Entitlement API](https://docs.stigg.io)
- [OpenMeter Metering](https://openmeter.io)
- [RFC 7231 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc7231)
- [OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/rfc6749/)
- [CloudEvents Specification](https://cloudevents.io/)
- [OpenAPI 3.1 Specification](https://spec.openapis.org/oas/v3.1.0.html)
