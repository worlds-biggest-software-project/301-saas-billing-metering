# SaaS Billing & Metering — Feature & Functionality Survey

> Candidate #301 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Stripe Billing | Commercial SaaS | Proprietary | https://stripe.com/billing |
| Orb | Commercial SaaS | Proprietary | https://www.withorb.com |
| Chargebee | Commercial SaaS | Proprietary | https://www.chargebee.com |
| Zuora | Commercial SaaS | Proprietary | https://www.zuora.com |
| Lago | Open source / Managed | Apache 2.0 / Cloud | https://getlago.com |
| Maxio (fka Chargify) | Commercial SaaS | Proprietary | https://www.maxio.com |
| Flexprice | Commercial SaaS + Open Source | Proprietary / Apache 2.0 | https://flexprice.io |
| Stigg | Commercial SaaS | Proprietary | https://www.stigg.io |
| OpenMeter | Open source / Managed | Apache 2.0 / Cloud | https://openmeter.io |
| Metronome (now Stripe) | Commercial SaaS | Proprietary | https://stripe.com/billing/pricing (acquired by Stripe) |

## Feature Analysis by Solution

### Stripe Billing

**Core features**
- Usage-based metering for consumption events (API calls, tokens, compute hours, agent tasks)
- Flexible pricing models: subscriptions, usage-based, credit-based, tiered, hybrid combinations
- Smart dunning with automated retry logic to reduce churn
- Quotes and multi-phase subscription schedules
- Tax handling and revenue recognition support
- Up to 100 million events per month in base tier
- Payment processing integrated with Stripe ecosystem
- Discount and credit management

**Differentiating features**
- Metronome acquisition (Jan 2026, ~$1B) brings advanced contract complexity handling to Stripe
- AI usage billing features (private preview) with automatic markup percentage calculation across multiple AI providers simultaneously
- Massive developer ecosystem and payment familiarity (virtually all SaaS developers know Stripe)
- 0.7% of billing volume pricing (competitive for high-volume use cases)

**UX patterns**
- Developer-first API design with excellent documentation
- Dashboard-driven configuration for non-technical stakeholders
- Clear separation of billing meter configuration from revenue events
- Gradual feature rollout (e.g., AI features in private preview)

**Integration points**
- Native Stripe payment processing
- Revenue recognition integrations
- Dunning and collection workflows
- Webhook delivery for billing events
- SDKs for JavaScript, Python, Go, Java, Ruby, .NET
- Third-party integrations with accounting systems

**Known gaps**
- Usage-based features historically bolted on, not native (though improving post-Metronome acquisition)
- AI usage billing still in private preview (not generally available)
- Legacy usage-records API being deprecated; migration required to billing meters
- Complex contract handling less polished than Metronome (pre-integration)

**Licence / IP notes**
- Proprietary SaaS; no known patent concerns. Metronome acquisition brings advanced contract IP under Stripe control.

---

### Orb

**Core features**
- Real-time usage metering with 250,000+ events per second throughput
- Flexible metrics: bill on any metric queryable from raw events, supporting averages, maximums, minimums, and custom aggregations
- Pricing simulations: A/B test different usage tiers and pricing models before live deployment
- Orb Simulations for backtesting and scenario modeling against historical usage
- Invoice generation with automatic dunning
- Pending changes and draft invoice previews
- Subscription lifecycle management (trials, upgrades, downgrades, prorations)
- Multi-entity and multi-currency support
- Real-time analytics and revenue insights

**Differentiating features**
- Pricing experimentation as a core platform feature; A/B test revenue strategies before deploying
- Purpose-built from ground up for usage-based pricing (not retrofitted)
- Raw event data model enabling complex monetization workflows
- Endpoint previews to simulate API call results before execution
- Exceptional API documentation and developer experience

**UX patterns**
- API-first design with strong visual dashboard for configuration
- Experimentation-centric workflows encouraging iterative pricing optimization
- Clear separation of metering infrastructure from pricing logic
- Usage-based pricing experiments as first-class feature

**Integration points**
- REST API for event ingestion and querying
- Cloud storage integrations (S3) for bulk event processing
- Webhooks for billing events
- SDKs for Python, JavaScript, Java, Go, Ruby
- Integration with payment gateways and accounting systems

**Known gaps**
- Requires engineering investment for initial implementation
- No self-hosted option (cloud-only)
- Less maturity in compliance and revenue recognition automation vs. Zuora
- Entitlement management (feature gating) is secondary to metering

**Licence / IP notes**
- Proprietary commercial software; no known patent encumbrances.

---

### Chargebee

**Core features**
- Flexible subscription models with add-ons (multiple add-ons per subscription)
- Usage-based metering with near real-time aggregation on billions of events (2026 update)
- Schemaless usage ingestion allowing new metrics without pipeline reworking
- Metered billing for usage-based line items alongside fixed charges
- Multi-plan subscriptions in single checkout session
- Dunning and collection automation
- Revenue recognition (ASC 606 / IFRS 15)
- Hosted billing portal for customer self-service
- Tax handling and compliance
- Payment gateway integrations (20+ providers)

**Differentiating features**
- Broad integration ecosystem (20+ payment gateways globally)
- Mature platform with strong SaaS finance workflow support
- 2026 push into AI monetization with token/API call billing
- Usage visibility APIs exposing trends and overage insights in customer portals
- Invoice estimation with usage-based projections

**UX patterns**
- Configuration-heavy but flexible interface
- Clear subscription/usage separation in billing model
- Portal-centric customer experience (self-service updates, billing visibility)
- Progressive disclosure for advanced features

**Integration points**
- 20+ payment gateway integrations
- REST API for subscriptions, customers, usage, invoicing
- Webhooks for billing events
- Client libraries: Node, Python, PHP, Java, Go, Ruby, .NET
- Framework adapters (Laravel, Next.js)
- Accounting system integrations

**Known gaps**
- Usage-based billing is secondary feature, not primary (subscription-centric design)
- Complex pricing models require manual configuration
- Surprise overage fees at higher tiers (aggressive pricing tier structure)
- Support and feature access gated by plan tier
- UI complexity can be steep for non-technical users

**Licence / IP notes**
- Proprietary commercial software; no known patent concerns.

---

### Zuora

**Core features**
- Multi-entity, multi-currency subscription billing at enterprise scale
- Complex mediation and rating engines for hybrid billing (subscriptions + usage + minimums + overages, tiered pricing)
- ASC 606 and IFRS 15 revenue recognition automation
- Detailed compliance reporting and audit trails
- Payment orchestration supporting 40+ payment methods
- Intelligent retry logic protecting recurring revenue
- Subscription management with lifecycle automation
- Tax handling with country-specific templates
- Revenue dashboard and financial reporting

**Differentiating features**
- Enterprise-grade revenue recognition (ASC 606/IFRS 15) as core feature
- Multi-entity billing and consolidated reporting for large organizations
- Country-specific tax templates and compliance automation
- Deep integration with finance workflows (GL, revenue management)
- Authoritative source for SaaS metrics (ARR, MRR, NRR)

**UX patterns**
- Enterprise-focused configuration and deployment model
- Finance and RevOps team-centric workflows
- Deep reporting and analytics for CFO/controller audiences
- Complex onboarding; typically requires systems integrator support

**Integration points**
- REST API v1 for subscriptions, billing, payments, revenue
- Revenue APIs for ERP data integration
- OAuth 2.0 authentication
- Webhooks for billing and payment events
- Native connectors to major ERPs and GL systems
- SDKs: Python, Java, JavaScript, Ruby, Go

**Known gaps**
- Expensive ($50K–$200K+/year) limiting adoption to enterprise
- Slow to implement; typically 6–12 months for large deployments
- Less developer-friendly than Orb or Metronome
- Overly complex for simpler use cases (small SaaS, API companies)
- Innovation slower than smaller, more focused platforms

**Licence / IP notes**
- Proprietary enterprise software; no known patent concerns.

---

### Lago

**Core features**
- Real-time usage tracking and metering
- Hybrid pricing (usage-based, subscription, add-ons combined)
- Automated invoice generation with fees, taxes, and customer info
- Prepaid credit management (credit wallets, top-ups, expirations, rollover)
- Event ingestion at scale with deduplication
- Subscription management with lifecycle automation
- Payment-agnostic architecture (Stripe, Adyen, GoCardless, etc.)
- Self-hosted and cloud-hosted deployment options
- Open-source codebase enabling full customization

**Differentiating features**
- Open-source Apache 2.0 licence enabling full code inspection and customization
- Self-hosted or fully on-premise/air-gapped deployment
- Zero vendor lock-in; you own your billing infrastructure
- Payment-agnostic design (choose your payment provider)
- Cost-effective for engineering-heavy teams (free self-hosted)

**UX patterns**
- Developer-centric configuration via YAML/API
- Self-service infrastructure; team bears hosting and operations burden
- Full transparency into billing logic (open-source advantage)
- Gradual feature adoption; start with metering, add revenue recognition later

**Integration points**
- REST API for metering, subscriptions, invoicing
- Python, JavaScript, Node.js SDKs
- Webhook support for event notifications
- Payment gateway adapters (Stripe, Adyen, GoCardless)
- ERP and accounting integrations via webhooks and custom connectors
- OpenTelemetry observability

**Known gaps**
- Hosting and operations burden on customer (no managed service)
- Smaller ecosystem vs. commercial platforms
- Limited commercial support (community-driven development)
- Revenue recognition and compliance automation less mature than Zuora
- Requires engineering expertise to deploy and maintain

**Licence / IP notes**
- Apache 2.0 open-source licence; full transparency and no IP encumbrances. Can be self-hosted with zero licensing costs.

---

### Maxio (fka Chargify + SaaSOptics)

**Core features**
- Subscription billing with flexible pricing (flat, tiered, volume-based, hybrid)
- Usage-based billing with unified usage tracking and management
- No-code pricing customization (bundles, usage-based options)
- Dunning and collection automation
- Revenue recognition (ASC 606 / IFRS 15)
- SaaS metrics dashboards (MRR, ARR, churn, LTV)
- Revenue waterfalls and cohort analysis
- Integration with 20+ payment gateways and multiple currencies
- Financial operations and reporting focused on B2B SaaS

**Differentiating features**
- Strong financial operations and SaaS metrics focus (metrics-driven platform)
- Revenue recognition as core feature with detailed audit trails
- Integration ecosystem (GL systems: QuickBooks, Xero, Netsuite, Sage Intacct)
- Hosted billing portal for customer self-service
- Cohort-based revenue analysis

**UX patterns**
- Finance and RevOps team focused UI and workflows
- Metrics-centric dashboards for CFO/controller audiences
- Streamlined dunning and collection workflows
- Self-service customer portal for billing management

**Integration points**
- REST API for subscriptions, billing, usage, and revenue
- 20+ payment gateway integrations
- GL system connectors (QuickBooks, Xero, Netsuite, Sage Intacct)
- CRM integrations (Hubspot, Salesforce)
- Webhook support
- SDKs for common languages

**Known gaps**
- Less developer-friendly than Orb or Stripe
- Usage-based billing less native than Orb or Metronome
- Smaller ecosystem vs. Chargebee
- Custom pricing less flexible than Orb
- Less innovation velocity vs. specialist platforms

**Licence / IP notes**
- Proprietary commercial software; no known patent concerns.

---

### Flexprice

**Core features**
- No-code usage-based pricing configuration (prompt-to-plan via AI)
- Real-time metering supporting up to 60,000 events per second
- Credit-based pricing (credit wallets, top-ups, expiration, rollover)
- Entitlement management for feature gating and usage enforcement
- Hybrid pricing support (subscriptions + usage + credits combined)
- A/B pricing experimentation
- Self-hosted and managed cloud options
- Open-source codebase (Apache 2.0)
- Developer-first implementation (reduces billing engineer overhead)

**Differentiating features**
- AI-powered "prompt-to-plan" pricing configuration (describe pricing, Flexprice configures it)
- No-code pricing changes without engineering involvement
- Credit wallet as first-class billing primitive (ideal for AI and token-based pricing)
- High-throughput metering (60,000 events/sec) suitable for AI inference platforms
- Open-source with both self-hosted and managed options

**UX patterns**
- Product manager and pricing team focused (non-engineer friendly)
- Natural language pricing specification
- Visual dashboard for pricing experiment results
- Minimal ongoing engineering involvement (key differentiator)

**Integration points**
- REST API for metering, pricing, subscriptions, invoicing
- Python, JavaScript SDKs
- Webhook support
- Payment gateway agnostic (Stripe, Adyen, etc.)
- Open-source extension points

**Known gaps**
- Newer entrant; smaller ecosystem vs. established platforms
- Limited compliance and revenue recognition automation (not focus area)
- Self-hosted option requires DevOps infrastructure
- Community support smaller than Lago or LitmusChaos

**Licence / IP notes**
- Apache 2.0 open-source (platform); proprietary managed cloud. No known patent concerns.

---

### Stigg

**Core features**
- Entitlement management as core feature (feature gating, usage limits, access control)
- Advanced metering for complex usage aggregations
- Hybrid pricing support (base + usage + metered charges)
- Subscription lifecycle management (trials, upgrades, downgrades, prorations, custom contracts)
- Plan management and pricing experimentation
- Pre-built UI components (pricing tables, customer portal, widgets)
- Billing system integration (connects to third-party billing backends)
- Real-time usage tracking and reporting

**Differentiating features**
- Separates entitlement logic from billing logic (unique positioning)
- Hybrid pricing combining entitlements and advanced metering
- Pre-built UI components for fast customer-facing deployment
- Flexible plan configuration without engineering changes
- Integration with multiple billing backends (not locked to one platform)

**UX patterns**
- Product manager and growth team focused
- Plan configuration UI designed for non-engineers
- Pre-built components for rapid pricing page deployment
- Real-time usage visibility in customer portal

**Integration points**
- REST API for entitlements, metering, subscriptions
- SDKs for JavaScript, Python, Go
- Pre-built React components and UI widgets
- Integration with billing backends (Stripe, Chargebee, etc.)
- Webhook support
- CRM and analytics integrations

**Known gaps**
- Requires integration with separate billing platform (not standalone)
- Newer entrant (smaller ecosystem, customer base)
- Usage-based metering less sophisticated than Orb
- Limited revenue recognition and compliance features
- Best suited for self-serve and growth-led motions (not enterprise contracts)

**Licence / IP notes**
- Proprietary commercial software; no known patent concerns. Strategic partnership with Auth0.

---

### OpenMeter

**Core features**
- Real-time metering engine for CloudEvents format
- Event deduplication (source + id uniqueness)
- Flexible meter aggregation (SUM, COUNT, AVG, MIN, MAX)
- Real-time usage query API
- Subscription management primitives
- Invoice generation
- CNCF CloudEvents specification integration
- Open-source deployment options
- Payment-agnostic (integrates with any payment provider)

**Differentiating features**
- CloudEvents standardization (CNCF-aligned event format)
- Purpose-built for real-time metering without other features
- Strong focus on event deduplication and correctness
- Unopinionated architecture (integrates with your billing backend)

**UX patterns**
- Developer and infrastructure engineer focused
- Event-driven architecture alignment
- CloudEvents specification adherence
- Minimal opinionation on business logic

**Integration points**
- REST API for event ingestion and meter querying
- CloudEvents format compliance
- Webhook support
- Integration with any billing system
- SDKs for common languages
- Payment gateway agnostic

**Known gaps**
- Not a complete billing platform (requires separate invoicing, subscription, payment processing)
- Smaller ecosystem vs. full-stack platforms
- Limited business features (no entitlements, pricing experiments)
- Requires custom development for most use cases
- Community-driven support (smaller than commercial platforms)

**Licence / IP notes**
- Apache 2.0 open-source licence; full transparency, no IP encumbrances.

---

### Metronome (now Stripe, Jan 2026)

**Core features**
- Developer-first usage rating engine for contract-complex billing
- Extreme precision for complex enterprise contracts
- Real-time metering with high-throughput event processing
- Contract interpretation and automated rating
- Revenue recognition support
- Advanced pricing experimentation
- Multi-dimensional billing (multiple meters per contract)
- Subscription and usage billing combined

**Differentiating features**
- Purpose-built for extreme contract complexity (before Stripe acquisition)
- Developer-first API design with excellent documentation
- Precision and correctness as foundational principle
- High-volume event processing without data loss
- Advanced pricing flexibility for AI and API companies

**UX patterns**
- Developer and API-first design
- Experimentation-centric workflows
- Contract-as-code paradigm
- Technical audience (requires engineering investment)

**Integration points**
- REST API for metering, contracts, pricing experiments
- Event ingestion at massive scale (100M+ events/month)
- SDKs for Python, JavaScript, Go, Java
- Integration with revenue recognition systems
- Webhook support

**Known gaps**
- Heavy implementation burden (minimum $10K/year, typically higher)
- Now integrated into Stripe; Metronome branding/independence decreasing
- Less focus on compliance and finance features vs. Zuora
- Best fit for API/AI companies at scale (not SMB)
- Customer success and onboarding resource-intensive

**Licence / IP notes**
- Proprietary (acquired by Stripe); complex contract handling IP now under Stripe control. ~$1B acquisition signals valuable IP portfolio.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

- **Usage-based metering**: All platforms support recording consumption events and generating usage-based charges; table-stakes in post-2021 SaaS pricing.
- **Subscription management**: Core subscription lifecycle (create, update, renew, cancel, proration) is baseline across all platforms.
- **Flexible pricing models**: Hybrid models combining fixed subscriptions + usage charges + add-ons are now expected.
- **Automated invoicing**: Invoice generation, tax handling, and payment collection are baseline automations.
- **API-first design**: REST APIs with event ingestion are universal; proprietary UIs are secondary.
- **Payment gateway integrations**: Support for multiple payment providers (Stripe, Adyen, etc.) is expected.
- **Real-time billing**: Batch-based monthly billing is superseded by event-driven, near-real-time charges.
- **Dunning and retry logic**: Automated payment retry and customer retention workflows are expected.

### Differentiating Features

- **Pricing experimentation** (Orb, Stripe post-Metronome, Flexprice): A/B test pricing changes before deployment; rarely offered in 2021, now competitive advantage.
- **Entitlement management** (Stigg, Flexprice, Orb emerging): Gate features and enforce usage limits directly from billing platform; separates billing from product access control.
- **Revenue recognition automation** (Zuora, Chargebee, Maxio, Lago emerging): ASC 606/IFRS 15 compliance with audit trails; critical for regulated companies but expensive.
- **Contract complexity** (Metronome, Zuora): Handle multi-dimensional contracts with custom terms, seat minimums, usage tiers, and commitment periods without engineering.
- **No-code configuration** (Flexprice, Stigg, Chargebee 2026): Enable product/pricing teams to change plans without engineering tickets (competitive advantage for growth-led teams).
- **Self-hosting option** (Lago, Flexprice, open-source variants): Control over infrastructure, data residency, and customization without vendor lock-in.
- **AI pricing optimization** (Emerging across platforms 2026): Automated pricing recommendations based on usage patterns and market data (e.g., Stripe's AI usage billing).

### Underserved Areas / Opportunities

- **Predictive revenue recognition**: No platform flags ASC 606 compliance risks in real-time as usage arrives; all perform compliance at period-close. Opportunity for AI-driven early warning.
- **Anomaly detection in usage patterns**: No platform proactively detects fraud, runaway usage, or billing errors before invoices generate. Opportunity for ML-powered guardrails.
- **Natural-language querying of billing data**: Finance and RevOps teams resort to SQL or BI tools to answer billing questions. Opportunity for LLM-powered analytics interface.
- **Automatic contract-to-billing mapping**: CPQ and CRM systems contain free-text deal terms; no platform automatically maps these into billing engine config (Metronome addresses partially).
- **Multi-currency and consolidation reporting**: Zuora leads here; competitors underserve global SaaS companies needing consolidated USD reporting across regional entities.
- **Autonomous pricing optimization**: Current platforms enable experimentation; no platform autonomously recommends and tests pricing changes based on expansion/churn data and market signals.
- **Feature-to-billing binding**: No platform tracks which product features were used by each customer to automatically inform pricing (e.g., "3 AI features used → charge premium tier"). Opportunity for product-billing alignment.
- **Credit wallet sophistication**: Limited platforms treat credit wallets as first-class (Flexprice does); opportunity for advanced credit models (expiration windows, rollover rules, credit splits).

### AI-Augmentation Candidates

- **Automated pricing experimentation**: Continuously test price points and usage tiers against conversion, churn, and expansion data; recommend changes without manual analysis.
- **Anomaly detection**: Flag unusual usage spikes, potential billing errors, or fraud patterns before invoicing.
- **Natural-language analytics**: Allow finance/RevOps teams to query billing and revenue data via conversational interface without SQL expertise.
- **AI-assisted contract interpretation**: Map free-text deal terms from CRM/CPQ into billing engine configuration automatically.
- **Predictive revenue recognition**: Flag ASC 606 compliance risks in real-time as events arrive, rather than at period close.
- **Intelligent dunning**: Predict optimal retry timing, payment method, and messaging for failed payment recovery based on customer history and cohort patterns.
- **Feature-to-billing mapping**: Automatically correlate product feature usage with billing tier to optimize pricing and identify upsell opportunities.
- **Churn prediction with pricing intervention**: Identify at-risk customers and recommend targeted pricing or plan adjustments to prevent churn.

---

## Legal & IP Summary

All commercial platforms (Stripe, Orb, Chargebee, Zuora, Maxio, Flexprice cloud, Stigg, Metronome/Stripe) operate under proprietary SaaS licences with no known patent claims flagged in public domain. Stripe's $1B Metronome acquisition signals high-value IP around contract-complex billing. Open-source platforms (Lago, Flexprice OSS, OpenMeter) are licensed under Apache 2.0 with no IP encumbrances, enabling full customization and self-hosting without restrictions. Chargebee's 2026 pivot toward AI monetization and Stripe's AI usage billing (private preview) signal both companies are developing AI-native billing features that may eventually carry patent protections. No significant licence incompatibilities identified between platforms when integrating with third-party payment gateways or accounting systems.

---

## Recommended Feature Scope

### Must-have (MVP)

- **Real-time usage metering** supporting at least 10,000 events/sec with sub-second latency for event ingestion and charge calculation
- **Hybrid pricing engine** supporting flat subscriptions + usage-based charges + add-ons + tiered pricing in a single invoice
- **Subscription lifecycle management** (create, update, renew, cancel, proration calculations)
- **Invoice generation** with automatic tax calculation and customer-facing details
- **REST API** for usage ingestion, billing configuration, subscription management, and reporting
- **Payment gateway integration** with at least 2 major providers (Stripe, Adyen)
- **Dunning and retry logic** to reduce involuntary churn from payment failures
- **Basic compliance tracking** (audit logs, immutable billing records for ASC 606/IFRS 15 alignment)

### Should-have (v1.1)

- **Pricing experimentation UI** enabling A/B testing of pricing changes before production deployment
- **Entitlement management** for feature gating and usage enforcement separate from invoicing
- **No-code pricing configuration** allowing non-engineers to modify pricing plans
- **Credit wallet / prepaid credits** for AI and token-based pricing models
- **Advanced metering** (aggregations: SUM, AVG, MIN, MAX; custom metric definitions)
- **Revenue recognition automation** for ASC 606/IFRS 15 compliance with audit trails
- **Customer self-service portal** for viewing usage, invoices, and plan changes
- **Anomaly detection** for billing errors, fraud, or runaway usage alerts
- **Export and reporting**: CSV, PDF, and BI tool integrations (Tableau, Looker)

### Nice-to-have (backlog)

- **Multi-entity and multi-currency** consolidated reporting for global SaaS companies
- **Advanced contract management** for complex enterprise deals with custom terms and minimums
- **Autonomous pricing optimization** via AI recommendations based on usage and market data
- **Natural-language analytics** interface for finance teams to query billing data conversationally
- **Feature-to-billing binding** automatically mapping product feature usage to pricing tier recommendations
- **Self-hosted deployment option** for data residency and compliance requirements
- **ERP and GL integrations** with QuickBooks, Xero, NetSuite, Sage Intacct
- **Churn prediction with pricing intervention** recommending targeted offers to at-risk customers

---

## Sources

- [Stripe Billing Documentation](https://docs.stripe.com/billing)
- [Orb Billing Platform](https://www.withorb.com)
- [Chargebee Subscription Billing](https://www.chargebee.com)
- [Zuora Subscription Management](https://www.zuora.com)
- [Lago Open-Source Billing](https://getlago.com)
- [Maxio Billing Platform](https://www.maxio.com)
- [Flexprice Usage-Based Pricing](https://flexprice.io)
- [Stigg Entitlement Management](https://www.stigg.io)
- [OpenMeter Metering Engine](https://openmeter.io)
