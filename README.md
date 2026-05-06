# SaaS Billing & Metering

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native usage-based billing and metering engine for SaaS and AI companies that need real-time pricing without vendor lock-in.

SaaS Billing & Metering is a developer-first platform for ingesting consumption events, configuring hybrid pricing models, and generating compliant invoices. It targets engineering, finance, and RevOps teams at API-first and AI companies that have outgrown subscription-only billing tools but cannot justify the cost or implementation burden of enterprise platforms like Zuora or Metronome.

---

## Why SaaS Billing & Metering?

- **Incumbents bolt usage onto subscription cores.** Stripe Billing and Chargebee treat usage-based billing as a secondary feature; the underlying models remain subscription-centric, making complex hybrid pricing brittle.
- **Enterprise platforms are expensive and slow to deploy.** Zuora typically costs $50K–$200K+ per year and takes 6–12 months to roll out; Metronome (now Stripe) starts at a $10K/year minimum with significant engineering investment.
- **Open-source options carry an operational burden.** Lago and OpenMeter are Apache 2.0 licensed but ship hosting, scaling, and compliance work back to the customer.
- **Consolidation is shrinking buyer choice.** Stripe's ~$1B Metronome acquisition in January 2026 narrowed the pool of independent, developer-focused metering engines.
- **Pricing changes still require engineers.** Most platforms force engineering tickets for every plan or tier change, slowing growth-led pricing experimentation.

---

## Key Features

### Real-Time Metering & Pricing

- High-throughput event ingestion with sub-second latency for charge calculation
- Flexible meter aggregations (SUM, COUNT, AVG, MIN, MAX) over raw event streams
- Event deduplication aligned with the CloudEvents specification
- Hybrid pricing engine combining flat subscriptions, usage charges, add-ons, and tiered rates in a single invoice
- Credit wallets with top-ups, expiration, and rollover for AI and token-based pricing

### Subscription & Invoice Lifecycle

- Subscription creation, updates, renewals, cancellations, and proration calculations
- Automated invoice generation with tax handling and customer-facing details
- Dunning and intelligent payment retry logic to reduce involuntary churn
- Customer self-service portal for usage visibility, invoices, and plan changes
- Payment-gateway-agnostic design supporting Stripe, Adyen, and others

### Pricing Experimentation & Entitlements

- A/B testing of pricing changes with backtesting against historical usage
- No-code pricing configuration for product and RevOps teams
- Entitlement management for feature gating and usage limit enforcement
- Plan management with trials, upgrades, downgrades, and custom contracts

### Compliance & Reporting

- Audit logs and immutable billing records aligned with ASC 606 and IFRS 15
- Revenue recognition automation with audit trails
- Export to CSV, PDF, and BI tools (Tableau, Looker)
- Multi-entity and multi-currency consolidated reporting

### Developer Experience

- REST API for event ingestion, billing configuration, subscriptions, and reporting
- SDKs for common languages (Python, JavaScript, Go, Java)
- Webhooks for billing and payment events
- Self-hosted and managed cloud deployment options

---

## AI-Native Advantage

AI is woven into the platform rather than appended to it. Automated pricing experimentation continuously tests price points and tiers against conversion, churn, and expansion data, surfacing recommendations without manual analysis. Anomaly detection flags fraud, runaway usage, and billing errors before invoices are generated, while a natural-language analytics interface lets finance and RevOps teams query billing data without SQL or BI expertise. AI-assisted contract interpretation maps free-text deal terms from CRM and CPQ systems into billing engine configuration, and predictive revenue recognition flags ASC 606 compliance risks in real time as events arrive instead of at period close.

---

## Tech Stack & Deployment

The platform is designed for both self-hosted and managed cloud deployment, giving teams a path from full data residency control to fully managed operations. Event ingestion follows the CNCF CloudEvents specification, and integrations are payment-gateway-agnostic so customers can choose Stripe, Adyen, GoCardless, or another provider. SDKs target Python, JavaScript, Go, and Java; REST APIs cover metering, pricing, subscriptions, and revenue, with webhooks for downstream systems and connectors for ERPs and GL systems such as QuickBooks, Xero, NetSuite, and Sage Intacct.

---

## Market Context

The global usage-based billing software market is projected to exceed $7 billion by 2026, driven by the shift from seat-based to consumption-based pricing across SaaS and AI companies, with 63% of SaaS companies now using usage-based or hybrid pricing (up from 45% in 2021). Incumbent pricing spans percentage-of-revenue models (Chargebee free to $250K then 0.75%), platform fees with per-event pricing (Metronome from $10K/year minimum), and enterprise contracts ($50K–$200K+ for Zuora). Primary buyers are engineering leads at API-first and AI companies, finance and RevOps teams at scaling SaaS companies, and CFOs of mid-market SaaS needing accurate revenue recognition for audits.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
