# High-Level Requirements Document: Adeo

## Project Overview

**Adeo** (Latin: "I attend to") is a domain-agnostic, AI-powered system that continuously monitors a professional's incoming communications, evaluates them against configurable rules, and surfaces actionable items with draft responses ready for human approval. The system addresses a universal problem across professions: critical action items buried in high-volume communication channels get missed, leading to dropped balls, missed deadlines, and damaged client relationships.

The core insight is that most professionals already know what they should be watching for — a lawyer knows a cancelled hearing needs to be rescheduled, a property manager knows a maintenance complaint needs a work order — but the volume and noise of daily email makes it easy for important signals to slip through. This system acts as an always-on assistant that applies the professional's own rules to every incoming message, detects when action is needed, and presents ready-to-execute responses that the professional simply reviews and approves.

The system is built around two domain-agnostic primitives: **Rules** (LLM-powered evaluators that determine whether incoming content requires attention) and **Actions** (LLM-powered generators that produce draft responses or documents for human approval). Rules and Actions are configured via prompts, ship with domain-specific presets, and can be created or customized by users through the UI. Phase 1 monitors email only, but the architecture is designed for extensibility to additional data sources (APIs, court filing systems, social media, RSS feeds, etc.).

For the initial release, the system ships with a **legal domain preset** — targeting solo practitioners and small firms — that includes pre-configured rules for common scenarios like cancelled hearings, missed deadlines, discovery requests, and client follow-ups, along with actions for drafting client status updates, rescheduling motions, and response letters.

## Design System

The UI is a **light, professional workspace** — clean surfaces, restrained depth, and a dual-accent color system that encodes meaning at a glance. The overall feel is a well-organized desk: structured without being sterile, warm without being casual. The design prioritizes fast scanning and confident decision-making — this is a tool professionals check between client calls, not a dashboard they stare at all day.

---

### Theme Architecture

Theme variables live in `frontend/src/globals.css` as CSS custom properties. ShadCN components consume them via `--primary`, `--secondary`, etc. Raw Tailwind color classes are used directly for status indicators, urgency badges, and decorative accents.

**Two separate color systems are used intentionally:**
- **ShadCN theme vars** (`bg-primary`, `text-secondary-foreground`) — for interactive UI components (Button, Input, Card)
- **Raw Tailwind classes** (`bg-amber-500`, `text-blue-600`) — for status indicators, urgency badges, and semantic accents

---

### Color Palette

#### Dual Accent System — Urgency vs. Routine

The accent pair encodes the most important distinction in the UI: **does this need attention now, or is it part of the normal workflow?**

##### Amber / Orange — Urgency, Attention Required

| Use | Class |
|-----|-------|
| Urgent action item badge | `bg-amber-500 text-white` |
| Urgent card left border | `border-l-4 border-amber-500` |
| High-confidence trigger indicator | `text-amber-600` |
| Pending approval CTA | `bg-amber-500 hover:bg-amber-600 text-white` |
| Urgency dot / pulse | `bg-amber-500` with subtle pulse animation |
| Warning state | `bg-amber-50 border-amber-200 text-amber-800` |

##### Blue — Routine, Steady-State Workflow

| Use | Class |
|-----|-------|
| Primary navigation / branding | `text-blue-700` |
| Standard action item badge | `bg-blue-100 text-blue-700` |
| Routine card left border | `border-l-4 border-blue-400` |
| Secondary CTA / standard buttons | `bg-blue-600 hover:bg-blue-700 text-white` |
| Active tab / selected state | `bg-blue-600 text-white` |
| Informational state | `bg-blue-50 border-blue-200 text-blue-800` |
| Links | `text-blue-600 hover:text-blue-800` |

##### Surfaces and Neutrals

| Role | Class | Notes |
|------|-------|-------|
| Page background | `bg-stone-100` | Warm off-white, clearly distinguished from card surfaces |
| Card / panel | `bg-white` | Clean white cards on warm background |
| Card border | `border-stone-200` | Subtle, not heavy |
| Card shadow | `shadow-sm` | Restrained depth — visible but not dramatic |
| Hover elevation | `hover:shadow-md` | Gentle lift on interaction |
| Input background | `bg-white border-stone-300` | Clear input boundaries |
| Primary text | `text-stone-900` | Near-black, warm undertone |
| Secondary text | `text-stone-500` | Subdued but readable |
| Tertiary text / metadata | `text-stone-400` | De-emphasized timestamps, IDs |
| Dividers | `border-stone-200` | Consistent with card borders |
| Sidebar / nav background | `bg-stone-100` or `bg-white` | Slightly differentiated from page |

##### Status Colors (beyond the dual accent)

| Status | Class | Use |
|--------|-------|-----|
| Success / approved | `text-emerald-600` / `bg-emerald-50 border-emerald-200` | Approved actions, resolved items |
| Rejected / dismissed | `text-stone-400` / `line-through` | Rejected actions, dismissed items |
| Error | `text-red-600` / `bg-red-50 border-red-200` | System errors, failed evaluations |

---

### Typography

Adeo uses a professional, readable type system. Headings use a slightly warmer sans-serif for personality; body text prioritizes clarity at small sizes.

| Usage | Font | Classes | Notes |
|-------|------|---------|-------|
| Headings (h1–h3) | Plus Jakarta Sans | `font-semibold tracking-tight` | Warm, professional, modern |
| Body / UI text | Inter | `font-normal leading-relaxed` | Maximally readable at all sizes |
| Monospace / metadata | JetBrains Mono | `font-mono text-xs` | Rule IDs, timestamps, confidence scores |
| Action item previews | Inter | `font-normal text-stone-700` | Slightly subdued, scannable |

Fonts are configured in `globals.css`:
- `--font-sans`: Plus Jakarta Sans (headings), Inter (body)
- `--font-mono`: JetBrains Mono (metadata, technical labels)

---

### Key Patterns

#### Action Item Card
```tsx
<div className="bg-white rounded-lg border border-stone-200 shadow-sm hover:shadow-md transition-shadow">
  <div className="border-l-4 border-amber-500 p-4 flex items-start gap-4">
    <div className="flex-1">
      <div className="flex items-center gap-2 mb-1">
        <span className="inline-flex items-center px-2 py-0.5 rounded-full bg-amber-500 text-white text-xs font-medium">Urgent</span>
        <span className="text-xs text-stone-400 font-mono">2 min ago</span>
      </div>
      <h3 className="text-sm font-semibold text-stone-900">Hearing cancelled — Martinez v. Clark</h3>
      <p className="text-sm text-stone-500 mt-1">Opposing counsel cancelled the Jan 15 hearing. No reschedule proposed.</p>
    </div>
    <div className="flex gap-2">
      <button className="px-3 py-1.5 bg-amber-500 hover:bg-amber-600 text-white text-xs font-medium rounded-md">Review Actions</button>
    </div>
  </div>
</div>
```

#### Routine Item Card
```tsx
<div className="bg-white rounded-lg border border-stone-200 shadow-sm hover:shadow-md transition-shadow">
  <div className="border-l-4 border-blue-400 p-4 flex items-start gap-4">
    <div className="flex-1">
      <div className="flex items-center gap-2 mb-1">
        <span className="inline-flex items-center px-2 py-0.5 rounded-full bg-blue-100 text-blue-700 text-xs font-medium">Routine</span>
        <span className="text-xs text-stone-400 font-mono">15 min ago</span>
      </div>
      <h3 className="text-sm font-semibold text-stone-900">Client inquiry — Davis estate planning</h3>
      <p className="text-sm text-stone-500 mt-1">Client asking for status update on document preparation.</p>
    </div>
    <div className="flex gap-2">
      <button className="px-3 py-1.5 bg-blue-600 hover:bg-blue-700 text-white text-xs font-medium rounded-md">Review Actions</button>
    </div>
  </div>
</div>
```

#### Badge / Status Pill
```tsx
{/* Urgency badge */}
<span className="inline-flex items-center px-2 py-0.5 rounded-full bg-amber-500 text-white text-xs font-medium">
  Urgent
</span>

{/* Routine badge */}
<span className="inline-flex items-center px-2 py-0.5 rounded-full bg-blue-100 text-blue-700 text-xs font-medium">
  Routine
</span>

{/* Approved badge */}
<span className="inline-flex items-center px-2 py-0.5 rounded-full bg-emerald-100 text-emerald-700 text-xs font-medium">
  Approved
</span>
```

#### ShadCN Button Usage
Use ShadCN `Button` for standard interactive elements (navigation, form submissions, dialog triggers). Use native `<button>` with raw Tailwind for urgency-coded CTAs where the amber/blue semantic color must be explicit.

---

### Changing the Theme

The primary and secondary accent colors are defined in `frontend/src/globals.css`:

```css
:root {
  --primary: oklch(0.75 0.18 45);    /* amber — urgency */
  --secondary: oklch(0.55 0.2 240);  /* blue — routine */
}
```

To adjust the accent pair, change the hue values (third parameter in `oklch`). After changing CSS vars, update any hardcoded raw Tailwind classes — search for `amber-` and `blue-` and replace with the corresponding Tailwind color scale for the new hues.

## Core Functionality

### Rules Engine

#### Rule Structure
- Each rule consists of one or more **compatible data source types**, a pre-filter, an evaluation prompt, a priority level, and a list of associated actions
- Every rule must declare at least one data source type it applies to (e.g., email, Twitter, API webhook, RSS). A rule can be compatible with multiple source types if its evaluation prompt is written broadly enough — for example, a "scheduling change detection" rule might work across both email and court filing API notifications
- Only rules whose compatible source types include the incoming data's type are considered during evaluation — an email does not run through Twitter-only rules
- The **pre-filter** is a lightweight check (keyword/pattern matching or a short, inexpensive LLM call) that determines whether the full evaluation prompt should run against a given message — this prevents expensive LLM calls on irrelevant content like newsletters and spam
- The **evaluation prompt** is sent to the LLM along with the source content to determine whether the rule's condition is met — it returns a structured result including whether the rule triggered, a confidence score, and extracted context (e.g., relevant dates, parties, case references)
- Rules have a **priority level** that determines evaluation order — when new content arrives, rules are evaluated in priority order and can short-circuit (stop evaluating lower-priority rules) if a high-priority rule triggers and its actions cover the situation
- Rules can be enabled or disabled without deletion

[SME:AIWorkflow] What's the optimal structure for rule evaluation prompts to maximize detection accuracy while minimizing false positives? Should prompts include few-shot examples, chain-of-thought reasoning instructions, or structured output schemas? How do we handle the tradeoff between prompt complexity and inference cost?

[SME:AIWorkflow] For the pre-filter LLM call option, what's the right model tier? A small/fast model with a very short prompt ("Is this email potentially about a scheduling change? yes/no") could be significantly cheaper than running every email through the full evaluation prompt. What's the cost/accuracy tradeoff?

#### Rule Evaluation Pipeline
- When a new email is ingested, the system runs the pre-filter for each active rule (in priority order)
- Emails that pass a rule's pre-filter are evaluated against that rule's full prompt
- Multiple rules can trigger on the same email — each triggered rule independently surfaces its associated actions
- The system records evaluation results (which rules ran, which triggered, confidence scores) for auditability and future tuning
- Short-circuit behavior is configurable per rule: some rules may explicitly allow continued evaluation of lower-priority rules even when they trigger

[SME:AIWorkflow] Should rule evaluation be synchronous (block until all rules are evaluated) or asynchronous (surface actions as rules trigger, allowing the user to act while lower-priority rules are still evaluating)? What's the UX tradeoff?

[SME:AIWorkflow] How should we handle conflicting rules — e.g., two rules trigger on the same email with contradictory suggested actions? Should the system flag conflicts for user resolution, or should priority ordering implicitly resolve them?

#### Predefined Rules (Legal Domain Preset)
The legal preset ships with rules covering common scenarios including but not limited to:
- **Cancelled/rescheduled hearing detection** — identifies emails from courts, opposing counsel, or mediators indicating a hearing, deposition, or mediation has been cancelled or needs rescheduling
- **Deadline notification** — detects emails containing explicit or implied deadlines for filings, discovery responses, or compliance requirements
- **Client inquiry requiring response** — identifies direct client emails asking questions or requesting status updates that have not been addressed
- **Opposing counsel action items** — detects requests, demands, or notices from opposing counsel that require a response or action
- **Court order / filing notification** — identifies automated or manual notifications of new court orders, filings, or docket entries

[SME:LegalDomain] What are the most common email-driven dropped balls in solo/small firm legal practice? Which scenarios cause the most harm when missed? Are there practice-area-specific rules that should be included (e.g., family law vs. personal injury vs. criminal defense)?

[SME:LegalDomain] How do we handle the nuance of urgency in legal communications? A motion deadline is different from a courtesy copy. Should the legal preset include urgency classification within rules, or is that a separate cross-cutting concern?

#### Predefined Rules (Social Media Manager Preset)
The social media preset ships with rules covering common scenarios including but not limited to:
- **Trending topic keyword match** — monitors the Twitter/X API for trending topics or high-engagement posts that reference configured keywords, brand names, or industry terms. Urgency is high because trend relevance decays quickly.
- **Brand mention spike detection** — detects when the volume of @mentions or brand name references on Twitter/X exceeds a configurable threshold within a time window, indicating a viral moment (positive or negative) that requires a response
- **Negative sentiment mention** — identifies Twitter/X mentions or email inquiries that express complaints, frustration, or negative sentiment about the brand, product, or service
- **Influencer / high-follower engagement** — detects when an account above a configurable follower threshold mentions, quotes, or replies to the brand, indicating an opportunity for high-visibility engagement
- **Review site alert** — identifies emails from review platforms (Google Business, Yelp, Trustpilot, G2) notifying of new reviews, particularly negative reviews that need a public response
- **Client campaign request** — identifies emails from agency clients requesting new content, campaign changes, or approvals that require a timely response

[SME:SocialMediaDomain] What are the most common "missed moment" scenarios for social media managers and agencies? Is trend response speed (minutes) realistic as a use case, or is the real value in not missing overnight/weekend activity?

[SME:SocialMediaDomain] How should keyword configuration work for trend monitoring? Should users provide a flat keyword list, or should the system support boolean logic (e.g., "AI AND regulation NOT crypto")? Should the system suggest related keywords based on the brand's industry?

[SME:AIWorkflow] Twitter/X API rate limits and cost are significant constraints. How should the system balance polling frequency against API budget? Should there be a tiered approach where high-priority keywords are polled more frequently?

#### Predefined Rules (Small E-commerce Seller Preset)
The e-commerce preset ships with rules covering common scenarios including but not limited to:
- **Order issue detection** — monitors Shopify/Amazon seller API notifications for orders with problems: payment failures, address verification issues, items out of stock after purchase, or fulfillment errors that need seller intervention
- **Negative product review alert** — detects API notifications of new product reviews at or below a configurable star threshold (e.g., 1-2 stars), indicating a customer experience issue that may need a public response and/or investigation
- **Inventory low-stock warning** — detects when inventory levels reported via the Shopify/Amazon API drop below a configurable threshold for a tracked product, prompting a reorder decision before stockout
- **Customer email requiring response** — identifies direct customer emails with questions about orders, returns, product details, or complaints that haven't been addressed within a configurable time window
- **Refund / return request** — detects API notifications or customer emails requesting a refund or return, particularly for high-value orders or repeat customers where the response impacts retention
- **Marketplace policy or listing change** — identifies API notifications or emails from Shopify/Amazon indicating listing removals, policy violations, fee changes, or account health warnings that require seller action

[SME:EcommerceDomain] What are the highest-impact missed notifications for small e-commerce sellers? Is the primary pain point customer-facing (unanswered complaints, review damage) or operational (stockouts, listing issues, payment failures)?

[SME:EcommerceDomain] How should the system handle the difference between Shopify and Amazon seller APIs? Should the preset define rules generically enough to work with either, or should there be sub-configurations per marketplace?

[SME:AIWorkflow] E-commerce API data is highly structured (JSON payloads with order IDs, statuses, quantities). Should rule evaluation prompts for API data sources be structured differently than email-based prompts? Could some API rules use deterministic logic (threshold checks) rather than LLM evaluation for speed and cost?

### Actions Engine

#### Action Structure
- Each action consists of a description, a generation prompt, and an execution method
- The **description** is a human-readable summary of what the action does (e.g., "Draft a client status update email about the scheduling change")
- The **generation prompt** is sent to the LLM along with the triggering email content and extracted context to produce draft output — this could be an email body, a letter, a document, or any text-based artifact
- The **execution method** defines what happens when the user approves — for Phase 1 this means displaying the draft for copy/paste or simulated send; future phases could integrate with email APIs, document management systems, etc.
- Actions can reference entity context (client name, case number, relevant dates) extracted by the triggering rule

[SME:AIWorkflow] How should action generation prompts be structured to produce professional, domain-appropriate output? Should we use strict templates with fill-in-the-blank sections, or give the LLM creative freedom with tone/format guidelines? How do we ensure consistency across multiple generations for the same action type?

#### Action Selection
When a rule triggers, the system must decide which of its associated actions to present to the user. Three selection modes are supported:

- **Present all** — every action associated with the triggered rule is shown to the user, regardless of context. Simplest mode, appropriate when a rule has a small number of clearly distinct actions.
- **Context-aware selection** — the system sends the triggering email content and each action's description to the LLM, which recommends the most relevant subset of actions. Appropriate when a rule has many associated actions and not all are relevant to every trigger.
- **Custom selection prompt** — the user configures a prompt that describes how to analyze the triggering content and choose actions. This allows domain-specific selection logic without modifying the core system.

The selection mode is configurable per rule. The default for predefined rules is context-aware selection.

[SME:UXDesigner] How should multiple triggered rules and their actions be presented in the dashboard? Should they be grouped by rule, by email, by urgency, or by action type? What's the right information density for a professional who is scanning quickly between tasks?

#### Predefined Actions (Legal Domain Preset)
The legal preset ships with actions including but not limited to:
- **Draft client status update** — generates an email to the client explaining what happened and what the next steps are
- **Draft rescheduling request** — generates a letter or email to the court or opposing counsel requesting a new date
- **Draft response to opposing counsel** — generates a professional response acknowledging receipt and indicating next steps
- **Draft internal note** — generates a case note summarizing the email and flagged action items for the case file
- **Flag for urgent review** — creates a high-priority dashboard item with no draft, just a notification that the attorney needs to personally review the email

[SME:LegalDomain] What tone and format conventions apply to different types of legal communications? Should client updates be more conversational while court-directed documents follow stricter formatting? Are there jurisdiction-specific formatting requirements we should account for?

#### Predefined Actions (Social Media Manager Preset)
The social media preset ships with actions including but not limited to:
- **Draft trend response post** — generates a brand-voice tweet or thread that ties a trending topic back to the brand's messaging or product, optimized for engagement. Includes relevant hashtags and a suggested posting window based on trend velocity.
- **Draft brand mention reply** — generates a contextually appropriate reply to a Twitter/X mention, adjusting tone based on sentiment (thanking positive mentions, addressing concerns in negative ones, engaging playfully with neutral ones)
- **Draft review response** — generates a public response to a review site notification, following platform-appropriate conventions (professional and empathetic for negative reviews, appreciative for positive ones)
- **Draft client campaign update** — generates an email to the agency client summarizing relevant social activity (mentions, trending opportunities, review alerts) with recommended actions
- **Draft escalation brief** — generates an internal summary for the account lead or client when a brand mention spike or negative sentiment wave is detected, including volume metrics, representative posts, and suggested response strategy
- **Flag for real-time engagement** — creates a high-priority dashboard item with no draft, just an alert that a time-sensitive engagement opportunity exists (e.g., a viral mention or influencer interaction that needs a human-crafted response)

[SME:SocialMediaDomain] What tone and voice conventions should the system follow when generating social media content? Should the preset include a configurable brand voice profile (formal, playful, edgy, etc.) that shapes all generated posts? How do we prevent generated content from sounding generic or bot-like?

[SME:SocialMediaDomain] How should the system handle the speed/quality tradeoff for trend response posts? A trend response that takes 30 minutes to approve is often worthless. Should there be a "fast draft" mode that prioritizes speed over polish?

#### Predefined Actions (Small E-commerce Seller Preset)
The e-commerce preset ships with actions including but not limited to:
- **Draft review response** — generates a public response to a negative product review, following marketplace conventions (empathetic, professional, offers resolution without being defensive). Adjusts tone based on review severity and whether the issue is product-related or shipping-related.
- **Draft customer service reply** — generates a reply to a customer email about an order issue, return request, or product question, referencing order details and offering a clear next step
- **Draft refund/return approval** — generates a customer-facing email approving a return or refund, including instructions (return label, timeline, refund method) pre-populated from order context
- **Draft supplier reorder email** — generates an email to the supplier for a product approaching stockout, referencing the product, quantity needed, and standard order terms
- **Draft order issue resolution** — generates an internal note or customer email addressing a flagged order problem (payment failure, address issue, fulfillment error), with a recommended resolution path
- **Flag for seller action** — creates a high-priority dashboard item for marketplace policy violations, account health warnings, or listing removals that require the seller to take manual action in the marketplace admin panel

[SME:EcommerceDomain] What tone conventions apply to marketplace review responses vs. direct customer emails? Review responses are public-facing and affect purchase decisions for future buyers — should they follow a different structure than private customer communications?

[SME:EcommerceDomain] How should the system handle product-specific context in responses? A customer complaint about a sizing issue needs different language than a complaint about a damaged item. Should product category metadata influence action generation prompts?

### Human-in-the-Loop Approval

The HITL approval flow is implemented using CopilotKit's built-in human-in-the-loop primitives rather than a custom approval UI. This ensures the approval experience is consistent with the rest of the CopilotKit-powered interface and reduces implementation effort.

- Every action must be explicitly approved by the user before execution — the system never sends, files, or acts autonomously
- When an action is presented, the user can:
  - **Approve** — accept the generated draft as-is and execute
  - **Edit** — modify the draft in-place before executing
  - **Revise with feedback** — provide natural language feedback to the LLM (e.g., "make the tone more formal" or "mention that we need the new date before the 15th") and receive a revised draft
  - **Reject** — dismiss the action entirely, optionally with a reason (used for tuning)
- Approved and rejected actions are logged with timestamps and user identity for audit purposes
- The revision loop supports multiple rounds — the user can provide feedback, review the revision, and provide additional feedback until satisfied

[SME:AIWorkflow] How should the revision prompt be structured? Should it include the original email, the original draft, and the user's feedback as a single prompt, or should it use a multi-turn conversation approach? How do we prevent the draft from degrading over multiple revision rounds?

[SME:UXDesigner] CopilotKit's HITL components provide the approval/reject primitives, but the revision-with-feedback flow may need custom UI layered on top. What's the right balance between using CopilotKit's built-in components and building custom UI for the revision experience?

### Custom Rule and Action Creation

#### Creating Rules
Users can create custom rules through the UI via three paths:
- **Manual prompt writing** — the user writes the evaluation prompt directly, for cases where they know exactly what they want to detect
- **LLM-assisted creation** — the user describes in natural language what they want to detect (e.g., "I want to know when opposing counsel files a motion to compel"), and the system uses an LLM to generate the evaluation prompt, pre-filter keywords, and suggested priority level
- **Example-based creation** — the user provides example emails that should trigger the rule (and optionally examples that should not), and the system uses an LLM to infer the pattern and generate the evaluation prompt

All three paths produce the same output: a rule with a pre-filter, evaluation prompt, priority, and (optionally) associated actions. The user can review and edit the generated prompt before saving.

[SME:AIWorkflow] For example-based rule creation, how many examples are needed to generate a reliable evaluation prompt? Should the system actively ask for negative examples ("show me an email that looks similar but should NOT trigger this rule") to improve specificity?

[SME:AIWorkflow] How do we handle prompt quality validation? If a user writes a poor evaluation prompt manually, can we detect that it's likely to produce bad results before they deploy it? Should there be a "test rule" feature that runs it against recent emails and shows what would have triggered?

#### Creating Actions
Users can create custom actions through the same three paths as rules: manual prompt writing, LLM-assisted creation from a natural language description, or example-based creation where the user provides sample outputs and the system generates the prompt. Custom actions can be associated with any rule (predefined or custom).

#### Testing and Iteration
- Users can test rules and actions against historical emails in the system to see what would have triggered and what drafts would have been generated
- A "dry run" mode allows newly created rules to evaluate incoming emails and log results without surfacing actions to the dashboard — this lets users validate accuracy before going live
- The system tracks rule accuracy over time (based on approval/rejection rates of associated actions) and surfaces rules with high rejection rates for user review

[SME:UXDesigner] How should the rule testing interface work? A simple "paste an email and see if it triggers" input, or a more sophisticated view that replays historical emails through the rule and shows results in a timeline?

### Data Source Ingestion

All data sources are abstracted behind a **provider interface** so that each source type (email, API, RSS, etc.) implements the same contract. The rule evaluation pipeline consumes normalized content from providers — it does not know or care whether the content originated from an email, an API webhook, or a Twitter stream. Adding a new data source requires implementing the provider interface only.

#### Simulated Data Sources (Phase 1)
- All three domain presets use simulated data sources seeded with synthetic data for demo purposes
- **Simulated email inbox** (legal, social media, e-commerce) — synthetic emails covering domain-specific scenarios plus noise (newsletters, spam, irrelevant correspondence)
- **Simulated Twitter/X API** (social media) — synthetic API payloads representing mentions, trending topic matches, quote tweets, and influencer interactions, delivered as if polled from the API
- **Simulated Shopify/Amazon seller API** (e-commerce) — synthetic API payloads representing order notifications, inventory level updates, product review alerts, and marketplace policy notifications
- New synthetic data can be "received" on a configurable schedule or triggered manually to demonstrate the system in action
- Each simulated source implements the same provider interface that real connectors would, ensuring the demo architecture matches production architecture

[SME:AIWorkflow] Should the synthetic data generators use an LLM to produce realistic content, or is hand-crafted seed data with Faker-generated names/dates sufficient? LLM-generated content could cover more edge cases but adds complexity. Is the answer different for emails (varied natural language) vs. API payloads (structured JSON)?

#### Content Processing
- Incoming content from any source is normalized into a common format: source type, metadata (sender/origin, timestamp, identifiers), and body content (text, HTML, or structured JSON)
- Content is linked to existing entities based on source-specific matching (email: sender address and subject patterns; API: entity identifiers in payload; Twitter: handle matching against configured brand accounts)
- If content cannot be automatically linked to an entity, it is flagged for manual association
- Processed content is stored with its original payload, normalized metadata, entity associations, and evaluation history (which rules were run, results)

### Entity Management

#### Data Model
- The system maintains lightweight entity tables for domain-specific concepts, defined by the active domain preset
- Entity data is used to enrich rule evaluation and action generation with context (e.g., addressing a client by name in a draft email, referencing the correct case number)

**Legal Preset Entities:**
- **Clients** — name, email, phone, linked to one or more cases
- **Cases** — case number, court, case type, status, key dates, linked to one or more clients and communications

**Social Media Manager Preset Entities:**
- **Brands / Accounts** — brand name, Twitter/X handle, configured keywords, brand voice description, industry vertical
- **Clients** (agency context) — agency client name, contact info, linked to one or more brands they manage
- **Campaigns** — campaign name, date range, target keywords, associated brand, status (active/paused/completed)

**Small E-commerce Seller Preset Entities:**
- **Products** — product name, SKU, marketplace listing ID(s), current inventory level, reorder threshold, reorder quantity, supplier, product category
- **Customers** — name, email, order history summary, linked to one or more orders
- **Orders** — order ID, marketplace, order date, status (pending/shipped/delivered/issue/returned), items, shipping tracking, linked customer
- **Suppliers** — company name, contact info, products supplied, standard lead time, order terms

[SME:AIWorkflow] How much entity context should be injected into rule evaluation and action generation prompts? Including the full case history provides better context but increases token usage and cost. Should we use a summary approach or let the LLM pull only what it needs via tool calls?

#### Synthetic Entity Data (Phase 1)
- The demo ships with pre-seeded entities and communication histories for all three domain presets, creating realistic snapshots that demonstrate the system in action
- Seed data is internally consistent within each preset — communications reference correct entity identifiers, party names, and plausible timelines

**Legal preset seed data:** A mix of active cases (pending hearings, open discovery), dormant cases, and recently closed cases across multiple clients. Synthetic emails include court notifications, opposing counsel correspondence, client inquiries, and noise (newsletters, bar association updates).

**Social media preset seed data:** Two or three brand accounts with configured keywords, an active campaign per brand, and a backlog of simulated Twitter/X API payloads (mentions, trending topic matches, influencer interactions) plus agency client emails requesting content and campaign updates. Includes noise (irrelevant mentions, low-engagement posts).

**E-commerce seller preset seed data:** A product catalog with varying inventory levels (some near threshold, some healthy), recent orders in various states (fulfilled, pending, issue flagged, return requested), simulated Shopify/Amazon API payloads (order notifications, inventory alerts, review alerts, policy warnings), supplier emails, and customer emails (questions, complaints, return requests). Includes noise (marketplace marketing emails, promotional newsletters).

### Dashboard

#### Action Items View
- Primary view shows pending action items — triggered rules with their suggested actions awaiting user decision
- Each item displays: the triggering email summary, the rule that triggered, confidence score, associated entity (client/case), urgency level, and available actions
- Items are sorted by urgency and timestamp by default, with the ability to filter by rule, entity, urgency level, or status
- Completed (approved/rejected) items move to a history view

#### Email View
- Chronological view of all ingested emails with their evaluation status
- Each email shows: which rules were evaluated, which triggered, and what actions were taken
- Emails can be filtered by entity association, rule trigger status, or time range

#### Rules and Actions Management
- View all active rules with their current performance metrics (trigger rate, approval rate, average confidence)
- Create, edit, enable/disable, and delete rules and actions
- Configure rule priority ordering via drag-and-drop
- Test rules against historical emails

[SME:UXDesigner] What's the right dashboard layout for a solo practitioner who checks this tool between client calls and court appearances? Should it be a single-page priority queue, or a multi-tab layout with separate views for action items, emails, and rule management?

### Authentication and Multi-Tenancy

Authentication uses the existing Better Auth implementation (email/password, self-hosted). Multi-tenancy is new work for this project and is required to ensure complete data isolation between users.

- Each user account constitutes a **tenant** — all rules, actions, entity data (clients, cases), email inboxes, and evaluation history are scoped to the tenant
- No cross-tenant data visibility: a user cannot see, query, or reference another user's data under any circumstance
- The multi-tenancy model uses a shared database with tenant ID foreign keys on all tenant-scoped tables (not separate schemas or databases per tenant)
- Tenant isolation is enforced with **defense in depth** at two layers:
  - **Application layer** — all Drizzle ORM queries include tenant ID scoping as a standard pattern (global filter or repository-level enforcement)
  - **Database layer** — **Postgres Row-Level Security (RLS)** policies on all tenant-scoped tables ensure that even if the application layer has a bug, the database itself will not return cross-tenant rows. RLS policies filter on the tenant ID set in the session context for each request.
- Future phases may extend tenancy to support **organizations** (e.g., a law firm as a tenant with multiple user seats, shared rules, and role-based access such as paralegal can triage but not approve, attorney can approve)

[SME:IntegrationEngineer] What's the best approach for setting the tenant ID in the Postgres session context per request? Options include `SET app.current_tenant` via a connection-level variable, or passing it through Drizzle's query context. How does this interact with connection pooling? How do we ensure RLS policies are tested as part of the CI pipeline to prevent regressions?

## Success Criteria

The system will be considered successful if:
- The Rule → Action → Approval pipeline is clearly demonstrated as a domain-agnostic pattern that could apply to multiple professions
- The legal domain preset is realistic enough that a practicing attorney would recognize the scenarios and find the suggested actions useful
- Custom rule and action creation via the UI is intuitive enough that a non-technical user can create a working rule in under 5 minutes
- The HITL approval flow (including revision with feedback) feels faster than manually writing the response from scratch
- The system correctly handles noise — newsletters, spam, and irrelevant emails do not trigger false positive action items at a disruptive rate
- The pre-filter → evaluation → action selection pipeline demonstrates meaningful cost optimization compared to running every email through every full evaluation prompt
- The architecture clearly supports future extensibility: new data source connectors, new domain presets, and new action execution methods can be added without modifying core logic

## Out of Scope for Phase 1

The following features are explicitly out of scope for the initial release but may be considered for future phases:

- Real data source integration (Gmail OAuth, Outlook OAuth, Twitter/X API, shipping carrier APIs) — Phase 1 uses simulated data sources only
- Actual email sending, tweet posting, or document filing — Phase 1 presents drafts for copy/paste only
- Multi-user organization accounts with shared rules and role-based access
- Additional domain presets beyond the three shipped (legal, social media, e-commerce)
- Analytics and reporting on rule performance trends over time
- Mobile-optimized interface
- Notification system (push notifications, SMS alerts for urgent items)
- Integration with external practice management, CRM, or ERP systems
- Automated entity extraction and linking from content (Phase 1 uses pre-seeded entity data with manual association as fallback)
- Real-time streaming from Twitter/X API — Phase 1 simulates API polling with synthetic payloads

## Technical Constraints

- All LLM inference must route through **OpenRouter** to enable easy model switching without code changes
- Model selection is constrained to the **Kimi, Qwen, MiniMax, and GLM** model families — the specific models for each pipeline stage (pre-filter, rule evaluation, action generation, revision) should be determined by cost/capability tradeoffs
- Rule evaluation (pre-filter + full prompt) must complete within 5 seconds per email to avoid perceptible lag when emails arrive
- Action generation must complete within 10 seconds to keep the approval flow responsive
- The revision feedback loop should return updated drafts within 10 seconds
- The system must function with the simulated inbox without requiring any external API credentials beyond an OpenRouter API key
- The data source provider interface must be clearly defined so that adding a new connector requires implementing the interface only — no modifications to the rule evaluation pipeline or dashboard

[SME:AIWorkflow] Given the constraint to Kimi, Qwen, MiniMax, and GLM families via OpenRouter, which specific models are best suited for each pipeline stage? The pre-filter needs a fast/cheap model, rule evaluation needs strong instruction-following, and action generation needs good writing quality. Research current model availability and pricing on OpenRouter for these families.

[SME:AIWorkflow] What's the right approach for batching rule evaluations to optimize LLM API usage? Should we batch multiple rules into a single prompt per email, or keep them separate for cleaner error handling and auditability? What's the latency/cost tradeoff?

## Assumptions

- Professionals are willing to invest initial setup time configuring rules (or will use domain presets as-is)
- LLM-based evaluation is accurate enough for the pre-filter → full evaluation two-pass approach to meaningfully reduce false positives compared to a single-pass approach
- Email content provides sufficient signal to detect most actionable items without access to external context (case management systems, court calendars, etc.)
- Users will tolerate some false positives in rule triggering if the approval flow makes it easy to quickly dismiss irrelevant items
- The simulated inbox with synthetic data is sufficient for demonstrating the system's value in a portfolio context
- Domain presets with 5-10 predefined rules and 5-10 predefined actions per domain provide enough coverage to be credibly useful
