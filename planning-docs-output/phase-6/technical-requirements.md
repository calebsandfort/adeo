# Adeo Technical Specification — Phase 6

**Project:** Adeo — AI-Powered Professional Communication Assistant
**Phase:** 6 — Technical Specification
**Date:** 2026-03-13
**Status:** IMPLEMENTATION-READY

---

## 1. Project Goal

Adeo is a domain-agnostic, AI-powered system that monitors a professional's incoming communications, evaluates them against configurable rules, and surfaces actionable items with draft responses ready for human approval. The system addresses the universal problem of critical action items buried in high-volume communication channels.

For Phase 1, the system monitors simulated email data sources across three domain presets: Legal, Social Media Management, and E-commerce.

---

## 2. Functional Requirements

### FR-1.0 Rules Engine Architecture

The system **SHALL** provide a configurable rules engine that evaluates incoming communications against user-defined or preset rules to detect actionable items.

#### Technical Implementation

**Data Model (`rules`):**
```sql
CREATE TABLE rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL DEFAULT current_setting('app.current_tenant')::UUID,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    compatible_data_sources JSONB NOT NULL DEFAULT '["email"]',
    prefilter_type VARCHAR(50) NOT NULL DEFAULT 'keyword',
    prefilter_config JSONB,
    evaluation_prompt TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 100,
    short_circuit BOOLEAN NOT NULL DEFAULT false,
    action_selection_mode VARCHAR(50) NOT NULL DEFAULT 'present_all',
    custom_selection_prompt TEXT,
    is_enabled BOOLEAN NOT NULL DEFAULT true,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    domain_preset VARCHAR(50),
    confidence_thresholds JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rules_tenant ON rules(tenant_id);
CREATE INDEX idx_rules_priority ON rules(tenant_id, priority);
CREATE INDEX idx_rules_status ON rules(tenant_id, status);
```

**Drizzle Schema (`frontend/src/db/schema/rules.ts`):**
```typescript
import { pgTable, uuid, varchar, text, jsonb, boolean, integer, timestamp, index } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const rules = pgTable('rules', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().default(''),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  compatibleDataSources: jsonb('compatible_data_sources').notNull().$type<string[]>().default(['email']),
  prefilterType: varchar('prefilter_type', { length: 50 }).notNull().default('keyword'),
  prefilterConfig: jsonb('prefilter_config').$type<{
    keywords?: string[];
    patterns?: string[];
    llm_model?: string;
  }>(),
  evaluationPrompt: text('evaluation_prompt').notNull(),
  priority: integer('priority').notNull().default(100),
  shortCircuit: boolean('short_circuit').notNull().default(false),
  actionSelectionMode: varchar('action_selection_mode', { length: 50 }).notNull().default('present_all'),
  customSelectionPrompt: text('custom_selection_prompt'),
  isEnabled: boolean('is_enabled').notNull().default(true),
  status: varchar('status', { length: 20 }).notNull().default('active'),
  domainPreset: varchar('domain_preset', { length: 50 }),
  confidenceThresholds: jsonb('confidence_thresholds').$type<{
    court_deadline?: number;
    case_number?: number;
    party_name?: number;
    hearing_date?: number;
  }>(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_rules_tenant').on(table.tenantId),
  index('idx_rules_priority').on(table.tenantId, table.priority),
  index('idx_rules_status').on(table.tenantId, table.status),
]);

export const rulesRelations = relations(rules, ({ many }) => ({
  actions: many(ruleActions),
  evaluations: many(evaluations),
}));
```

**Zod Validation (`frontend/src/lib/schemas/rules.ts`):**
```typescript
import { z } from 'zod';

export const PrefilterConfigSchema = z.object({
  keywords: z.array(z.string()).optional(),
  patterns: z.array(z.string()).optional(),
  llm_model: z.string().optional(),
}).optional();

export const ConfidenceThresholdsSchema = z.object({
  court_deadline: z.number().min(0).max(1).optional(),
  case_number: z.number().min(0).max(1).optional(),
  party_name: z.number().min(0).max(1).optional(),
  hearing_date: z.number().min(0).max(1).optional(),
});

export const RuleCreateSchema = z.object({
  name: z.string().min(1).max(255),
  description: z.string().optional(),
  compatibleDataSources: z.array(z.string()).default(['email']),
  prefilterType: z.enum(['keyword', 'pattern', 'llm_classification']).default('keyword'),
  prefilterConfig: PrefilterConfigSchema,
  evaluationPrompt: z.string().min(10).max(10000),
  priority: z.number().int().min(1).max(1000).default(100),
  shortCircuit: z.boolean().default(false),
  actionSelectionMode: z.enum(['present_all', 'context_aware', 'custom_prompt']).default('present_all'),
  customSelectionPrompt: z.string().optional(),
  isEnabled: z.boolean().default(true),
  status: z.enum(['draft', 'active', 'disabled']).default('active'),
  domainPreset: z.enum(['legal', 'social_media', 'ecommerce']).optional(),
  confidenceThresholds: ConfidenceThresholdsSchema.optional(),
});

export const RuleUpdateSchema = RuleCreateSchema.partial().omit({ tenantId: true });

export const RuleResponseSchema = RuleCreateSchema.extend({
  id: z.string().uuid(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});
```

**API Endpoints:**
- `POST /api/v1/rules` — Create rule → 201 `{ data: RuleResponseSchema }`
- `GET /api/v1/rules` — List rules → 200 `{ data: RuleResponseSchema[], meta: { total, page, limit } }`
- `GET /api/v1/rules/:id` — Get rule → 200 `{ data: RuleResponseSchema }` | 404
- `PATCH /api/v1/rules/:id` — Update rule → 200 `{ data: RuleResponseSchema }` | 404
- `DELETE /api/v1/rules/:id` — Soft delete (set status=disabled) → 204 | 404
- `POST /api/v1/rules/:id/toggle` — Toggle enabled state → 200 `{ data: RuleResponseSchema }` | 404
- `POST /api/v1/rules/:id/dry-run` — Dry run mode → 200 `{ data: EvaluationResult[] }`

**Component Structure:**
- `frontend/src/components/rules/RuleList.tsx` — Paginated list with filters
- `frontend/src/components/rules/RuleEditor.tsx` — Create/edit form with tabs
- `frontend/src/components/rules/RuleTester.tsx` — Quick test textarea
- `frontend/src/components/rules/RuleTemplates.tsx` — Domain preset templates

**Security Considerations:**
- All queries MUST filter by `tenant_id` from authenticated session
- RLS policies required on `rules` table (see FR-6.0 for full RLS spec)
- Evaluation prompts should be sanitized for prompt injection (NFR-4.5)
- Pre-filter prompts require injection resistance validation

---

- **FR-1.1** The system **SHALL** support rule definitions consisting of: compatible data source types, a pre-filter, an evaluation prompt, a priority level, and associated actions.

  #### Technical Implementation
  - Schema includes: `compatibleDataSources`, `prefilterType`, `prefilterConfig`, `evaluationPrompt`, `priority`, `shortCircuit` fields (see FR-1.0)
  - **API:** `POST /api/v1/rules` accepts full rule definition
  - **Validation:** `RuleCreateSchema` enforces required fields and types

- **FR-1.2** The system **SHALL** support multiple pre-filter strategies: keyword/pattern matching and lightweight LLM-based classification (yes/no).

  #### Technical Implementation
  - **Prefilter Type Enum:** `'keyword' | 'pattern' | 'llm_classification'`
  - **Keyword/Pattern:** Implemented in `backend/src/agent/prefilter/keyword_filter.py`
  - **LLM Classification:** Implemented in `backend/src/agent/prefilter/llm_classifier.py` using Qwen2.5-0.5B model
  - **Config Store:** `prefilterConfig` JSONB stores keywords, regex patterns, or LLM model selection

- **FR-1.3** The system **SHALL** use structured output schemas for rule evaluation results, including:
  - `triggered`: boolean indicating if the rule condition is met
  - `confidence`: float (0.0 to 1.0) representing confidence score
  - `extracted_context`: object containing relevant entities (dates, parties, case references)
  - `reasoning`: string providing a brief (1-2 sentence) justification for audit purposes

  #### Technical Implementation
  **Evaluation Result Schema:**
  ```typescript
  // frontend/src/lib/schemas/evaluations.ts
  export const ExtractedContextSchema = z.object({
    dates: z.array(z.object({
      value: z.string(),
      type: z.enum(['deadline', 'hearing', 'meeting', 'deadline_implied', 'other']),
      is_ambiguous: z.boolean().optional(),
      note: z.string().optional(),
    })).optional(),
    parties: z.array(z.object({
      name: z.string(),
      role: z.string().optional(),
      is_new: z.boolean().optional(),
      requires_conflict_check: z.boolean().optional(),
    })).optional(),
    case_references: z.array(z.object({
      case_number: z.string(),
      court: z.string().optional(),
      jurisdiction: z.string().optional(),
    })).optional(),
    // Domain-specific context
    legal: z.object({
      document_type: z.string().optional(),
      deadline_source: z.string().optional(),
      opposing_counsel: z.string().optional(),
    }).optional(),
    social_media: z.object({
      sentiment: z.enum(['positive', 'negative', 'neutral', 'sarcastic']).optional(),
      follower_count: z.number().optional(),
      is_influencer: z.boolean().optional(),
      platform: z.string().optional(),
    }).optional(),
    ecommerce: z.object({
      order_id: z.string().optional(),
      issue_type: z.enum(['sizing', 'damage', 'quality', 'shipping', 'wrong_item', 'not_as_described', 'defective', 'missing_parts']).optional(),
      star_rating: z.number().min(1).max(5).optional(),
      marketplace: z.string().optional(),
    }).optional(),
  });

  export const EvaluationResultSchema = z.object({
    triggered: z.boolean(),
    confidence: z.number().min(0).max(1),
    extracted_context: ExtractedContextSchema.optional(),
    reasoning: z.string().max(500), // 1-2 sentences max per FR-1.9
  });
  ```

- **FR-1.4** The system **SHALL** evaluate rules in priority order, with configurable short-circuit behavior allowing high-priority rules to halt lower-priority evaluations.

  #### Technical Implementation
  - **Backend Logic:** `backend/src/agent/evaluator/rules_engine.py`
    ```python
    async def evaluate_rules(communication: Communication, rules: list[Rule]) -> list[EvaluationResult]:
        results = []
        for rule in sorted(rules, key=lambda r: r.priority):
            result = await evaluate_single_rule(communication, rule)
            results.append(result)
            if rule.short_circuit and result.triggered:
                logger.info(f"Rule {rule.id} triggered short-circuit")
                break
        return results
    ```
  - **Short-circuit config:** `shortCircuit` boolean in rule schema

- **FR-1.5** The system **SHALL** support three action selection modes:
  - **Present all**: Show every action associated with the triggered rule
  - **Context-aware selection**: Use LLM to recommend the most relevant subset of actions
  - **Custom selection prompt**: User-configured prompt for domain-specific selection logic

  #### Technical Implementation
  - **Enum:** `action_selection_mode: 'present_all' | 'context_aware' | 'custom_prompt'`
  - **Implementation:** `backend/src/agent/actions/selector.py`
    ```python
    async def select_actions(rule: Rule, context: EvaluationContext) -> list[Action]:
        if rule.action_selection_mode == 'present_all':
            return rule.actions
        elif rule.action_selection_mode == 'context_aware':
            return await llm_select_actions(rule.actions, context)
        else:
            return await custom_prompt_select(rule.custom_selection_prompt, context)
    ```

- **FR-1.6** Rules **SHALL** support enable/disable functionality without deletion.

  #### Technical Implementation
  - **Status field:** `status: 'draft' | 'active' | 'disabled'`
  - **Toggle API:** `POST /api/v1/rules/:id/toggle`
    ```typescript
    // Returns updated rule with toggled isEnabled
    { data: RuleResponseSchema }
    ```
  - **Soft delete:** DELETE sets status to 'disabled', not physical deletion

- **FR-1.7** The system **SHALL** run pre-filters in parallel to minimize latency.

  #### Technical Implementation
  - **Backend:** `backend/src/agent/prefilter/parallel_executor.py`
    ```python
    async def run_parallel_prefilters(communication: Communication, rules: list[Rule]) -> dict[UUID, bool]:
        tasks = [run_prefilter(communication, rule) for rule in rules]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return {rule.id: r for rule, r in zip(rules, results) if not isinstance(r, Exception)}
    ```

- **FR-1.8** The evaluation prompt **SHALL** include 2-3 few-shot examples (one positive trigger, one clear negative) to improve consistency without significant token overhead.

  #### Technical Implementation
  - **Prompt Template:** `backend/src/agent/prompts/evaluation_prompt.py`
    ```python
    def build_evaluation_prompt(rule: Rule, communication: Communication) -> str:
        examples = ""
        if rule.few_shot_examples:
            examples = "\n\n## Examples\n" + "\n\n".join([
                f"Input: {ex['input']}\nOutput: {json.dumps(ex['output'])}"
                for ex in rule.few_shot_examples[:3]
            ])
        return f"""
    {rule.system_prompt}

    ## Input Data
    {json.dumps(communication.to_dict())}

    ## Output Schema
    {json.dumps(EvaluationResultSchema)}

    {examples}

    ## Task
    Evaluate whether this communication triggers the rule: {rule.name}
    """
    ```

- **FR-1.9** The evaluation prompt **SHALL NOT** include chain-of-thought reasoning in production to minimize latency and token costs. The `reasoning` field in FR-1.3, when required for audit purposes, **SHALL** be limited to 1-2 sentences of concise justification, not full chain-of-thought.

  #### Technical Implementation
  - **Validation:** Post-processing step truncates `reasoning` field to 500 chars max
  - **System prompt:** Explicitly instructs model to provide "brief 1-2 sentence justification"
  - **Database:** `reasoning` column has CHECK constraint `char_length(reasoning) <= 500`

- **FR-1.10** The system **SHALL** validate LLM outputs against the required schema, with retry logic (max 1 retry) for parse failures, and a fallback to log the failure and surface the item for manual review.

  #### Technical Implementation
  - **Backend:** `backend/src/agent/validators/output_validator.py`
    ```python
    async def validate_and_parse(schema: type, raw_output: str, max_retries: int = 1) -> ParsedResult:
        for attempt in range(max_retries + 1):
            try:
                parsed = parse_json_into_schema(schema, raw_output)
                return ParsedResult(success=True, data=parsed)
            except ValidationError as e:
                if attempt < max_retries:
                    logger.warning(f"Parse attempt {attempt + 1} failed, retrying: {e}")
                    continue
                logger.error(f"All parse attempts failed: {e}")
                return ParsedResult(success=False, needs_manual_review=True, error=str(e))
        return ParsedResult(success=False, needs_manual_review=True)
    ```
  - **Fallback:** On failure, communication is flagged for manual review in dashboard

- **FR-1.11** The system **SHALL** apply domain-specific confidence thresholds to determine if extracted context requires manual verification, with results flagged but not blocked when thresholds are not met.

  #### Technical Implementation
  - **Confidence Check:** Post-evaluation step in `backend/src/agent/evaluator/threshold_checker.py`
  - **Flagging:** Adds `requires_manual_verification: true` to evaluation result
  - **UI:** Dashboard shows amber badge "Verify Context" on items below threshold
  - **Thresholds:** Configurable per domain in `confidence_thresholds` JSONB field

- **FR-1.12** The evaluation prompt **SHALL** follow a consistent template structure:
  - Role definition
  - Input data description with placeholders
  - Output schema definition
  - Few-shot examples (if applicable)
  - Any domain-specific constraints

  #### Technical Implementation
  - **Template:** `backend/src/agent/prompts/templates/evaluation_v1.yaml`
  - **Structure:**
    ```yaml
    template:
      role: "You are a legal document analyzer..."
      input_format: "Communication: {sender}, {subject}, {body}..."
      output_schema: <embedded JSON schema>
      constraints:
        - "Return null for ambiguous values rather than guessing"
        - "Provide 1-2 sentence reasoning only"
    ```

---

### FR-2.0 Actions Generation and Selection

The system **SHALL** generate draft responses and action items based on triggered rules, with entity context injection for personalization.

#### Technical Implementation

**Data Model (`actions`):**
```sql
CREATE TABLE actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL DEFAULT current_setting('app.current_tenant')::UUID,
    rule_id UUID NOT NULL REFERENCES rules(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    generation_prompt TEXT NOT NULL,
    execution_method VARCHAR(50) NOT NULL DEFAULT 'draft_response',
    required_sections JSONB,
    tone_guidelines TEXT,
    max_word_count INTEGER,
    domain_specific_config JSONB,
    disclaimer_required BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_actions_tenant ON actions(tenant_id);
CREATE INDEX idx_actions_rule ON actions(rule_id);
```

**Drizzle Schema (`frontend/src/db/schema/actions.ts`):**
```typescript
export const actions = pgTable('actions', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  ruleId: uuid('rule_id').notNull().references(() => rules.id),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  generationPrompt: text('generation_prompt').notNull(),
  executionMethod: varchar('execution_method', { length: 50 }).notNull().default('draft_response'),
  requiredSections: jsonb('required_sections').$type<string[]>(),
  toneGuidelines: text('tone_guidelines'),
  maxWordCount: integer('max_word_count'),
  domainSpecificConfig: jsonb('domain_specific_config').$type<{
    platform_limits?: Record<string, number>;
    sentiment_mapping?: Record<string, string>;
    brand_voice_scores?: Record<string, number>;
  }>(),
  disclaimerRequired: boolean('disclaimer_required').default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Zod Validation:**
```typescript
// frontend/src/lib/schemas/actions.ts
export const ActionCreateSchema = z.object({
  ruleId: z.string().uuid(),
  name: z.string().min(1).max(255),
  description: z.string().optional(),
  generationPrompt: z.string().min(10).max(10000),
  executionMethod: z.enum([
    'draft_response',
    'draft_email',
    'calendar_entry',
    'internal_note',
    'flag_for_review',
    'conflict_check',
  ]).default('draft_response'),
  requiredSections: z.array(z.string()).optional(),
  toneGuidelines: z.string().optional(),
  maxWordCount: z.number().int().min(10).max(10000).optional(),
  domainSpecificConfig: z.record(z.unknown()).optional(),
  disclaimerRequired: z.boolean().default(false),
});
```

**Component Structure:**
- `frontend/src/components/actions/ActionCard.tsx` — Individual action preview
- `frontend/src/components/actions/ActionEditor.tsx` — Create/edit action form
- `frontend/src/components/actions/ActionTemplates.tsx` — Domain preset templates

---

- **FR-2.1** Each action **SHALL** consist of: a human-readable description, a generation prompt, and an execution method.

  #### Technical Implementation
  - Schema fields: `name`, `description`, `generationPrompt`, `executionMethod` (see FR-2.0)
  - **API:** `POST /api/v1/actions` creates action linked to rule

- **FR-2.2** The system **SHALL** inject entity context into action generation prompts, including:
  - Entity name and type
  - Entity status
  - Key dates relevant to the entity
  - Last interaction timestamp

  #### Technical Implementation
  - **Context Injection:** `backend/src/agent/actions/context_injector.py`
    ```python
    def build_action_context(entity: Entity, extracted: ExtractedContext) -> str:
        return f"""
    Entity: {entity.name} ({entity.type})
    Status: {entity.status}
    Key Dates: {entity.key_dates}
    Last Interaction: {entity.last_interaction}
    Extracted Context: {json.dumps(extracted)}
    """
    ```
  - **Token Budget:** Context capped at 200-500 tokens per NFR-3.2

- **FR-2.3** The system **SHALL** generate actions using structured templates with LLM creative freedom within content sections, enforcing:
  - Required section headings
  - Domain-specific tone guidelines
  - Maximum word count constraints

  #### Technical Implementation
  - **Template enforcement:** Generation prompt includes strict section requirements
  - **Tone injection:** `toneGuidelines` field appended to prompt
  - **Word count:** Hard limit enforced in post-processing

- **FR-2.3.1** Action generation prompts **SHALL** follow the template:
  - Context summary (entity data, extracted info)
  - Tone/style guidelines from domain preset or brand voice
  - Required sections with format guidance
  - Constraints (word count, platform limits)

  #### Technical Implementation
  - **Prompt builder:** `backend/src/agent/prompts/action_prompt_builder.py`

- **FR-2.4** The system **SHALL** support three draft generation modes for social media domain:
  - **Fast Draft** (3-5 seconds generation): Minimal formatting, single-tap approval
  - **Standard Draft** (8-12 seconds): Platform conventions, one auto-revision cycle
  - **Enhanced Draft** (15-20 seconds): Multiple options, full revision support

  #### Technical Implementation
  - **Draft Mode Enum:** `'fast' | 'standard' | 'enhanced'`
  - **Implementation:** `backend/src/agent/actions/draft_generator.py`
    ```python
    async def generate_draft(action: Action, context: ActionContext, mode: DraftMode) -> DraftResult:
        if mode == 'fast':
            return await quick_generate(action, context)
        elif mode == 'standard':
            draft = await quick_generate(action, context)
            return await refine_once(draft, context)
        else:
            options = await generate_multiple(action, context, n=3)
            return DraftResult(mode='enhanced', options=options)
    ```

- **FR-2.5** The system **SHALL** auto-select draft mode based on:
  - Trend velocity (spiking = Fast Draft)
  - Time since post (older = Standard/Enhanced)
  - Follower count (high-follower = Fast Draft priority)
  - Sentiment (negative = Fast Draft with escalation)

  #### Technical Implementation
  - **Auto-selector:** `backend/src/agent/actions/draft_mode_selector.py`
    ```python
    def select_draft_mode(context: ActionContext) -> DraftMode:
        if context.trend_velocity > 5 or context.follower_count > 10000:
            return 'fast'
        elif context.sentiment == 'negative':
            return 'fast'  # With escalation flag
        elif context.age_hours > 24:
            return 'enhanced'
        return 'standard'
    ```

- **FR-2.6** The system **SHALL** generate actions in parallel with timeout handling, where failed generations are surfaced with a "tap to retry" option rather than blocking other actions.

  #### Technical Implementation
  - **Parallel generation:** `asyncio.gather` with individual timeouts
  - **Timeout config:** 15 seconds default (FR-2.6.1)
  - **Error handling:** Individual action failures don't block others
  - **Retry UI:** "Tap to retry" button on failed action cards

- **FR-2.6.1** Action generation timeouts **SHALL** default to 15 seconds, with failed generations surfaced as "tap to retry" within 20 seconds of initiation.

  #### Technical Implementation
  - **Config:** `action_generation_timeout_seconds: 15` in settings
  - **Timeout handler:** Catches `TimeoutError`, marks action as `needs_retry`
  - **UI:** Retry button calls same generation endpoint

- **FR-2.7** For legal domain actions, the system **SHALL** append a configurable disclaimer ("This draft was generated with AI assistance and requires attorney review before use") as a post-generation step, not within the prompt. The system **SHALL** provide guidance on when the disclaimer is ethically required versus optional based on jurisdiction, and **SHALL** include a checkbox in the approval workflow: "I have reviewed this draft for accuracy."

  #### Technical Implementation
  - **Disclaimer append:** Post-generation step in `backend/src/agent/actions/legal_disclaimer.py`
  - **Config:** `disclaimer_text` stored per tenant in `tenant_settings`
  - **Ethics guidance:** Tooltip shows jurisdiction-specific requirements
  - **Checkbox:** Required field in approval form
  - **Database:** `approval_checkbox_confirmed` boolean in `generated_actions` table

- **FR-2.8** The system **SHALL** integrate sentiment detection into the generation prompt (not as a separate call) for social media content generation.

  #### Technical Implementation
  - **Integrated sentiment:** Part of `extracted_context` from rule evaluation (FR-1.3)
  - **Prompt injection:** Sentiment mapped to tone guidelines
    ```python
    SENTIMENT_TONE_MAP = {
        'positive': 'Match or exceed the enthusiasm in the original message',
        'negative': 'Empathize first, then offer a solution',
        'neutral': 'Helpful and concise',
        'sarcastic': 'Acknowledge concern without being defensive',
    }
    ```

---

### FR-3.0 Human-in-the-Loop (HITL) Approval Workflow

The system **SHALL** require explicit human approval before executing any generated action.

#### Technical Implementation

**Data Model (`generated_actions`):**
```sql
CREATE TABLE generated_actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL DEFAULT current_setting('app.current_tenant')::UUID,
    communication_id UUID NOT NULL REFERENCES communications(id),
    rule_id UUID NOT NULL REFERENCES rules(id),
    action_id UUID NOT NULL REFERENCES actions(id),
    generated_content TEXT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    revision_count INTEGER NOT NULL DEFAULT 0,
    last_feedback TEXT,
    approval_checkbox_confirmed BOOLEAN DEFAULT false,
    rejection_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    approved_at TIMESTAMPTZ,
    executed_at TIMESTAMPTZ,
    undone_at TIMESTAMPTZ
);

CREATE INDEX idx_generated_actions_tenant ON generated_actions(tenant_id);
CREATE INDEX idx_generated_actions_status ON generated_actions(tenant_id, status);
CREATE INDEX idx_generated_actions_communication ON generated_actions(communication_id);
```

**Drizzle Schema:**
```typescript
// frontend/src/db/schema/approval.ts
export const generatedActions = pgTable('generated_actions', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  communicationId: uuid('communication_id').notNull().references(() => communications.id),
  ruleId: uuid('rule_id').notNull().references(() => rules.id),
  actionId: uuid('action_id').notNull().references(() => actions.id),
  generatedContent: text('generated_content').notNull(),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
  revisionCount: integer('revision_count').notNull().default(0),
  lastFeedback: text('last_feedback'),
  approvalCheckboxConfirmed: boolean('approval_checkbox_confirmed').default(false),
  rejectionReason: text('rejection_reason'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  approvedAt: timestamp('approved_at'),
  executedAt: timestamp('executed_at'),
  undoneAt: timestamp('undone_at'),
});
```

**Zod Schema:**
```typescript
export const ApprovalDecisionSchema = z.object({
  actionId: z.string().uuid(),
  decision: z.enum(['approve', 'edit', 'revise', 'reject']),
  editedContent: z.string().optional(),
  feedback: z.string().optional(),
  rejectionReason: z.string().optional(),
  approvalCheckboxConfirmed: z.boolean().optional(),
});

export const RevisionFeedbackSchema = z.object({
  actionId: z.string().uuid(),
  feedback: z.string().min(1).max(1000),
});
```

---

- **FR-3.1** The system **SHALL** present four approval options:
  - **Approve**: Accept the generated draft as-is and execute
  - **Edit**: Modify the draft in-place before executing
  - **Revise with feedback**: Provide natural language feedback and receive a revised draft
  - **Reject**: Dismiss the action entirely, optionally with a reason for tuning

  #### Technical Implementation
  - **API:** `POST /api/v1/approvals/decide`
    ```typescript
    // Request
    { actionId: string, decision: 'approve' | 'edit' | 'revise' | 'reject', ... }
    // Response: 200 { data: { status: string, ... } }
    ```
  - **Status transitions:** `pending` -> `approved` | `edit_mode` | `revising` | `rejected`
  - **Edit mode:** Returns editable content, user submits `editedContent`

- **FR-3.2** The system **SHALL** log all approval and rejection decisions with timestamps and user identity for audit purposes.

  #### Technical Implementation
  - **Audit log:** `approval_audit_log` table
    ```sql
    CREATE TABLE approval_audit_log (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        tenant_id UUID NOT NULL,
        user_id UUID NOT NULL REFERENCES users(id),
        action_id UUID NOT NULL REFERENCES generated_actions(id),
        decision VARCHAR(20) NOT NULL,
        timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        metadata JSONB
    );
    ```
  - **Logging:** Every approval endpoint writes to audit log

- **FR-3.3** The system **SHALL** support multiple revision rounds, with a cap at 5 rounds to prevent draft degradation. After 5 rounds, the system **SHALL** prompt the user to make direct edits.

  #### Technical Implementation
  - **Counter:** `revisionCount` in `generated_actions` table
  - **Check:** If `revisionCount >= 5`, returns `{ requires_direct_edit: true }`
  - **UI:** Shows "Maximum revisions reached" message with Edit button

- **FR-3.4** The revision workflow **SHALL** use a multi-turn conversation approach that includes: the original prompt, triggering content, the original draft, and all feedback rounds.

  #### Technical Implementation
  - **Conversation history:** Stored in `revision_history` JSONB field
    ```typescript
    {
      original_trigger: "...",
      original_draft: "...",
      feedback_rounds: [
        { feedback: "...", revised_draft: "...", timestamp: "..." },
        // ...
      ]
    }
    ```
  - **Prompt construction:** `backend/src/agent/actions/revision_builder.py`

- **FR-3.5** The system **SHALL** prevent revision degradation by always including the original triggering content in context and flagging revisions that significantly lose key information.

  #### Technical Implementation
  - **Degradation check:** Post-revision validation compares key entities
  - **Flag:** If key entities missing, shows warning "Original context may be lost"
  - **Inclusion:** Original always prepended to revision prompt

- **FR-3.6** The system **SHALL** provide a 30-second undo window after approval or rejection, allowing users to reverse their decision before execution completes.

  #### Technical Implementation
  - **Undo window:** 30 seconds from `approvedAt` timestamp
  - **API:** `POST /api/v1/approvals/:id/undo`
    ```typescript
    // Request: {}
    // Response: 200 { data: { status: 'pending', can_undo: false } }
    // Error: 410 Gone if window expired
    ```
  - **Execution:** Delayed by 30 seconds; undo cancels scheduled execution job

---

### FR-4.0 Dashboard and UI Components

The system **SHALL** provide a dashboard that enables rapid scanning and decision-making for professionals checking the tool between tasks.

#### Technical Implementation

**Component Structure:**
```
frontend/src/components/dashboard/
├── DashboardPage.tsx           # Main dashboard layout
├── ActionCard.tsx              # Individual action item card
├── ActionCardSkeleton.tsx      # Loading skeleton
├── ActionDrawer.tsx            # Review actions sheet (side drawer)
├── EmptyState.tsx              # No pending items, no rules, no entities
├── ErrorState.tsx              # LLM failure, timeout, network error
├── FilterBar.tsx               # Urgency, rule, entity, status filters
├── UrgencyBadge.tsx            # Amber/blue pill badges
├── DeadlineCountdown.tsx       # Legal domain deadline display
├── ConflictIndicator.tsx      # Conflicting actions display
├── ToastNotifications.tsx     # Success confirmation toasts
└── TestingMode.tsx             # Quick test & historical replay
```

**Zod Schemas for Dashboard:**
```typescript
// frontend/src/lib/schemas/dashboard.ts
export const DashboardFiltersSchema = z.object({
  urgency: z.enum(['critical', 'high', 'medium', 'low', 'all']).optional(),
  ruleId: z.string().uuid().optional(),
  entityId: z.string().uuid().optional(),
  status: z.enum(['pending', 'approved', 'rejected', 'all']).optional(),
});

export const ActionItemCardSchema = z.object({
  id: z.string().uuid(),
  urgency: z.enum(['critical', 'high', 'medium', 'low']),
  timestamp: z.string().datetime(),
  emailSubject: z.string().max(60),
  ruleName: z.string(),
  entityName: z.string().optional(),
  entityType: z.string().optional(),
  triggeredRulesCount: z.number().optional(),
  status: z.enum(['pending', 'approved', 'rejected']),
});

export const ActionDrawerSchema = z.object({
  actionId: z.string().uuid(),
  extractedContext: ExtractedContextSchema,
  originalEmail: z.string().optional(),
  generatedActions: z.array(GeneratedActionSchema),
  conflicts: z.array(ConflictSchema).optional(),
});
```

---

- **FR-4.1** The dashboard **SHALL** group action items by urgency first, then by email thread (same email showing all triggered rules/actions together).

  #### Technical Implementation
  - **Backend:** `GET /api/v1/dashboard/items` returns grouped by urgency then communication
  - **SQL:**
    ```sql
    SELECT * FROM generated_actions
    WHERE tenant_id = $1 AND status = 'pending'
    ORDER BY
      CASE urgency
        WHEN 'critical' THEN 1
        WHEN 'high' THEN 2
        WHEN 'medium' THEN 3
        WHEN 'low' THEN 4
      END,
      communication_id,
      created_at DESC
    ```

- **FR-4.2** The system **SHALL** use the amber/blue accent system:
  - Amber (urgent): Critical and high-urgency items
  - Blue (routine): Medium and low-urgency items
  - Extended for legal domain: Critical (amber + pulsing dot), High (amber), Medium (blue), Low (gray)

  #### Technical Implementation
  - **Colors (CSS variables):**
    ```css
    .urgency-critical { --badge-bg: oklch(0.6 0.18 55); /* amber */ }
    .urgency-high { --badge-bg: oklch(0.6 0.15 55); }
    .urgency-medium { --badge-bg: oklch(0.6 0.15 235); /* blue */ }
    .urgency-low { --badge-bg: oklch(0.5 0.1 250); /* gray */ }
    .urgency-critical.pulsing::after {
      content: '';
      animation: pulse-dot 2s infinite;
    }
    ```
  - **Component:** `frontend/src/components/dashboard/UrgencyBadge.tsx`

- **FR-4.3** Each action item card **SHALL** display:
  - Urgency badge (amber/blue pill)
  - Timestamp (relative, e.g., "5 min ago" in monospace)
  - Email subject line (truncated to 60 characters)
  - Rule name that triggered
  - Associated entity (client name, case number, brand name)
  - Single primary CTA: "Review Actions" button

  #### Technical Implementation
  - **Component:** `frontend/src/components/dashboard/ActionCard.tsx`
    ```tsx
    <Card>
      <UrgencyBadge urgency={item.urgency} />
      <span className="font-mono text-xs">{formatRelativeTime(item.timestamp)}</span>
      <h3 className="truncate max-w-[60ch]">{item.emailSubject}</h3>
      <span className="text-sm text-muted">{item.ruleName}</span>
      {item.entityName && <span className="text-sm">{item.entityName}</span>}
      <Button>Review Actions</Button>
    </Card>
    ```

- **FR-4.4** The dashboard **SHALL** show a single card when multiple rules trigger on the same email, with a badge showing "X rules triggered" and the highest-urgency indicator.

  #### Technical Implementation
  - **Grouping logic:** Server groups by `communication_id`
  - **Badge:** `{ triggeredRulesCount } rules triggered` pill
  - **Urgency:** Max urgency from grouped items

- **FR-4.5** The "Review Actions" sheet **SHALL** be implemented as a side drawer (Sheet component) that:
  - Shows extracted context in a compact summary card (always visible)
  - Provides a collapsible "Original Email" panel for verification
  - Displays all generated actions with Approve/Edit/Revise/Reject buttons
  - Preserves dashboard context behind the drawer

  #### Technical Implementation
  - **Component:** `frontend/src/components/dashboard/ActionDrawer.tsx`
  - **ShadCN:** Uses `Sheet` component with `side="right"`
  - **Layout:** Fixed 500px width, scrollable content

- **FR-4.6** The system **SHALL** detect conflicting actions from different triggered rules and present them side-by-side in the review sheet with explicit conflict indication. For legal domain, the system **SHALL** explicitly address conflicts where actions to different recipients would create inconsistency.

  #### Technical Implementation
  - **Conflict detection:** `backend/src/agent/conflicts/detector.py`
    ```python
    def detect_conflicts(actions: list[GeneratedAction]) -> list[Conflict]:
        # Check for:
        # - Different recipients with contradictory content
        # - Multiple calendar entries for same time
        # - Opposing tone recommendations
    ```
  - **UI:** `frontend/src/components/dashboard/ConflictIndicator.tsx`

- **FR-4.7** For legal domain deadline display, the system **SHALL** show countdown prominently ("3 days remaining") with the specific date available on hover. The system **SHALL** also indicate if the deadline falls on a weekend or court holiday.

  #### Technical Implementation
  - **Component:** `frontend/src/components/dashboard/DeadlineCountdown.tsx`
    ```tsx
    const daysRemaining = differenceInBusinessDays(deadline, now);
    const isWeekend = isWeekend(deadline);
    const isHoliday = await checkCourtHolidays(deadline, jurisdiction);
    // Display: "3 days remaining" (with holiday badge if applicable)
    // Hover: full date tooltip
    ```

- **FR-4.8** For review response previews, the system **SHALL** show the first 100 characters with character count, indicating platform-specific limits. Character limits **SHALL** be configurable per platform (e.g., Google: 400 chars, Yelp: 5000 chars, Trustpilot: 1000 chars, G2: 500 chars, Twitter/X: 280 chars).

  #### Technical Implementation
  - **Platform limits:** Stored in `platform_limits` config
    ```typescript
    const PLATFORM_LIMITS = {
      'google': 400,
      'yelp': 5000,
      'trustpilot': 1000,
      'g2': 500,
      'twitter': 280,
    };
    ```
  - **UI:** Character counter with warning at 90% capacity

- **FR-4.9** The dashboard **SHALL** support filtering by: urgency level, rule, entity, and status (pending/approved/rejected).

  #### Technical Implementation
  - **API:** `GET /api/v1/dashboard/items?urgency=high&ruleId=...&entityId=...&status=pending`
  - **Component:** `frontend/src/components/dashboard/FilterBar.tsx`

- **FR-4.10** The system **SHALL** support two testing modes:
  - **Quick Test**: Textarea input to paste/type email content and see immediate trigger results
  - **Historical Replay**: Filter all emails by triggered/not triggered status, with expandable details per email

  #### Technical Implementation
  - **Component:** `frontend/src/components/testing/TestingMode.tsx`
  - **Quick Test:** `POST /api/v1/rules/dry-run` with text input
  - **Historical:** `GET /api/v1/communications?triggered=true` with pagination

- **FR-4.11** The system **SHALL** display contextual empty states for: (1) no pending items, (2) no rules configured, (3) no entities in database. Each empty state **SHALL** include appropriate messaging and a clear call-to-action.

  #### Technical Implementation
  - **Component:** `frontend/src/components/dashboard/EmptyState.tsx`
    ```typescript
    type EmptyStateType = 'no_items' | 'no_rules' | 'no_entities';
    // Different messaging and CTA per type
    ```

- **FR-4.12** The system **SHALL** display error states for: (1) LLM evaluation failure, (2) action generation timeout, (3) network error. Error cards **SHALL** include retry CTAs with clear placement.

  #### Technical Implementation
  - **Component:** `frontend/src/components/dashboard/ErrorState.tsx`
  - **Retry:** Each error card has individual retry button
  - **Error types:** `evaluation_failed`, `generation_timeout`, `network_error`

- **FR-4.13** The system **SHALL** display toast notifications confirming successful action execution after approval, with specified duration and dismissal behavior.

  #### Technical Implementation
  - **Component:** `frontend/src/components/dashboard/ToastNotifications.tsx`
  - **Duration:** 5 seconds auto-dismiss (configurable)
  - **Dismiss:** Click X to dismiss early

---

### FR-5.0 Custom Rule and Action Creation

The system **SHALL** enable users to create custom rules and actions through the UI without requiring technical expertise.

#### Technical Implementation

**LLM-Assisted Creation Endpoint:**
```typescript
// POST /api/v1/rules/llm-assist
// Request: { goal: string, domain: string }
// Response: { suggestedPrompt: string, suggestedKeywords: string[], suggestedPriority: number }

// POST /api/v1/rules/example-based
// Request: { positiveExamples: string[], negativeExamples: string[] }
// Response: { generatedPrompt: string }
```

**Component Structure:**
- `frontend/src/components/rules/RuleWizard.tsx` — Multi-step creation flow
- `frontend/src/components/rules/LLMAssistedEditor.tsx` — Natural language input
- `frontend/src/components/rules/ExampleBasedEditor.tsx` — Example input UI
- `frontend/src/components/rules/PromptValidator.tsx` — Static analysis warnings

---

- **FR-5.1** The system **SHALL** support three rule creation paths:
  - **Manual prompt writing**: User writes the evaluation prompt directly
  - **LLM-assisted creation**: User describes the detection goal in natural language; system generates the evaluation prompt, pre-filter keywords, and suggested priority
  - **Example-based creation**: User provides 3-5 positive examples and 2-3 negative examples; system infers the pattern and generates the prompt

  #### Technical Implementation
  - **Manual:** Direct edit form with `RuleCreateSchema` validation
  - **LLM-assisted:** `POST /api/v1/rules/llm-assist` endpoint
  - **Example-based:** `POST /api/v1/rules/example-based` endpoint

- **FR-5.2** The system **SHALL** support the same three creation paths for custom actions.

  #### Technical Implementation
  - Same endpoints: `/api/v1/actions/llm-assist`, `/api/v1/actions/example-based`

- **FR-5.3** The system **SHALL** validate generated prompts using static analysis to detect:
  - Missing examples in prompt (warn: likely inconsistent)
  - Ambiguous language overuse ("may", "might", "possibly")
  - Missing output schema (warn: unparseable)
  - Conflicting instructions
  - Excessive length (>2000 tokens: warn may timeout)

  #### Technical Implementation
  - **Validator:** `backend/src/agent/validators/prompt_validator.py`
    ```python
    def validate_prompt(prompt: str) -> ValidationResult:
        warnings = []
        if count_examples(prompt) == 0:
            warnings.append("No examples found - results may be inconsistent")
        if count_ambiguous(prompt) > 3:
            warnings.append("High use of ambiguous language")
        if not has_output_schema(prompt):
            warnings.append("Missing output schema")
        if token_count(prompt) > 2000:
            warnings.append("Prompt exceeds 2000 tokens - may timeout")
        return ValidationResult(warnings=warnings)
    ```
  - **UI:** Warnings displayed inline during creation

- **FR-5.4** The system **SHALL** provide a "dry run" mode where newly created rules evaluate incoming emails and log results without surfacing actions to the dashboard.

  #### Technical Implementation
  - **API:** `POST /api/v1/rules/:id/dry-run`
  - **Query param:** `?dryRun=true` on evaluation endpoints
  - **Response:** `{ results: EvaluationResult[], dryRun: true }`
  - **Storage:** Results in `evaluation_logs` with `is_dry_run: true`

- **FR-5.5** The system **SHALL** support rule lifecycle states: **Draft** (not yet active), **Active** (evaluating and surfacing actions), and **Disabled** (not evaluating). Users **SHALL** be able to transition rules between these states through the UI.

  #### Technical Implementation
  - **Status field:** `status: 'draft' | 'active' | 'disabled'`
  - **Transitions:** Via `PATCH /api/v1/rules/:id` or toggle UI
  - **Active rules only:** Evaluation pipeline queries `WHERE status = 'active'`

---

### FR-6.0 Multi-Tenancy and Data Isolation

The system **SHALL** ensure complete data isolation between tenants using defense-in-depth architecture.

#### Technical Implementation

**Data Model (`tenant_settings`):**
```sql
CREATE TABLE tenant_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL UNIQUE,
    domain_preset VARCHAR(50),
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Enable RLS on all tables
ALTER TABLE rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE actions ENABLE ROW LEVEL SECURITY;
ALTER TABLE communications ENABLE ROW LEVEL SECURITY;
ALTER TABLE generated_actions ENABLE ROW LEVEL SECURITY;
ALTER TABLE entities ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_settings ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY rules_tenant_isolation ON rules
    USING (tenant_id::text = current_setting('app.current_tenant'));

CREATE POLICY actions_tenant_isolation ON actions
    USING (tenant_id::text = current_setting('app.current_tenant'));

-- ... similar for all tables
```

**Drizzle Schema:**
```typescript
// frontend/src/db/schema/tenant.ts
export const tenantSettings = pgTable('tenant_settings', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().unique(),
  domainPreset: varchar('domain_preset', { length: 50 }),
  settings: jsonb('settings').notNull().$type<Record<string, unknown>>().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Tenant Context Setup:**
```typescript
// frontend/src/lib/db/tenant.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { sql } from 'drizzle-orm';

export async function setTenantContext(tenantId: string) {
  const db = getDb();
  await db.execute(sql`SET app.current_tenant = ${tenantId}`);
}

export async function resetTenantConnection() {
  const db = getDb();
  // Reset all session variables to prevent tenant contamination
  await db.execute(sql`RESET app.current_tenant`);
}
```

---

- **FR-6.1** Each user account **SHALL** constitute a tenant, with all rules, actions, entity data, and evaluation history scoped to the tenant. (Note: The architecture SHOULD support organization-level tenancy in future phases by adding an `organization_id` foreign key with appropriate RLS policies.)

  #### Technical Implementation
  - **Tenant ID:** Uses `user.id` from Better Auth as `tenant_id`
  - **Future proof:** Schema includes comment about `organization_id` addition

- **FR-6.2** The system **SHALL** enforce tenant isolation at two layers:
  - **Application layer**: Drizzle ORM queries include tenant ID scoping as a standard pattern
  - **Database layer**: Postgres Row-Level Security (RLS) policies on all tenant-scoped tables

  #### Technical Implementation
  - **App layer:** All queries include `.where(eq(table.tenantId, tenantId))`
  - **DB layer:** RLS policies use `current_setting('app.current_tenant')`

- **FR-6.3** The system **SHALL** set tenant context via `SET app.current_tenant` at the database session level, not through query context filtering.

  #### Technical Implementation
  - **Connection:** Per-request tenant context setup
  - **Middleware:** `frontend/src/lib/db/middleware.ts`

- **FR-6.4** RLS policies **SHALL** reference `current_setting('app.current_tenant')` to ensure database-level isolation even if application-layer bugs exist.

  #### Technical Implementation
  - See FR-6.0 SQL policies

- **FR-6.5** The system **SHALL** implement connection reset logic to prevent connection contamination between tenants. If a connection pooler (PgBouncer or equivalent) is used, it **SHALL** be configured with `server_reset_query = RESET app.current_tenant`. If no pooler is used, the application layer **SHALL** implement equivalent connection reset logic.

  #### Technical Implementation
  - **App-layer reset:** `resetTenantConnection()` called on connection checkout
  - **No PgBouncer currently:** Uses connection pooler with reset query

- **FR-6.6** The system **SHALL** create indexes on `tenant_id` columns for all tenant-scoped tables to ensure RLS performance.

  #### Technical Implementation
  - All tables have `idx_<table>_tenant` index (see schema definitions)

- **FR-6.7** The system **SHALL** include RLS isolation tests in the CI pipeline that verify:
  - Tenant A cannot read Tenant B's data
  - Tenant A can read their own data
  - Direct SQL injection cannot bypass RLS

  #### Technical Implementation
  - **Test file:** `frontend/__tests__/rls-isolation.test.ts`
    ```typescript
    test('RLS: Tenant isolation', async () => {
      await asTenant('tenant-a');
      const result = await db.select().from(rules).execute();
      expect(result.every(r => r.tenantId === 'tenant-a')).toBe(true);
    });
    ```

- **FR-6.8** The system **SHALL** configure connection pool sizing appropriate for multi-tenant workloads, with per-tenant connection limits to prevent any single tenant from monopolizing database resources.

  #### Technical Implementation
  - **Pool config:** In `drizzle.config.ts`
    ```typescript
    const poolConfig = {
      max: 20, // total connections
      idleTimeoutSeconds: 30,
      connectionTimeoutSeconds: 10,
    };
    ```

- **FR-6.9** The system **SHALL** implement a context injection pattern where the frontend fetches tenant-scoped data (rules, actions, entity summaries) via Drizzle and injects it into the CopilotKit conversation context for the LangGraph pipeline to use, without the backend needing direct database access or tenant context.

  #### Technical Implementation
  - **Frontend:** Fetch rules/entities, pass as CopilotKit state
  - **Backend:** Receives context via CopilotKit protocol
  - **Pattern:** `frontend/src/hooks/useCopilotContext.ts`

- **FR-6.10** The system **SHALL** provide REST API endpoints (via Next.js API routes) for:
  - CRUD operations on rules and actions
  - Evaluation history and audit logs
  - Tenant configuration and preferences
  - Data source registration and status
  All endpoints **SHALL** enforce tenant isolation by querying with `tenant_id` from the authenticated session.

  #### Technical Implementation
  - **API Route Pattern:**
    ```typescript
    // frontend/src/app/api/v1/rules/route.ts
    export async function GET() {
      const session = await auth();
      const tenantId = session.user.id;
      const rules = await db.select().from(rules).where(eq(rules.tenantId, tenantId));
      return Response.json({ data: rules });
    }
    ```

---

### FR-7.0 Legal Domain Preset

The system **SHALL** ship with a legal domain preset targeting solo practitioners and small law firms.

#### Technical Implementation

**Domain Preset Data Model:**
```sql
CREATE TABLE domain_presets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    domain_type VARCHAR(50) NOT NULL,
    rules JSONB NOT NULL,
    actions JSONB NOT NULL,
    tone_guidelines JSONB,
    confidence_thresholds JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Preset Seed Data:**
```typescript
// frontend/src/db/seeds/legal-preset.ts
export const legalPresetRules = [
  {
    name: 'Cancelled/Rescheduled Hearing',
    prefilterKeywords: ['hearing', 'cancelled', 'rescheduled', 'motion to continue'],
    evaluationPrompt: 'Detect if a hearing has been cancelled or rescheduled...',
    urgency: 'critical',
    actions: ['draft_reschedule_request', 'calendar_entry'],
  },
  {
    name: 'Court Deadline Notification',
    prefilterKeywords: ['deadline', 'due', 'filing', 'response due'],
    evaluationPrompt: 'Extract explicit or implied deadlines from legal communications...',
    urgency: 'critical',
    actions: ['calendar_entry', 'internal_note'],
  },
  // ... 7 rules total per FR-7.1
];
```

**Component Structure:**
- `frontend/src/components/domain/legal/LegalDashboard.tsx`
- `frontend/src/components/domain/legal/DeadlineAlert.tsx`
- `frontend/src/components/domain/legal/ConflictCheck.tsx`

---

- **FR-7.1** The legal preset **SHALL** include pre-configured rules for:
  - Cancelled/rescheduled hearing detection
  - Deadline notification (explicit and implied)
  - Client inquiry requiring response
  - Opposing counsel action items
  - Court order/filing notification
  - Retainer/low-balance inquiries (flagged as high-urgency when matter is active)
  - Discovery request detection with response deadline extraction

  #### Technical Implementation
  - 7 rules in seed data (see `legalPresetRules`)

- **FR-7.1.1** The legal preset **SHALL** detect new parties or opposing counsel mentions and flag for conflict check.

  #### Technical Implementation
  - **Extracted context:** `requires_conflict_check: true` on new parties
  - **Action:** `conflict_check` action type

- **FR-7.2** The legal preset **SHALL** include urgency classification embedded within each rule's evaluation, with four levels:
  - **Critical**: Court-ordered deadlines, statute of limitations, emergency hearings, any court order containing a deadline or requiring action (regardless of label)
  - **High**: Scheduling changes, opposing counsel requests, court orders without deadlines, retainer requests
  - **Medium**: Routine client status updates
  - **Low**: Newsletters, bar updates, CC'd documents

  #### Technical Implementation
  - **Urgency per rule:** In rule's `confidenceThresholds` or evaluation logic

- **FR-7.3** The legal preset **SHALL** return confidence scores for extracted context, with thresholds:
  - Court deadlines: 85% minimum confidence; below = "verify manually"
  - Case numbers: 90% minimum confidence
  - Party names: 80% minimum confidence
  - Hearing dates: 90% minimum confidence
  - Deadline source (court, judge, document): 85% minimum confidence

  #### Technical Implementation
  - **Thresholds:** In `confidence_thresholds` JSONB (see FR-1.11)

- **FR-7.4** The system **SHALL** implement the "return unknown rather than guess" principle: ambiguous dates ("sometime next week") trigger the rule but return null for the date with a note that the date is ambiguous. This principle **SHALL** extend to all extracted entities, not just dates.

  #### Technical Implementation
  - **Prompt instruction:** Explicit in evaluation prompt template
  - **Schema:** `is_ambiguous: true` flag on uncertain extractions

- **FR-7.5** The legal preset **SHALL** include pre-configured actions for:
  - Draft client status update (conversational tone)
  - Draft rescheduling request (formal court format)
  - Draft response to opposing counsel (professional, minimal)
  - Draft internal note (factual, complete)
  - Flag for urgent review (notification only, no draft)
  - Draft conflict check reminder
  - Calendar entry or task with deadline

  #### Technical Implementation
  - 7 action templates in seed data

- **FR-7.5.6** The legal preset **SHALL** generate calendar entries or task items with deadlines as an action option, not just draft communications.

  #### Technical Implementation
  - **Execution method:** `execution_method: 'calendar_entry'`

- **FR-7.6** The system **SHALL** support jurisdiction-specific court document templates, selectable by user rather than hardcoded.

  #### Technical Implementation
  - **Config:** `jurisdiction_templates` in tenant settings
  - **UI:** Dropdown to select jurisdiction in rule editor

- **FR-7.7** The legal preset **SHALL** specify tone guidelines for each action type:
  - **Client updates**: Clear, accessible language, honest about risks, never promise outcomes
  - **Court filings**: Formal, properly cited, procedural, uses proper legal terminology
  - **Opposing counsel**: Professional but not cordial, avoids admissions, minimal unnecessary communication
  - **Internal notes**: Factual, chronological, includes document references, avoids speculation

  #### Technical Implementation
  - **Tone guidelines:** Per action type in `tone_guidelines` JSONB

- **FR-7.8** The legal preset **SHALL** handle deadline edge cases:
  - Court holiday detection and adjustment
  - Weekend deadline adjustment
  - Business day calculations
  - Service-date-based deadline computation
  - Cross-jurisdiction variation awareness

  #### Technical Implementation
  - **Helper:** `backend/src/agent/utils/legal_deadlines.py`
    ```python
    def adjust_for_holidays_and_weekends(deadline: date, jurisdiction: str) -> date:
        # Federal court holidays by default, configurable per jurisdiction
        while is_weekend(deadline) or is_court_holiday(deadline, jurisdiction):
            deadline = deadline - timedelta(days=1)
        return deadline

    def calculate_response_deadline(service_date: date, jurisdiction: str) -> date:
        # Service-by-mail, etc.
        return add_business_days(service_date, response_days[jurisdiction])
    ```

- **FR-7.9** The legal preset **SHOULD** support practice-area-specific sub-configurations (family law, criminal defense, personal injury, estate planning, corporate) in future phases.

  #### Technical Implementation
  - **Future:** Placeholder in architecture, not implemented in Phase 1

---

### FR-8.0 Social Media Domain Preset

The system **SHALL** ship with a social media management domain preset for solo managers and small agencies.

#### Technical Implementation

**Data Model (Brand Voice Profiles):**
```sql
CREATE TABLE brand_voice_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    scores JSONB NOT NULL, -- { formality: 7, humor: 5, personality: 8, expertise: 9, empathy: 7 }
    platform_overrides JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Preset Seed Data:**
```typescript
// frontend/src/db/seeds/social-media-preset.ts
export const socialMediaPresetRules = [
  {
    name: 'Trending Topic Mention',
    prefilterType: 'keyword',
    prefilterConfig: { keywords: ['trending', 'viral', '#trending'] },
    urgency: 'high',
  },
  {
    name: 'Brand Mention Spike',
    prefilterType: 'llm_classification',
    urgency: 'high',
  },
  // ... 8 rules total per FR-8.1
];
```

**Component Structure:**
- `frontend/src/components/domain/social/SocialDashboard.tsx`
- `frontend/src/components/domain/social/BrandVoiceEditor.tsx`
- `frontend/src/components/domain/social/CrisisAlert.tsx`
- `frontend/src/components/domain/social/PlatformPreview.tsx`

---

- **FR-8.1** The social media preset **SHALL** include pre-configured rules for:
  - Trending topic keyword match
  - Brand mention spike detection
  - Negative sentiment mention
  - Influencer/high-follower engagement
  - Review site alert
  - Client campaign request
  - Crisis signal detection (negative mention velocity exceeding 5x baseline within 15-minute window)
  - Troll/hostile pattern detection (no specific grievance, inflammatory language, history of hostile behavior)

  #### Technical Implementation
  - 8 rules in seed data

- **FR-8.2** For simulated Twitter/X API behavior, the system **SHALL** implement configurable polling intervals:
  - Critical keywords: 30-second intervals
  - Important keywords: 2-minute intervals
  - Standard keywords: 5-minute intervals
  (Note: This is a simulation parameter for demo purposes, not an API capability claim.)

  #### Technical Implementation
  - **Polling config:** In `data_source` table
    ```sql
    CREATE TABLE data_sources (
        id UUID PRIMARY KEY,
        tenant_id UUID NOT NULL,
        source_type VARCHAR(50) NOT NULL, -- 'simulated_email', 'simulated_twitter', 'simulated_shopify'
        config JSONB, -- { polling_interval_seconds: 30, keywords: [...] }
        status VARCHAR(20) DEFAULT 'active'
    );
    ```

- **FR-8.3** The system **SHALL** support three keyword configuration tiers:
  - Tier 1: Simple flat keyword list
  - Tier 2: Boolean logic (AND, OR, NOT)
  - Tier 3: Industry-suggested keyword clusters

  #### Technical Implementation
  - **Parser:** `backend/src/agent/keywords/parser.py`
    ```python
    class KeywordParser:
        def parse(self, config: dict) -> list[str]:
            if config.get('boolean_logic'):
                return self.parse_boolean(config['expression'])
            elif config.get('clusters'):
                return self.expand_clusters(config['clusters'])
            return config.get('keywords', [])
    ```

- **FR-8.4** The social media preset **SHALL** include configurable brand voice profiles with dimensions: Formality, Humor, Personality, Expertise, Empathy. Brand voice profiles **SHALL** use a structured scoring approach (e.g., "Formality: 7/10") to fit within token budgets.

  #### Technical Implementation
  - **Scores:** 0-10 integer scale per dimension
  - **Token budget:** Max 100-200 tokens when injected

- **FR-8.5** The system **SHALL** generate content using sentiment-aware tone mapping:
  - Positive: Match or exceed enthusiasm
  - Negative: Empathize first, offer solution
  - Neutral: Helpful and concise
  - Sarcastic: Acknowledge concern without being defensive

  #### Technical Implementation
  - **Tone map:** Hardcoded in action generation (FR-2.8)

- **FR-8.6** The social media preset **SHALL** include pre-configured actions for:
  - Draft trend response post
  - Draft brand mention reply
  - Draft review response
  - Draft client campaign update
  - Draft escalation brief
  - Flag for real-time engagement
  - "No response needed" recommendation for troll/hostile mentions

  #### Technical Implementation
  - 7 action templates

- **FR-8.7** The system **SHALL** detect crisis signals when negative mention velocity exceeds 5x baseline within a 15-minute window and escalate to client notification.

  #### Technical Implementation
  - **Velocity tracking:** `backend/src/agent/monitoring/velocity_tracker.py`
  - **Threshold:** `negative_velocity > 5 * baseline`

- **FR-8.8** The system **SHALL** inject platform-specific formatting conventions into generation prompts, including:
  - Twitter/X thread formatting requirements
  - Instagram hashtag placement conventions
  - LinkedIn professional tone requirements
  - TikTok caption conventions (limited to ~2,200 characters)
  - Review site response conventions (Google: thank-then-address, Yelp: semi-formal, Trustpilot: GDPR-aware, G2: B2B technical)

  #### Technical Implementation
  - **Platform specs:** `backend/src/agent/prompts/platform_specs.yaml`

- **FR-8.9** The system **SHALL** escalate to client notification when: (1) mention sentiment is strongly negative AND account has >10K followers, (2) velocity spike exceeds 10x baseline, (3) legal/risk keywords detected, or (4) influencer (>50K followers) posts negative brand content.

  #### Technical Implementation
  - **Escalation rules:** In `crisis_detector.py`

- **FR-8.10** The system **SHALL** require a minimum of Standard Draft mode for accounts exceeding a configurable follower threshold regardless of trend velocity, to ensure brand safety review.

  #### Technical Implementation
  - **Config:** `min_draft_mode` in `brand_voice_profiles`
  - **Override:** If `follower_count > threshold`, auto-select 'standard'

---

### FR-9.0 E-commerce Domain Preset

The system **SHALL** ship with an e-commerce domain preset for small sellers on Shopify and Amazon.

#### Technical Implementation

**Component Structure:**
- `frontend/src/components/domain/ecommerce/EcommerceDashboard.tsx`
- `frontend/src/components/domain/ecommerce/AccountHealthAlerts.tsx`
- `frontend/src/components/domain/ecommerce/InventoryWarning.tsx`
- `frontend/src/components/domain/ecommerce/RefundApproval.tsx`

**Preset Seed Data:**
```typescript
// frontend/src/db/seeds/ecommerce-preset.ts
export const ecommercePresetRules = [
  {
    name: 'Order Issue Detection',
    evaluationPrompt: 'Detect payment failures, address verification issues, fulfillment errors...',
    urgency: 'high',
  },
  {
    name: 'Negative Review Alert',
    prefilterConfig: { star_threshold: 3 },
    urgency: 'medium',
  },
  // ... 10 rules per FR-9.1
];
```

---

- **FR-9.1** The e-commerce preset **SHALL** include pre-configured rules for:
  - Order issue detection (payment failures, address verification, fulfillment errors)
  - Negative product review alert (configurable star threshold)
  - Inventory low-stock warning (configurable threshold)
  - Customer email requiring response
  - Refund/return request
  - Marketplace policy/listing change
  - Account health metric alerts (ODR, late shipment, cancellation rates)
  - A-to-Z guarantee claim detection
  - Chargeback/dispute early warning
  - Customer message response urgency (24hr for Amazon, 48hr for Etsy)

  #### Technical Implementation
  - 10 rules in seed data

- **FR-9.2** The system **SHALL** use hybrid evaluation: deterministic threshold checks for structured API fields plus LLM evaluation for semantic interpretation.

  #### Technical Implementation
  - **Hybrid evaluator:** `backend/src/agent/evaluators/hybrid_ecommerce.py`
    ```python
    async def evaluate(communication: Communication) -> EvaluationResult:
        # 1. Run deterministic checks first
        if deterministic_check_fails(communication):
            return EvaluationResult(triggered=False)
        # 2. LLM evaluation for semantic interpretation
        llm_result = await llm_evaluate(communication)
        return llm_result
    ```

- **FR-9.3** The e-commerce preset **SHALL** implement generic rules with marketplace-specific sub-configurations, supporting both Shopify and Amazon with overrideable field mappings. The system **SHALL** document marketplace-specific field mappings in a configuration file accessible to users.

  #### Technical Implementation
  - **Config:** `marketplace_field_mappings` in tenant settings
  - **Documentation:** `/settings/marketplace-fields` API

- **FR-9.4** The system **SHALL** extract issue type (sizing, damage, quality, shipping, wrong_item, not_as_described, defective, missing_parts) as part of the rule evaluation output.

  #### Technical Implementation
  - **Schema:** `issue_type` enum in `extracted_context.ecommerce` (see FR-1.3)

- **FR-9.5** The system **SHALL** inject product category metadata into action generation prompts, including common issues and resolution options for that category.

  #### Technical Implementation
  - **Product context:** `entity.product.category_metadata` injected into prompts
  - **Token budget:** 100-200 tokens (NFR-3.4)

- **FR-9.6** The e-commerce preset **SHALL** include pre-configured actions for:
  - Draft review response (public, marketplace-compliant, no shipping/troubleshooting details, no requests for removal)
  - Draft customer service reply (private, personalized)
  - Draft refund/return approval (with type-specific tone: no-return, full refund, partial refund, store credit option)
  - Draft supplier reorder email
  - Draft order issue resolution
  - Flag for seller action with suggested next steps

  #### Technical Implementation
  - 6 action templates

- **FR-9.7** The system **SHALL** differentiate tone and format between review responses (public, professional, concise, permanent, first response critical for SEO) and customer emails (private, warm, detailed).

  #### Technical Implementation
  - **Tone guidelines:** Per action type, explicitly different

- **FR-9.8** The system **SHALL** support FBA-specific inventory rules including:
  - Inbound shipment delay/loss detection
  - Stockout prediction based on lead time and velocity
  - Aged inventory threshold warnings (270+, 365+ days)

  #### Technical Implementation
  - **FBA rules:** Separate rule set with FBA-specific checks

- **FR-9.9** The system **SHALL** surface financial context in action previews where applicable:
  - Refunds: Show order value and profit margin context (if available)
  - Reorders: Show current stock value and supplier MOQ
  - Disputes: Show response deadline and escalation risk

  #### Technical Implementation
  - **Financial context:** Injected into `context_summary` for action preview

---

### FR-10.0 Data Source Ingestion and Synthetic Data

The system **SHALL** abstract data sources behind a provider interface and include synthetic data generators for Phase 1 demo purposes.

#### Technical Implementation

**Data Model:**
```sql
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    data_source_id UUID REFERENCES data_sources(id),
    source_type VARCHAR(50) NOT NULL,
    source_metadata JSONB NOT NULL, -- { sender, timestamp, identifiers }
    body_content TEXT NOT NULL,
    body_format VARCHAR(20) NOT NULL DEFAULT 'text', -- 'text', 'html', 'json'
    is_synthetic BOOLEAN DEFAULT false,
    triggered_rules JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    source_type VARCHAR(50) NOT NULL,
    source_name VARCHAR(255) NOT NULL,
    config JSONB,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    last_poll_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_communications_tenant ON communications(tenant_id);
CREATE INDEX idx_communications_source ON communications(data_source_id);
```

**Drizzle Schema:**
```typescript
// frontend/src/db/schema/communications.ts
export const communications = pgTable('communications', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  dataSourceId: uuid('data_source_id').references(() => dataSources.id),
  sourceType: varchar('source_type', { length: 50 }).notNull(),
  sourceMetadata: jsonb('source_metadata').notNull().$type<{
    sender?: string;
    timestamp?: string;
    identifiers?: Record<string, string>;
  }>(),
  bodyContent: text('body_content').notNull(),
  bodyFormat: varchar('body_format', { length: 20 }).notNull().default('text'),
  isSynthetic: boolean('is_synthetic').default(false),
  triggeredRules: jsonb('triggered_rules').$type<string[]>(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Data Source Provider Interface:**
```python
# backend/src/agent/data_sources/base.py
class DataSourceProvider(Protocol):
    async def connect(self) -> None: ...
    async def fetch(self, since: datetime) -> list[Communication]: ...
    async def disconnect(self) -> None: ...

class EmailProvider(DataSourceProvider):
    async def fetch(self, since: datetime) -> list[Communication]:
        # Normalize to Communication model
        ...

class SimulatedEmailProvider(DataSourceProvider):
    async def fetch(self, since: datetime) -> list[Communication]:
        return generate_synthetic_emails(count=10)

class SimulatedTwitterProvider(DataSourceProvider):
    async def fetch(self, since: datetime) -> list[Communication]:
        return generate_synthetic_tweets(count=5)
```

**Synthetic Data Generators:**
```python
# backend/src/agent/data_sources/synthetic/legal_emails.py
def generate_legal_emails(count: int) -> list[Communication]:
    templates = [
        "Dear {attorney}, The hearing scheduled for {date} has been {status}...",
        "Re: {case_number} - Response due {date}...",
    ]
    # Variable substitution and template selection
```

---

- **FR-10.1** The system **SHALL** implement a data source provider interface that normalizes content into: source type, metadata (sender, timestamp, identifiers), and body content (text, HTML, or structured JSON).

  #### Technical Implementation
  - See provider interface above

- **FR-10.2** The system **SHALL** provide simulated data sources for all three domain presets:
  - Simulated email inbox with synthetic emails covering domain scenarios plus noise
  - Simulated Twitter/X API payloads (social media domain)
  - Simulated Shopify/Amazon seller API payloads (e-commerce domain)

  #### Technical Implementation
  - Three providers: `SimulatedEmailProvider`, `SimulatedTwitterProvider`, `SimulatedEcommerceProvider`

- **FR-10.3** The synthetic email generator **SHALL** prioritize hand-crafted templates with variable substitution for reliability and faster iteration. LLM-generated content **MAY** be used for variety in later iterations.

  #### Technical Implementation
  - Template-based generation in `synthetic/*.py`

- **FR-10.4** The synthetic data **SHALL** include entity data (clients, cases, brands, products, orders) that is internally consistent within each domain preset.

  #### Technical Implementation
  - **Entity generators:** `synthetic_entities/legal.py`, `synthetic_entities/social.py`, `synthetic_entities/ecommerce.py`

- **FR-10.5** New synthetic data **SHALL** be receivable on a configurable schedule or triggered manually to demonstrate the system in action.

  #### Technical Implementation
  - **Scheduler:** `backend/src/agent/scheduler/synthetic_poller.py`
  - **Manual trigger:** `POST /api/v1/data-sources/:id/trigger`

- **FR-10.6** The backend **SHALL** expose webhook endpoints for future real-time data sources (email webhooks, API push notifications), normalized through the provider interface before evaluation.

  #### Technical Implementation
  - **Route:** `POST /api/webhooks/email`
  - **Normalization:** Converts to `Communication` model

- **FR-10.7** For Phase 1 simulated sources, the system **SHALL** generate synthetic data in the backend and expose it to the frontend via the CopilotKit agent for rules evaluation.

  #### Technical Implementation
  - **Flow:** Backend generates -> CopilotKit context -> Frontend dashboard

- **FR-10.8** The system **SHALL** implement a polling scheduler that supports configurable intervals per data source, with exponential backoff for failed polls.

  #### Technical Implementation
  - **Backoff:** `2^attempt * base_interval`, max 5 attempts

- **FR-10.9** The system **SHALL** implement rate limiting for external API calls, respecting provider limits and implementing request queuing to prevent 429 errors.

  #### Technical Implementation
  - **Rate limiter:** `backend/src/agent/utils/rate_limiter.py`

- **FR-10.10** The data source connector interface **SHALL** define error handling contracts including: connection failures, timeout handling, partial data handling, and retry policies. Failed data source operations **SHALL** be logged with sufficient context for debugging and surfaced to the user as actionable error messages.

  #### Technical Implementation
  - **Error contracts:** Defined in provider interface
  - **Logging:** `data_source_errors` table with context
  - **UI:** Error cards with retry (FR-4.12)

---

## 3. Non-Functional Requirements

### NFR-1.0 Performance

The system **SHALL** meet latency targets that enable responsive user experience.

#### Technical Implementation

- **NFR-1.1** Rule evaluation (pre-filter + full prompt) **SHALL** complete within 5 seconds per email.
  - **Implementation:** Timeout in `backend/src/agent/evaluator/rules_engine.py`

- **NFR-1.2** Action generation **SHALL** complete within 10 seconds to keep the approval flow responsive.
  - **Implementation:** Timeout in `backend/src/agent/actions/draft_generator.py`

- **NFR-1.3** The revision feedback loop **SHALL** return updated drafts within 10 seconds.
  - **Implementation:** Separate timeout from generation

- **NFR-1.4** Pre-filter LLM calls **SHALL** use lightweight models (Qwen2.5-0.5B or MiniMax-Text-01) targeting sub-500ms inference latency. Network latency to OpenRouter is excluded from this target but **SHALL** be monitored.
  - **Model config:** In `backend/src/config.py`

- **NFR-1.5** The system **SHALL** show loading states with skeleton loaders during evaluation. Skeleton loaders **SHALL** appear on: dashboard initial load, drawer open (context loading), revision submission.
  - **Components:** `ActionCardSkeleton.tsx`, `DrawerSkeleton.tsx`

- **NFR-1.6** The system **SHALL** cache evaluation results for 30-60 seconds to avoid re-evaluation on page refresh.
  - **Cache:** Redis or in-memory cache with TTL
  - **Cache key:** `evaluation:{communication_id}:{rule_id}`

---

### NFR-2.0 LLM Model Selection

The system **SHALL** use models from the Kimi, Qwen, MiniMax, and GLM families via OpenRouter.

#### Technical Implementation

**Model Configuration:**
```typescript
// backend/src/config/models.py
LLM_MODELS = {
    'prefilter': {
        'primary': 'qwen/qwen2.5-0.5b-instruct',
        'fallback': 'THUDM/glm-4-9b',
    },
    'evaluation': {
        'primary': 'minimax/minimax-m2.5',
        'fallback': 'qwen/qwen2.5-7b-instruct',
    },
    'generation': {
        'primary': 'minimax/minimax-m2.5',
        'fast_fallback': 'minimax/minimax-text-01',
        'fallback': 'THUDM/glm-4-9b',
    },
}
```

- **NFR-2.1** Pre-filter stage **SHALL** use Qwen2.5-0.5B or MiniMax-Text-01 for fast, cost-effective yes/no classification. Model selection **SHALL** prioritize availability and latency, defaulting to Qwen2.5-0.5B unless MiniMax-Text-01 demonstrates superior performance on yes/no classification benchmarks.

- **NFR-2.2** Rule evaluation stage **SHALL** use MiniMax-M2.5 or equivalent for strong instruction-following.

- **NFR-2.3** Action generation stage **SHALL** use MiniMax-M2.5 for writing quality, with MiniMax-Text-01 as the fast alternative.

- **NFR-2.4** The system **SHALL** include fallback models if primary models are unavailable:
  - Pre-filter fallback: GLM-4-9B
  - Evaluation fallback: Qwen2.5-7B
  - Generation fallback: GLM-4-9B

- **NFR-2.5** When all primary and fallback models are unavailable, the system **SHALL** log the failure, surface the communication for manual review, and NOT block the pipeline.
  - **Implementation:** Graceful degradation in `backend/src/agent/llm/client.py`

---

### NFR-3.0 Cost Optimization

The system **SHALL** implement cost-effective inference patterns.

#### Technical Implementation

- **NFR-3.1** The pre-filter stage **SHALL** reduce LLM costs by filtering emails before full evaluation, targeting 80-90% cost reduction.
  - **Metrics:** Track emails filtered vs. evaluated

- **NFR-3.2** Entity context injection **SHALL** use summary approach (200-500 tokens) rather than full history.
  - **Implementation:** `backend/src/agent/actions/entity_summarizer.py`

- **NFR-3.3** Brand voice profile injection **SHALL** use structured scoring (e.g., "Formality: 7/10") capped at 100-200 tokens to avoid quality degradation from excessive context.
  - **Implementation:** Compact format in generation prompts

- **NFR-3.4** Product context injection **SHALL** be optimized to 100-200 tokens using structured key fields only.
  - **Implementation:** Only `category`, `common_issues`, `resolution_options` fields

---

### NFR-4.0 Security

The system **SHALL** implement defense-in-depth security including multi-layer tenant isolation.

#### Technical Implementation

- **NFR-4.1** All tenant-scoped tables **SHALL** have RLS policies enabled.
  - See FR-6.0 SQL policies

- **NFR-4.2** The RLS test suite **SHALL** run in CI pipeline on every push and pull request.
  - **Test:** `frontend/__tests__/rls-isolation.test.ts`

- **NFR-4.3** The system **SHALL** not pass tenant ID as a prompt parameter to LLMs, relying instead on database-level isolation.
  - **Validation:** Audit code for tenant ID in prompts

- **NFR-4.4** Approved and rejected actions **SHALL** be logged with timestamps and user identity for audit.
  - See FR-3.2 audit log

- **NFR-4.5** The system **SHALL** implement input sanitization for LLM prompts, stripping or escaping user-controlled content that could inject instructions. Pre-filter prompts **SHALL** be evaluated for injection resistance.
  - **Sanitizer:** `backend/src/agent/utils/prompt_sanitizer.py`
    ```python
    def sanitize_prompt(user_content: str) -> str:
        # Remove or escape potential instruction injection
        patterns = [
            r'ignore\s+previous\s+instructions',
            r'system\s*:',
            r'\<system\>',
        ]
        for pattern in patterns:
            user_content = re.sub(pattern, '[FILTERED]', user_content)
        return user_content
    ```

- **NFR-4.6** For legal domain specifically, prompt data retention **SHALL** have configurable limits, and entity context (client names, case information) **SHALL** not appear in logs accessible to parties other than the authenticated tenant.
  - **Log redaction:** Redact PII in logs for legal tenants

---

### NFR-5.0 Usability

The system **SHALL** prioritize fast scanning and confident decision-making for time-constrained professionals.

#### Technical Implementation

- **NFR-5.1** The dashboard **SHALL** enable a user to scan and act on items in under 60 seconds per session.
  - **Target:** UI optimized for quick scanning

- **NFR-5.2** Mobile experience **SHALL** be review-only (view items, view context) with no approval capability in Phase 1. Mobile approval capability **SHALL** be deferred to Phase 2.
  - **Implementation:** Responsive design with `aria-readonly` on mobile

- **NFR-5.3** In-app notifications **SHALL** appear as badges on sidebar icons; critical items **MAY** support opt-in push notifications.
  - **Component:** `frontend/src/components/notifications/BadgeCount.tsx`

- **NFR-5.4** Notifications for multiple items arriving within 10 minutes **SHALL** be batched into a single notification.
  - **Batching:** 10-minute window in notification service

- **NFR-5.5** The system **SHALL** comply with WCAG AA accessibility standards:
  - Color contrast ratios of 4.5:1 for text on urgency badges
  - All interactive elements **SHALL** have accessible names for screen readers
  - Focus management **SHALL** trap focus in drawer open and return focus to trigger on close
  - Animations (pulsing dot, drawer slide) **SHALL** respect `prefers-reduced-motion`
  - Keyboard navigation **SHALL** allow tab through all action cards and drawer actions

- **NFR-5.6** The system **SHALL** support keyboard shortcuts: `j/k` for navigation, `a` for approve, `r` for reject, `e` for edit, `Enter` to open drawer.
  - **Implementation:** `frontend/src/hooks/useKeyboardShortcuts.ts`

---

### NFR-6.0 Maintainability

The system **SHALL** be designed for extensibility and maintainability.

#### Technical Implementation

- **NFR-6.1** Adding a new data source connector **SHALL** require only implementing the provider interface, with no changes to the rule evaluation pipeline.
  - **Interface:** `DataSourceProvider` protocol

- **NFR-6.2** Adding a new domain preset **SHALL** be achievable by defining domain-specific rules, actions, and entity schemas.
  - **Architecture:** Domain presets as seed data packages

- **NFR-6.3** The rules engine **SHALL** support versioning of prompts to track changes over time.
  - **Schema:** `rule_versions` table
    ```sql
    CREATE TABLE rule_versions (
        id UUID PRIMARY KEY,
        rule_id UUID REFERENCES rules(id),
        prompt_version INTEGER NOT NULL,
        evaluation_prompt TEXT NOT NULL,
        created_at TIMESTAMPTZ DEFAULT NOW()
    );
    ```

- **NFR-6.4** The system **SHALL** track rule accuracy over time based on approval/rejection rates and surface high-rejection rules for review.
  - **Metrics:** `rule_accuracy_stats` aggregated table

---

## Appendix: Architecture Overview

### Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | Next.js (App Router) | Latest |
| Backend | FastAPI | 0.109+ |
| AI Orchestration | LangGraph | Latest |
| AI Integration | CopilotKit (AG-UI) | Latest |
| ORM | Drizzle | Latest |
| Database | PostgreSQL 16 / TimescaleDB | 16 |
| Auth | Better Auth | Latest |
| Validation | Zod (frontend), Pydantic (backend) | Latest |
| UI | ShadCN/ui + Tailwind | Latest |

### Data Flow Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Simulated │     │   Backend   │     │  Frontend   │
│ Data Source│────▶│   Pipeline  │────▶│  Dashboard  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Provider   │     │  LangGraph  │     │  CopilotKit │
│  Interface  │     │   Agent     │     │   Context   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  OpenRouter │
                    │   (LLMs)    │
                    └─────────────┘
```

### Component Tree

```
frontend/src/
├── app/
│   ├── (auth)/
│   │   └── sign-in/, sign-up/
│   ├── dashboard/
│   │   ├── page.tsx
│   │   └── actions/
│   ├── testing/
│   │   └── page.tsx
│   └── api/
│       ├── v1/
│       │   ├── rules/
│       │   ├── actions/
│       │   ├── approvals/
│       │   ├── dashboard/
│       │   └── communications/
│       └── copilotkit/
├── components/
│   ├── rules/
│   ├── actions/
│   ├── dashboard/
│   ├── domain/
│   │   ├── legal/
│   │   ├── social/
│   │   └── ecommerce/
│   └── ui/ (shadcn)
├── db/
│   └── schema/
│       ├── auth.ts
│       ├── rules.ts
│       ├── actions.ts
│       ├── communications.ts
│       ├── approval.ts
│       └── tenant.ts
└── lib/
    ├── schemas/ (Zod)
    ├── db/
    └── utils/

backend/src/
├── main.py
├── config.py
├── agent/
│   ├── prefilter/
│   ├── evaluator/
│   ├── actions/
│   ├── data_sources/
│   ├── prompts/
│   └── utils/
└── api/
    └── routes/
```

### Database Schema Overview

All tables include `tenant_id` column with RLS policies. Core tables:

1. **users** — Better Auth user table
2. **rules** — Rule definitions with prompts, priorities, thresholds
3. **actions** — Action templates linked to rules
4. **rule_actions** — Many-to-many rules-actions relationship
5. **communications** — Normalized incoming communications
6. **entities** — Client, case, brand, product data
7. **generated_actions** — Draft actions with status
8. **approval_audit_log** — Audit trail
9. **data_sources** — Registered data sources
10. **tenant_settings** — Tenant configuration
11. **domain_presets** — Preset rule/action bundles
12. **brand_voice_profiles** — Social media brand voices
13. **rule_versions** — Prompt versioning

### Security Architecture

1. **Application Layer:** All queries filter by `tenant_id` from session
2. **Database Layer:** RLS policies using `current_setting('app.current_tenant')`
3. **Connection Layer:** Reset tenant context on connection checkout
4. **API Layer:** All endpoints require authentication, enforce tenant scoping
5. **Prompt Layer:** Input sanitization to prevent prompt injection

### Deployment Topology

```
┌─────────────────────────────────────────────────────┐
│                    Docker Compose                    │
├─────────────┬─────────────┬─────────────────────────┤
│  Frontend   │   Backend   │       Database          │
│   :3000     │    :8000    │        :5432            │
│  Next.js    │  FastAPI    │    TimescaleDB         │
│             │  LangGraph  │                        │
└─────────────┴─────────────┴─────────────────────────┘
        │              │
        ▼              ▼
   OpenRouter      External APIs
   (LLMs)          (Future)
```

---

**Document Status:** IMPLEMENTATION-READY — Ready for development handoff to coding agents.
