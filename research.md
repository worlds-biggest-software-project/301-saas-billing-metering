# SaaS Billing & Metering

> Candidate #301 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Stripe Billing | Subscription and usage billing built into Stripe payments; acquired Metronome in Jan 2026 | Commercial SaaS | % of revenue + per-transaction | Strength: enormous ecosystem and developer familiarity. Weakness: usage-based features bolted on, not native |
| Orb | Real-time usage metering, billing, invoicing, and pricing experimentation for API/AI companies | Commercial SaaS | Custom enterprise pricing | Strength: native metering + pricing experiments in one tool. Weakness: requires engineering investment to implement |
| Metronome (now Stripe) | Developer-first usage rating engine built for high-volume, contract-complex billing; acquired by Stripe ~$1B | Commercial SaaS | Min. $10K/year; enterprise custom | Strength: extreme precision for complex contracts. Weakness: heavy implementation, now locked into Stripe stack |
| Chargebee | Subscription billing suite with usage add-ons, dunning, revenue recognition | Commercial SaaS | Free to $250K billing; 0.75% thereafter; $7,188+/yr plans | Strength: broad integrations, mature platform. Weakness: usage-based billing is secondary, not primary |
| Zuora | Enterprise subscription and revenue management platform | Commercial SaaS | $50K–$200K+/year | Strength: multi-entity, multi-currency, compliance. Weakness: expensive, slow to implement |
| Lago | Open-source usage-based billing engine; self-hostable | Open Source / Cloud | Free self-hosted; cloud paid | Strength: no vendor lock-in, full control. Weakness: owner bears hosting and engineering burden |
| Maxio (fka SaaSOptics + Chargify) | Billing and financial operations platform for B2B SaaS | Commercial SaaS | Custom pricing | Strength: strong revenue recognition and SaaS finance workflows. Weakness: less developer-friendly than Orb/Metronome |
| Flexprice | No-code usage-based pricing platform for AI and SaaS companies | Commercial SaaS | Custom pricing | Strength: no-code pricing changes. Weakness: newer entrant, smaller ecosystem |
| Stigg | Entitlement management and pricing layer that sits above billing providers | Commercial SaaS | Custom pricing | Strength: separates entitlement logic from billing logic. Weakness: requires integration with a billing backend |
| Ordway | Automated billing and revenue operations for complex B2B contracts | Commercial SaaS | Custom pricing | Strength: handles hybrid subscription + usage + milestone billing. Weakness: less known than Zuora/Chargebee |

## Relevant Industry Standards or Protocols

- **ASC 606 / IFRS 15** — Revenue recognition standards that govern how usage-based and subscription revenue must be recognised; billing engines must produce compliant audit trails
- **PCI DSS** — Payment Card Industry Data Security Standard; any engine that handles card payments or stores billing data must comply
- **OpenMetering / CloudEvents** — Event-driven usage ingestion patterns increasingly adopted for feeding raw events into billing engines
- **Webhook / REST APIs** — De-facto integration mechanism between metering pipelines, CRMs, and finance systems
- **GAAP SaaS Metrics Definitions** — ARR, MRR, churn, NRR definitions; billing engines are the authoritative source for these numbers

## Available Research Materials

1. Dodopayments (2026). *8 Best Usage-Based Billing Platforms 2026 (Stripe, Orb, Metronome, Lago Compared)*. Dodo Payments Blog. https://dodopayments.com/blogs/best-billing-platform-usage-based-pricing
2. Schematic HQ (2026). *6 Best Metered Billing Software for SaaS Companies in 2026*. Schematic Blog. https://schematichq.com/blog/metered-billing-software
3. Schematic HQ (2026). *Usage-Based Billing Explained for SaaS Teams (2026 Guide)*. Schematic Blog. https://schematichq.com/blog/why-usage-based-billing-is-taking-over-saas
4. Meteroid (2026). *Best SaaS Billing Systems in 2026: The Complete Guide*. Meteroid Blog. https://www.meteroid.com/blog/best-saas-billing-systems-in-2026-the-complete-guide
5. Flexprice (2026). *7 Best Enterprise Billing Software For AI And SaaS In 2026*. Flexprice Blog. https://flexprice.io/blog/best-enterprise-billing-software-for-ai-and-saas
6. Ordway Labs (2026). *Usage-Based Pricing for SaaS: Eight Things to Know Before You Adopt*. Ordway Blog. https://ordwaylabs.com/blog/usage-based-pricing-for-saas/
7. Alguna (2026). *9 Best Usage Based Billing Software (2026 Deep Dive)*. Alguna Blog. https://blog.alguna.com/usage-based-billing-software/
8. Lago (2026). *Top 7 Alternatives to Zuora for Usage-Based Billing*. Lago Blog. https://getlago.com/blog/top-7-alternatives-to-zuora-for-usage-based-billing

## Market Research

**Market Size:** The global usage-based billing software market is projected to exceed $7 billion by 2026, driven by the shift from seat-based to consumption-based pricing models across SaaS and AI companies.

**Funding:** Stripe acquired Metronome for approximately $1 billion in January 2026. Orb, Lago, and Flexprice have all raised venture rounds, reflecting strong investor appetite for infrastructure plays in this space.

**Pricing Landscape:** Three tiers dominate: (1) percentage-of-revenue models (Chargebee free to $250K then 0.75%); (2) platform fees with per-event pricing (Metronome min. $10K/year); (3) enterprise contracts ($50K–$200K+ for Zuora). Open-source self-hosting (Lago) has emerged as a fourth category.

**Key Buyer Personas:** Engineering leads at API-first or AI companies needing real-time metering; finance/RevOps teams at scaling SaaS companies moving from flat subscriptions to hybrid models; CFOs of mid-market SaaS needing accurate revenue recognition for audits.

**Notable Trends:** 63% of SaaS companies now use usage-based or hybrid pricing, up from 45% in 2021. Stripe's Metronome acquisition signals consolidation. Buyers increasingly demand pricing changes without engineering, credit wallets, entitlement management, and multi-entity billing as baseline capabilities.

## AI-Native Opportunity

- Automated pricing experimentation: an AI layer that continuously tests price points and usage tiers against conversion and expansion data, recommending changes without manual analysis
- Anomaly detection in usage patterns to surface potential fraud, runaway usage, or billing errors before invoices are generated
- Natural-language querying of billing and revenue data, allowing finance and RevOps teams to get answers without SQL or BI tool expertise
- AI-assisted contract interpretation that maps free-text deal terms (from CRM or CPQ) into billing engine configuration automatically
- Predictive revenue recognition that flags ASC 606 compliance risks in real time as usage events arrive, rather than at period close
