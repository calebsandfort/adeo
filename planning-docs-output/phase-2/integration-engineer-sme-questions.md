# Cross-SME Questions: Integration Engineer

## Questions from AI/NLP Architecture SME

### Context
AI/NLP Architecture SME provides expertise in LLM prompt design, model selection, latency optimization, and AI pipeline architecture.

### Question 1: RLS Policy Testing in CI
**Source:** AI/NLP SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer)

How should we test Postgres RLS policies in CI to ensure tenant isolation? Should we use transaction-level isolation tests or dedicated RLS test fixtures?

### Question 2: Tenant Context in Connection Pool
**Source:** AI/NLP SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer)

When using connection poolers (like PgBouncer), how do we ensure `SET app.current_tenant` persists correctly per request without session state leaking between requests?

---

## Questions from Legal Domain SME

### Context
Legal Domain SME provides domain expertise in legal practice workflows, communication patterns, and risk assessment for solo practitioners and small law firms.

### Question 3: Entity Linking for Legal
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer)

The HLRD mentions entity linking (email to case/client). For legal practice, the matching criteria should include case number patterns, client name matching, and possibly opposing counsel identification. What approach would you recommend for the matching logic — deterministic (exact match on case number) or probabilistic (fuzzy matching on names)? Legal data has specific patterns (case numbers like "CV2024-012345") that could enable deterministic matching.

### Question 4: Data Retention for Legal
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer)

Client communications, case files, and court documents may have specific retention requirements. Should the system track evaluation history and action approvals differently for legal domain (longer retention, audit trail for malpractice defense) compared to other domains?

---

## Questions from Social Media Domain SME

### Context
Social Media Domain SME provides expertise in social media management workflows, trend monitoring, brand voice preservation, and response timing.

### Question 5: Twitter/X API Simulation
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer SME)

For the demo, how should simulated Twitter data be structured to realistically represent API responses? Should we include engagement metrics (likes, retweets) as part of the trigger evaluation?

### Question 6: Historical Trend Data Detection
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer SME)

The system needs to detect "spikes" in mentions. What's the appropriate time window for spike detection (1 hour? 6 hours? 24 hours)? Should this be configurable per brand?

---

## Questions from E-commerce Domain SME

### Context
E-commerce Domain SME provides expertise in e-commerce operations, inventory management, marketplace policies, and customer communication for small sellers.

### Question 7: Product Entity Schema for Multi-Marketplace
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer SME)

How should the product entity schema handle multi-marketplace listings? A single product might have different SKUs on Shopify vs. Amazon, different inventory levels, and different review profiles. Should the entity model support marketplace-specific sub-records?

### Question 8: Simulated Data Providers
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer SME)

For the simulated data sources, how should we represent the difference between Shopify and Amazon API payloads? Should we have separate simulated providers for each, or a unified provider with marketplace configuration?

---

## Questions from UX Designer SME

### Context
UX Designer SME provides expertise in dashboard design, information hierarchy, and human-in-the-loop interaction patterns.

### Question 9: Synthetic Data Realism
**Source:** UX Designer SME Analysis (Section: Questions for Other SMEs > For IntegrationEngineer SME)

The HLRD mentions "simulated data sources" seeded with synthetic data. How realistic should the email subjects and body content be for testing the UX? Should we invest in LLM-generated synthetic emails for more varied edge cases, or is hand-crafted seed data sufficient for demonstrating the dashboard patterns?
