# Integration Engineer SME Review — Requirements Draft

**Review Date:** 2026-03-13
**Reviewer:** Integration Engineer SME
**Output Path:** `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-4/integration-engineer-sme-review.md`

---

## Executive Summary

The requirements draft demonstrates solid understanding of multi-tenant architecture patterns and data source abstraction. However, there are several critical gaps and inconsistencies that need to be addressed, particularly around connection pooling implementation, API specification, and the boundary between frontend database access and backend processing.

---

## 1. Accuracy of Multi-Tenancy and Data Isolation Requirements

### Correct Elements

- **FR-6.2 (Two-layer isolation):** Application layer + RLS is the correct defense-in-depth approach.
- **FR-6.3 (SET approach):** Using `SET app.current_tenant` at session level is the recommended pattern for Postgres multi-tenancy.
- **FR-6.4 (RLS policy reference):** Referencing `current_setting('app.current_tenant')` in RLS policies is correct.
- **FR-6.6 (Indexing):** Indexes on `tenant_id` columns are essential for RLS performance.
- **FR-6.7 (RLS testing):** CI pipeline tests for tenant isolation are critical for confidence.

### Issues Identified

#### CRITICAL: FR-6.5 — PgBouncer Not in Tech Stack

**FR-6.5** states: *"The system SHALL configure PgBouncer with `server_reset_query = RESET app.current_tenant` to prevent connection contamination between tenants."*

However, the tech stack reference (AGENTS.md) does not include PgBouncer in the infrastructure. The docker-compose.yml uses a direct TimescaleDB connection without a connection pooler.

**Recommendation:** Either:
1. Add PgBouncer to the docker-compose infrastructure, OR
2. Clarify that the requirement will be addressed in Phase 2 infrastructure hardening

**Suggested wording change:**
> **FR-6.5** The system **SHALL** configure a connection pooler (PgBouncer or equivalent) with `server_reset_query = RESET app.current_tenant` to prevent connection contamination between tenants, or implement equivalent connection reset logic at the application layer.

#### MEDIUM: FR-6.1 — Single-User Tenancy May Limit Future Growth

**FR-6.1** states: *"Each user account SHALL constitute a tenant"*

This is appropriate for Phase 1 but may cause migration pain if the system needs to support organizations/teams in the future. A user may want to collaborate with colleagues on the same rules and actions.

**Recommendation:** Add a clarifying note or future-proof the schema:
> **FR-6.1** (Note: For Phase 1, each user account constitutes a tenant. The architecture SHOULD support organization-level tenancy in future phases by adding an `organization_id` foreign key with appropriate RLS policies.)

---

## 2. Completeness of Database and API Specifications

### Missing Elements

#### CRITICAL: No API Specification

The requirements specify:
- Drizzle ORM usage (from tech stack)
- Database schema needs (implicit in FR-1 through FR-10)
- Data source provider interface (FR-10.1)

But there is **no explicit API specification** for:
- How frontend accesses tenant-scoped data (rules, actions, entities)
- How the CopilotKit proxy routes requests to the backend
- Whether backend can access the database directly or only through the frontend

Per AGENTS.md: *"Auth logic lives exclusively in the frontend (Next.js API routes + Drizzle); the backend handles AI/ML only."*

This architectural constraint means:
1. All tenant-scoped data access (rules, actions, evaluations) must go through Next.js API routes
2. The backend LangGraph pipeline receives only the necessary context, not direct DB access
3. The "data source connector" in FR-10.x likely means the backend polls for data and pushes to the frontend via CopilotKit, or the frontend ingests data through its own API routes

**Required addition:**
> **FR-X.Y** The system **SHALL** provide REST API endpoints (via Next.js API routes) for:
> - CRUD operations on rules and actions
> - Evaluation history and audit logs
> - Tenant configuration and preferences
> - Data source registration and status
>
> All endpoints **SHALL** enforce tenant isolation by querying with `tenant_id` from the authenticated session.

#### MEDIUM: Connection Pool Sizing Not Specified

Multi-tenant systems need explicit connection pool configuration:
- How many connections per tenant?
- What's the max connections for the pool?
- How to handle connection exhaustion?

**Recommended addition:**
> **FR-6.8** The system **SHALL** configure connection pool sizing appropriate for multi-tenant workloads, with per-tenant connection limits to prevent any single tenant from monopolizing database resources.

#### MEDIUM: Missing Audit Log Schema Details

FR-3.2 and NFR-4.4 mention logging approvals/rejections with timestamps and user identity, but the schema details are not specified:
- What tables store evaluation history?
- How long is audit data retained?
- Are there GDPR/data export requirements?

---

## 3. Gaps in Infrastructure and Integration Requirements

### Critical Gaps

#### 1. FR-10.x — Data Source Connector Architecture Incomplete

The requirements specify a "data source provider interface" (FR-10.1) but don't clarify the data flow:

- **For simulated data sources:** Are these generated in the backend and pushed to frontend via CopilotKit? Or does the frontend generate and submit to the rules engine?
- **For real webhook data sources (future):** Where do webhooks land? The Next.js frontend cannot receive webhooks (it's behind a browser). The backend would need a webhook endpoint.

**Recommendation:**
> **FR-10.6** The backend **SHALL** expose webhook endpoints for future real-time data sources (email webhooks, API push notifications), normalized through the provider interface before evaluation.
>
> **FR-10.7** For Phase 1 simulated sources, the system **SHALL** generate synthetic data in the backend and expose it to the frontend via the CopilotKit agent for rules evaluation.

#### 2. No Rate Limiting or Polling Architecture Details

FR-8.2 mentions "tiered polling" for Twitter/X API simulation, but:
- How is polling implemented? Background jobs? Cron? Celery?
- Rate limiting strategy for external APIs?
- Failed polling retry logic?

**Recommended additions:**
> **FR-10.8** The system **SHALL** implement a polling scheduler that supports configurable intervals per data source, with exponential backoff for failed polls.
>
> **FR-10.9** The system **SHALL** implement rate limiting for external API calls, respecting provider limits and implementing request queuing to prevent 429 errors.

#### 3. No Error Handling for Data Source Failures

What happens when:
- A simulated data source fails to generate?
- External API rate limits are hit during bulk evaluation?
- Database connection is lost mid-evaluation?

**Recommended addition:**
> **FR-10.10** The data source connector interface **SHALL** define error handling contracts including: connection failures, timeout handling, partial data handling, and retry policies.
>
> **FR-10.11** Failed data source operations **SHALL** be logged with sufficient context for debugging and surfaced to the user as actionable error messages.

---

## 4. Conflicts with Backend Architecture Best Practices

### Conflicts Identified

#### CONFLICT 1: Tenant Context in LangGraph Pipeline

**NFR-4.3** states: *"The system SHALL NOT pass tenant ID as a prompt parameter to LLMs, relying instead on database-level isolation."*

This is correct and important. However, the current architecture (per AGENTS.md) has the backend LangGraph pipeline running in FastAPI (backend container), while the database with RLS is accessed from the frontend (Next.js container).

**The tension:** How does the backend LangGraph agent access tenant-scoped data (rules, actions, entity context) without receiving tenant ID?

**Resolution paths:**
1. The frontend fetches tenant data (rules, entities) via Drizzle and passes it as context to the CopilotKit agent
2. The backend has its own database connection with tenant context set (not recommended per AGENTS.md architecture)
3. A shared service layer between frontend and backend handles data retrieval

**Recommendation:**
> **FR-6.9** The system **SHALL** implement a context injection pattern where the frontend fetches tenant-scoped data (rules, actions, entity summaries) via Drizzle and injects it into the CopilotKit conversation context for the LangGraph pipeline to use, without the backend needing direct database access or tenant context.

#### CONFLICT 2: FR-10.3 — LLM-Generated Synthetic Data May Be Over-Engineering

**FR-10.3** states: *"The synthetic email generator SHALL use LLM-generated content to capture natural language variety"*

This adds significant complexity and cost for a Phase 1 demo. For demonstration purposes, hand-crafted templates with variable substitution would achieve the same goal more reliably.

**Recommendation:**
> **FR-10.3 (Revised)** The synthetic email generator **MAY** use LLM-generated content for variety, but Phase 1 **SHALL** prioritize hand-crafted templates with variable substitution for reliability and faster iteration.

---

## 5. Additional Recommendations

### Database Schema Requirements

The requirements should specify the core tables needed:
- `tenants` / `users` (Better Auth schema + tenant mapping)
- `rules` (tenant-scoped)
- `actions` (tenant-scoped, linked to rules)
- `entities` (tenant-scoped: clients, cases, brands, products)
- `evaluations` (tenant-scoped: history of rule triggers)
- `approval_logs` (tenant-scoped: audit trail)

### Migration Path

FR-6.x assumes a shared database with RLS. If the system scales to many tenants (100+), consider:
- Database pooling solutions (PgBouncer, RDS Proxy)
- Read replicas for dashboard queries
- Background job isolation

---

## 6. Summary of Required Changes

| Priority | FR | Issue | Recommendation |
|----------|----|----|----------------|
| CRITICAL | FR-6.5 | PgBouncer not in tech stack | Add PgBouncer to infrastructure OR clarify as future requirement |
| CRITICAL | N/A | No API specification | Add REST API requirements for tenant data access |
| HIGH | FR-10.x | Data source connector architecture incomplete | Clarify webhook endpoints, polling architecture |
| HIGH | FR-6.9 | Tenant context in LangGraph pipeline unclear | Add context injection pattern requirement |
| MEDIUM | FR-6.1 | Single-user tenancy may limit future growth | Add note about future organization support |
| MEDIUM | N/A | No connection pool sizing guidance | Add connection pool configuration requirements |
| MEDIUM | FR-10.10 | No error handling for data source failures | Add error handling contract requirements |
| LOW | FR-10.3 | LLM-generated synthetic data is over-engineering | Simplify to templates for Phase 1 |

---

## Questions for Other SMEs

**For Product Manager:**
- Is the organization-level tenancy (multi-user per tenant) a planned Phase 2 feature, or should we design for it now?
- Do we need to support webhook ingestion in Phase 1, or is simulated polling sufficient?

**For Backend Architecture SME:**
- How should the LangGraph pipeline receive tenant context (rules, entities) given the constraint that backend doesn't access the database directly?
- Is there a preferred pattern for background job execution (polling scheduler) in the current stack?

---

*Review completed. Ready for PM Finalization phase.*
