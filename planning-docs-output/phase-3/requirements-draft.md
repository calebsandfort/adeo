# Adeo Requirements Document — Draft

**Project:** Adeo — AI-Powered Professional Communication Assistant
**Phase:** 3 — Requirements Synthesis
**Date:** 2026-03-13
**Status:** DRAFT for SME Review

---

## 1. Project Goal

Adeo is a domain-agnostic, AI-powered system that monitors a professional's incoming communications, evaluates them against configurable rules, and surfaces actionable items with draft responses ready for human approval. The system addresses the universal problem of critical action items buried in high-volume communication channels.

For Phase 1, the system monitors simulated email data sources across three domain presets: Legal, Social Media Management, and E-commerce.

---

## 2. Functional Requirements

### FR-1.0 Rules Engine Architecture

The system **SHALL** provide a configurable rules engine that evaluates incoming communications against user-defined or preset rules to detect actionable items.

- **FR-1.1** The system **SHALL** support rule definitions consisting of: compatible data source types, a pre-filter, an evaluation prompt, a priority level, and associated actions.

- **FR-1.2** The system **SHALL** support multiple pre-filter strategies: keyword/pattern matching and lightweight LLM-based classification (yes/no).

- **FR-1.3** The system **SHALL** use structured output schemas for rule evaluation results, including:
  - `triggered`: boolean indicating if the rule condition is met
  - `confidence`: float (0.0 to 1.0) representing confidence score
  - `extracted_context`: object containing relevant entities (dates, parties, case references)
  - `reasoning`: string providing brief justification for audit purposes

- **FR-1.4** The system **SHALL** evaluate rules in priority order, with configurable short-circuit behavior allowing high-priority rules to halt lower-priority evaluations.

- **FR-1.5** The system **SHALL** support three action selection modes:
  - **Present all**: Show every action associated with the triggered rule
  - **Context-aware selection**: Use LLM to recommend the most relevant subset of actions
  - **Custom selection prompt**: User-configured prompt for domain-specific selection logic

- **FR-1** Rules **SHALL** support enable/disable functionality without deletion.

- **FR-1.7** The system **SHALL** run pre-filters in parallel to minimize latency.

- **FR-1.8** The evaluation prompt **SHALL** include 2-3 few-shot examples (one positive trigger, one clear negative) to improve consistency without significant token overhead.

- **FR-1.9** The evaluation prompt **SHALL NOT** include chain-of-thought reasoning in production to minimize latency and token costs.

### FR-2.0 Actions Generation and Selection

The system **SHALL** generate draft responses and action items based on triggered rules, with entity context injection for personalization.

- **FR-2.1** Each action **SHALL** consist of: a human-readable description, a generation prompt, and an execution method.

- **FR-2.2** The system **SHALL** inject entity context into action generation prompts, including:
  - Entity name and type
  - Entity status
  - Key dates relevant to the entity
  - Last interaction timestamp

- **FR-2.3** The system **SHALL** generate actions using structured templates with LLM creative freedom within content sections, enforcing:
  - Required section headings
  - Domain-specific tone guidelines
  - Maximum word count constraints

- **FR-2.4** The system **SHALL** support three draft generation modes for social media domain:
  - **Fast Draft** (3-5 seconds generation): Minimal formatting, single-tap approval
  - **Standard Draft** (8-12 seconds): Platform conventions, one auto-revision cycle
  - **Enhanced Draft** (15-20 seconds): Multiple options, full revision support

- **FR-2.5** The system **SHALL** auto-select draft mode based on:
  - Trend velocity (spiking = Fast Draft)
  - Time since post (older = Standard/Enhanced)
  - Follower count (high-follower = Fast Draft priority)
  - Sentiment (negative = Fast Draft with escalation)

- **FR-2.6** The system **SHALL** generate actions in parallel with timeout handling, where failed generations are surfaced with a "tap to retry" option rather than blocking other actions.

- **FR-2.7** For legal domain actions, the system **SHALL** append a configurable disclaimer ("This draft was generated with AI assistance and requires attorney review before use") as a post-generation step, not within the prompt.

- **FR-2.8** The system **SHALL** integrate sentiment detection into the generation prompt (not as a separate call) for social media content generation.

### FR-3.0 Human-in-the-Loop (HITL) Approval Workflow

The system **SHALL** require explicit human approval before executing any generated action.

- **FR-3.1** The system **SHALL** present four approval options:
  - **Approve**: Accept the generated draft as-is and execute
  - **Edit**: Modify the draft in-place before executing
  - **Revise with feedback**: Provide natural language feedback and receive a revised draft
  - **Reject**: Dismiss the action entirely, optionally with a reason for tuning

- **FR-3.2** The system **SHALL** log all approval and rejection decisions with timestamps and user identity for audit purposes.

- **FR-3.3** The system **SHALL** support multiple revision rounds, with a cap at 5 rounds to prevent draft degradation. After 5 rounds, the system **SHALL** prompt the user to make direct edits.

- **FR-3.4** The revision workflow **SHALL** use a multi-turn conversation approach that includes: the original prompt, triggering content, the original draft, and all feedback rounds.

- **FR-3.5** The system **SHALL** prevent revision degradation by always including the original triggering content in context and flagging revisions that significantly lose key information.

### FR-4.0 Dashboard and UI Components

The system **SHALL** provide a dashboard that enables rapid scanning and decision-making for professionals checking the tool between tasks.

- **FR-4.1** The dashboard **SHALL** group action items by urgency first, then by email thread (same email showing all triggered rules/actions together).

- **FR-4.2** The dashboard **SHALL** use the amber/blue accent system:
  - Amber (urgent): Critical and high-urgency items
  - Blue (routine): Medium and low-urgency items
  - Extended for legal domain: Critical (amber + pulsing dot), High (amber), Medium (blue), Low (gray)

- **FR-4.3** Each action item card **SHALL** display:
  - Urgency badge (amber/blue pill)
  - Timestamp (relative, e.g., "5 min ago" in monospace)
  - Email subject line (truncated to 60 characters)
  - Rule name that triggered
  - Associated entity (client name, case number, brand name)
  - Single primary CTA: "Review Actions" button

- **FR-4.4** The dashboard **SHALL** show a single card when multiple rules trigger on the same email, with a badge showing "X rules triggered" and the highest-urgency indicator.

- **FR-4.5** The "Review Actions" sheet **SHALL** be implemented as a side drawer (Sheet component) that:
  - Shows extracted context in a compact summary card (always visible)
  - Provides a collapsible "Original Email" panel for verification
  - Displays all generated actions with Approve/Edit/Revise/Reject buttons
  - Preserves dashboard context behind the drawer

- **FR-4.6** The system **SHALL** detect conflicting actions from different triggered rules and present them side-by-side in the review sheet with explicit conflict indication.

- **FR-4.7** For legal domain deadline display, the system **SHALL** show countdown prominently ("3 days remaining") with the specific date available on hover.

- **FR-4.8** For review response previews, the system **SHALL** show the first 100 characters with character count, indicating platform limits (e.g., "185/200 chars").

- **FR-4.9** The dashboard **SHALL** support filtering by: urgency level, rule, entity, and status (pending/approved/rejected).

- **FR-4.10** The system **SHALL** support two testing modes:
  - **Quick Test**: Textarea input to paste/type email content and see immediate trigger results
  - **Historical Replay**: Filter all emails by triggered/not triggered status, with expandable details per email

### FR-5.0 Custom Rule and Action Creation

The system **SHALL** enable users to create custom rules and actions through the UI without requiring technical expertise.

- **FR-5.1** The system **SHALL** support three rule creation paths:
  - **Manual prompt writing**: User writes the evaluation prompt directly
  - **LLM-assisted creation**: User describes the detection goal in natural language; system generates the evaluation prompt, pre-filter keywords, and suggested priority
  - **Example-based creation**: User provides 3-5 positive examples and 2-3 negative examples; system infers the pattern and generates the prompt

- **FR-5.2** The system **SHALL** support the same three creation paths for custom actions.

- **FR-5.3** The system **SHALL** validate generated prompts using static analysis to detect:
  - Missing examples in prompt (warn: likely inconsistent)
  - Ambiguous language overuse ("may", "might", "possibly")
  - Missing output schema (warn: unparseable)
  - Conflicting instructions
  - Excessive length (>2000 tokens: warn may timeout)

- **FR-5.4** The system **SHALL** provide a "dry run" mode where newly created rules evaluate incoming emails and log results without surfacing actions to the dashboard.

### FR-6.0 Multi-Tenancy and Data Isolation

The system **SHALL** ensure complete data isolation between tenants using defense-in-depth architecture.

- **FR-6.1** Each user account **SHALL** constitute a tenant, with all rules, actions, entity data, and evaluation history scoped to the tenant.

- **FR-6.2** The system **SHALL** enforce tenant isolation at two layers:
  - **Application layer**: Drizzle ORM queries include tenant ID scoping as a standard pattern
  - **Database layer**: Postgres Row-Level Security (RLS) policies on all tenant-scoped tables

- **FR-6.3** The system **SHALL** set tenant context via `SET app.current_tenant` at the database session level, not through query context filtering.

- **FR-6.4** RLS policies **SHALL** reference `current_setting('app.current_tenant')` to ensure database-level isolation even if application-layer bugs exist.

- **FR-6.5** The system **SHALL** configure PgBouncer with `server_reset_query = RESET app.current_tenant` to prevent connection contamination between tenants.

- **FR-6.6** The system **SHALL** create indexes on `tenant_id` columns for all tenant-scoped tables to ensure RLS performance.

- **FR-6.7** The system **SHALL** include RLS isolation tests in the CI pipeline that verify:
  - Tenant A cannot read Tenant B's data
  - Tenant A can read their own data
  - Direct SQL injection cannot bypass RLS

### FR-7.0 Legal Domain Preset

The system **SHALL** ship with a legal domain preset targeting solo practitioners and small law firms.

- **FR-7.1** The legal preset **SHALL** include pre-configured rules for:
  - Cancelled/rescheduled hearing detection
  - Deadline notification (explicit and implied)
  - Client inquiry requiring response
  - Opposing counsel action items
  - Court order/filing notification

- **FR-7.2** The legal preset **SHALL** include urgency classification embedded within each rule's evaluation, with four levels:
  - **Critical**: Court-ordered deadlines, statute of limitations, emergency hearings
  - **High**: Scheduling changes, opposing counsel requests, court orders
  - **Medium**: Routine client status updates
  - **Low**: Newsletters, bar updates, CC'd documents

- **FR-7.3** The legal preset **SHALL** return confidence scores for extracted context, with thresholds:
  - Court deadlines: 85% minimum confidence; below = "verify manually"
  - Case numbers: 90% minimum confidence
  - Party names: 80% minimum confidence
  - Hearing dates: 90% minimum confidence

- **FR-7.4** The system **SHALL** implement the "return unknown rather than guess" principle: ambiguous dates ("sometime next week") trigger the rule but return null for the date with a note that the date is ambiguous.

- **FR-7.5** The legal preset **SHALL** include pre-configured actions for:
  - Draft client status update (conversational tone)
  - Draft rescheduling request (formal court format)
  - Draft response to opposing counsel (professional, minimal)
  - Draft internal note (factual, complete)
  - Flag for urgent review (notification only, no draft)

- **FR-7.6** The system **SHALL** support jurisdiction-specific court document templates, selectable by user rather than hardcoded.

### FR-8.0 Social Media Domain Preset

The system **SHALL** ship with a social media management domain preset for solo managers and small agencies.

- **FR-8.1** The social media preset **SHALL** include pre-configured rules for:
  - Trending topic keyword match
  - Brand mention spike detection
  - Negative sentiment mention
  - Influencer/high-follower engagement
  - Review site alert
  - Client campaign request

- **FR-8.2** The system **SHALL** implement tiered polling for Twitter/X API simulation:
  - Critical keywords: 30-second intervals
  - Important keywords: 2-minute intervals
  - Standard keywords: 5-minute intervals

- **FR-8.3** The system **SHALL** support three keyword configuration tiers:
  - Tier 1: Simple flat keyword list
  - Tier 2: Boolean logic (AND, OR, NOT)
  - Tier 3: Industry-suggested keyword clusters

- **FR-8.4** The social media preset **SHALL** include configurable brand voice profiles with dimensions: Formality, Humor, Personality, Expertise, Empathy.

- **FR-8.5** The system **SHALL** generate content using sentiment-aware tone mapping:
  - Positive: Match or exceed enthusiasm
  - Negative: Empathize first, offer solution
  - Neutral: Helpful and concise
  - Sarcastic: Acknowledge concern without being defensive

- **FR-8.6** The social media preset **SHALL** include pre-configured actions for:
  - Draft trend response post
  - Draft brand mention reply
  - Draft review response
  - Draft client campaign update
  - Draft escalation brief
  - Flag for real-time engagement

### FR-9.0 E-commerce Domain Preset

The system **SHALL** ship with an e-commerce domain preset for small sellers on Shopify and Amazon.

- **FR-9.1** The e-commerce preset **SHALL** include pre-configured rules for:
  - Order issue detection (payment failures, address verification, fulfillment errors)
  - Negative product review alert (configurable star threshold)
  - Inventory low-stock warning (configurable threshold)
  - Customer email requiring response
  - Refund/return request
  - Marketplace policy/listing change

- **FR-9.2** The system **SHALL** use hybrid evaluation: deterministic threshold checks for structured API fields plus LLM evaluation for semantic interpretation.

- **FR-9.3** The e-commerce preset **SHALL** implement generic rules with marketplace-specific sub-configurations, supporting both Shopify and Amazon with overrideable field mappings.

- **FR-9.4** The system **SHALL** extract issue type (sizing, damage, quality, shipping, wrong_item) as part of the rule evaluation output.

- **FR-9.5** The system **SHALL** inject product category metadata into action generation prompts, including common issues and resolution options for that category.

- **FR-9.6** The e-commerce preset **SHALL** include pre-configured actions for:
  - Draft review response (public, marketplace-compliant)
  - Draft customer service reply (private, personalized)
  - Draft refund/return approval
  - Draft supplier reorder email
  - Draft order issue resolution
  - Flag for seller action

- **FR-9.7** The system **SHALL** differentiate tone and format between review responses (public, professional, concise) and customer emails (private, warm, detailed).

### FR-10.0 Data Source Ingestion and Synthetic Data

The system **SHALL** abstract data sources behind a provider interface and include synthetic data generators for Phase 1 demo purposes.

- **FR-10.1** The system **SHALL** implement a data source provider interface that normalizes content into: source type, metadata (sender, timestamp, identifiers), and body content (text, HTML, or structured JSON).

- **FR-10.2** The system **SHALL** provide simulated data sources for all three domain presets:
  - Simulated email inbox with synthetic emails covering domain scenarios plus noise
  - Simulated Twitter/X API payloads (social media domain)
  - Simulated Shopify/Amazon seller API payloads (e-commerce domain)

- **FR-10.3** The synthetic email generator **SHALL** use LLM-generated content to capture natural language variety, with hand-crafted seed data for structured API payloads.

- **FR-10.4** The synthetic data **SHALL** include entity data (clients, cases, brands, products, orders) that is internally consistent within each domain preset.

- **FR-10.5** New synthetic data **SHALL** be receivable on a configurable schedule or triggered manually to demonstrate the system in action.

---

## 3. Non-Functional Requirements

### NFR-1.0 Performance

The system **SHALL** meet latency targets that enable responsive user experience.

- **NFR-1.1** Rule evaluation (pre-filter + full prompt) **SHALL** complete within 5 seconds per email.

- **NFR-1.2** Action generation **SHALL** complete within 10 seconds to keep the approval flow responsive.

- **NFR-1.3** The revision feedback loop **SHALL** return updated drafts within 10 seconds.

- **NFR-1.4** Pre-filter LLM calls **SHALL** use lightweight models (Qwen2.5-0.5B or MiniMax-Text-01) with sub-500ms latency.

- **NFR-1.5** The system **SHALL** show loading states with skeleton loaders during evaluation.

- **NFR-1.6** The system **SHALL** cache evaluation results for 30-60 seconds to avoid re-evaluation on page refresh.

### NFR-2.0 LLM Model Selection

The system **SHALL** use models from the Kimi, Qwen, MiniMax, and GLM families via OpenRouter.

- **NFR-2.1** Pre-filter stage **SHALL** use Qwen2.5-0.5B or MiniMax-Text-01 for fast, cost-effective yes/no classification.

- **NFR-2.2** Rule evaluation stage **SHALL** use MiniMax-M2.5 or equivalent for strong instruction-following.

- **NFR-2.3** Action generation stage **SHALL** use MiniMax-M2.5 for writing quality, with MiniMax-Text-01 as the fast alternative.

- **NFR-2.4** The system **SHALL** include fallback models if primary models are unavailable:
  - Pre-filter fallback: GLM-4-9B
  - Evaluation fallback: Qwen2.5-7B
  - Generation fallback: GLM-4-9B

### NFR-3.0 Cost Optimization

The system **SHALL** implement cost-effective inference patterns.

- **NFR-3.1** The pre-filter stage **SHALL** reduce LLM costs by filtering emails before full evaluation, targeting 80-90% cost reduction.

- **NFR-3.2** Entity context injection **SHALL** use summary approach (200-500 tokens) rather than full history.

- **NFR-3.3** Brand voice profile injection **SHALL** be capped at 200-400 tokens to avoid quality degradation from excessive context.

- **NFR-3.4** Product context injection **SHALL** be optimized to 100-200 tokens using structured key fields only.

### NFR-4.0 Security

The system **SHALL** implement defense-in-depth security including multi-layer tenant isolation.

- **NFR-4.1** All tenant-scoped tables **SHALL** have RLS policies enabled.

- **NFR-4.2** The RLS test suite **SHALL** run in CI pipeline on every push and pull request.

- **NFR-4.3** The system **SHALL** not pass tenant ID as a prompt parameter to LLMs, relying instead on database-level isolation.

- **NFR-4.4** Approved and rejected actions **SHALL** be logged with timestamps and user identity for audit.

### NFR-5.0 Usability

The system **SHALL** prioritize fast scanning and confident decision-making for time-constrained professionals.

- **NFR-5.1** The dashboard **SHALL** enable a user to scan and act on items in under 60 seconds per session.

- **NFR-5.2** Mobile experience **SHALL** be review-only (view items, view context) with no approval capability in Phase 1.

- **NFR-5.3** In-app notifications **SHALL** appear as badges on sidebar icons; critical items **MAY** support opt-in push notifications.

- **NFR-5.4** Notifications for multiple items arriving within 10 minutes **SHALL** be batched into a single notification.

### NFR-6.0 Maintainability

The system **SHALL** be designed for extensibility and maintainability.

- **NFR-6.1** Adding a new data source connector **SHALL** require only implementing the provider interface, with no changes to the rule evaluation pipeline.

- **NFR-6.2** Adding a new domain preset **SHALL** be achievable by defining domain-specific rules, actions, and entity schemas.

- **NFR-6.3** The rules engine **SHALL** support versioning of prompts to track changes over time.

- **NFR-6.4** The system **SHALL** track rule accuracy over time based on approval/rejection rates and surface high-rejection rules for review.

---

## 4. Synthesis Notes

### Technology Stack Validation

No explicit technology stack reference was provided for this synthesis. The requirements assume the established project stack (FastAPI + LangGraph backend, Next.js frontend, Drizzle ORM, TimescaleDB/PostgreSQL 16, CopilotKit for HITL primitives).

### Key Design Decisions Documented

1. **Pre-filter architecture**: Two-pass approach (keyword/pattern pre-filter then LLM evaluation) chosen over single-pass for cost optimization while maintaining accuracy.

2. **Urgency classification**: Embedded within rules rather than as a separate cross-cutting layer, per Legal Domain SME recommendation that urgency is context-dependent.

3. **Action generation tone**: Domain-specific templates with configurable voice profiles, avoiding generic outputs that could harm brand perception.

4. **Tenant isolation**: SET + RLS approach chosen over query context filtering for defense-in-depth, per Integration Engineer SME recommendation.

5. **Mobile strategy**: Review-only for Phase 1, with approval restricted to desktop, per UX Designer SME recommendation.

### Conflicts Resolved

1. **Legal domain urgency**: Extended amber/blue system rather than creating domain-specific colors (Legal SME suggested color alternatives; resolved to maintain consistency).

2. **Sentiment detection**: Integrated into generation prompt rather than separate call (AI/NLP SME initially suggested separate; resolved based on token/latency tradeoff analysis).

3. **Product context in actions**: Category + guidelines approach preferred over few-shot examples (AI/NLP SME recommended few-shot; resolved based on token optimization analysis).

---

## 5. Success Criteria

The system **SHALL** be considered successful when:

- [ ] Rule -> Action -> Approval pipeline demonstrates domain-agnostic pattern applicable to multiple professions
- [ ] Legal domain preset scenarios are realistic enough for a practicing attorney to recognize
- [ ] A non-technical user can create a working rule in under 5 minutes via the UI
- [ ] HITL approval flow (including revision) feels faster than manual writing
- [ ] Newsletter and spam emails do not trigger false positive action items at disruptive rates
- [ ] Pre-filter approach demonstrates meaningful cost reduction versus running every email through full evaluation
- [ ] Architecture clearly supports future extensibility for new data sources and domain presets

---

## Appendix: Cross-Reference to Source Documents

| Source | Sections Referenced |
|--------|-------------------|
| Original HLRD | Core Functionality, Rules Engine, Actions Engine, HITL Approval, Dashboard, Auth/Multi-tenancy |
| AI/NLP SME Analysis | Rule prompt structure, pre-filter model selection, async evaluation, conflict detection, model selection |
| Legal Domain SME Analysis | Common dropped balls, urgency classification, tone conventions, extraction accuracy |
| Social Media SME Analysis | Missed moment scenarios, keyword configuration, brand voice, speed/quality tradeoff |
| Ecommerce SME Analysis | Impact priorities, marketplace API differences, tone conventions, product context |
| UX Designer SME Analysis | Action item grouping, revision UI, rule testing, dashboard layout |
| Integration Engineer SME Analysis | Tenant context SET approach, RLS testing, connection pooling |
| Phase 2 Cross-SME Answers | All resolved questions on extraction confidence, disclaimer generation, conflict UI, etc. |

---

*Document Status: DRAFT — Ready for Phase 4 SME Review*
