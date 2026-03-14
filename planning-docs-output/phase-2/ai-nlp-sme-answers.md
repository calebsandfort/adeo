# Phase 2b: AI/NLP Architecture SME Answers

## Questions from Legal Domain SME

### Question 1: Prompt Accuracy for Legal Context Extraction

**Answer:**

For legal context extraction, the system should prioritize **explicit uncertainty over confident incorrectness**. The extraction prompts MUST be designed to return "unknown" rather than guess when confidence is low.

**Recommended Approach:**

1. **Structured extraction with confidence bands:**
```json
{
  "extracted_dates": [
    {
      "date": "2024-03-15",
      "confidence": "high",
      "source": "explicit March 15 mention"
    },
    {
      "date": null,
      "confidence": "low",
      "source": "referenced as 'next week' without specification"
    }
  ]
}
```

2. **Why ambiguous extraction is worse than no extraction:**
   - A wrong date creates false confidence — the attorney may NOT check because the system "already found it"
   - "Sometime next week" extracted as "March 15" is actively dangerous
   - Missing a date entirely is recoverable (attorney will manually check); acting on wrong date is not

3. **Implementation in prompt:**
```
## Extraction Rules
- If a specific date is mentioned (e.g., "March 15", "03/15/2024"), extract it with high confidence
- If relative dates are used ("next week", "in a few days"), return confidence: low and extracted_value: null
- If no date is found, return confidence: null, extracted_value: null
- NEVER convert ambiguous dates to specific ones — prefer "unknown" over guessing
```

**Legal-Specific Refinement:** For court-ordered deadlines specifically (the highest-risk category), the confidence threshold should be set higher than other date types. If the extracted date has <85% confidence, surface it as "date detected — verify manually" rather than presenting it as a confirmed deadline.

---

### Question 2: Legal Disclaimer Generation

**Answer:**

**Recommendation: Append as post-generation, not in prompt.**

There are three reasons for this architecture:

1. **Prompt complexity:** Adding disclaimer instructions to every legal generation prompt adds token overhead and may affect generation quality (the LLM may truncate content to fit the disclaimer)

2. **Configurability:** Different jurisdictions or practice areas may have different requirements. Post-generation append allows easy configuration:
```typescript
function appendLegalDisclaimer(draft: string, domain: string): string {
  const disclaimers = {
    legal: "\n\n---\nThis draft was generated with AI assistance and requires attorney review before use.",
    default: "\n\n---\nThis is an AI-generated draft. Please review and customize before sending."
  };
  return draft + (disclaimers[domain] || disclaimers.default);
}
```

3. **Workflow flexibility:** Some use cases may not need the disclaimer (internal notes, personal reminders). Post-generation append allows it to be toggled per action type.

**Implementation:**

- Add disclaimer configuration to the action type definition
- Append after the generated content, not inline
- Do NOT include in prompt — it affects generation quality and adds noise
- For Phase 1, default to including it; allow users to disable in settings

---

### Question 3: Extraction Confidence for Dates

**Answer:**

**Yes, implement confidence-based surface treatment.**

For court-ordered deadlines specifically, implement a three-tier confidence system:

| Confidence Level | UI Treatment | Example |
|-----------------|---------------|---------|
| **High (>90%)** | Show date prominently, auto-suggest action | "Deadline: March 15, 2024" |
| **Medium (70-90%)** | Show date with verification prompt | "Deadline detected: March 15 — please verify" |
| **Low (<70%)** | Do NOT show date, prompt manual review | "Court deadline detected — verify date manually" |

**Implementation in evaluation prompt:**
```json
{
  "extracted_context": {
    "deadline_date": "2024-03-15",
    "deadline_confidence": 0.82,
    "verification_required": true,
    "verification_prompt": "Court deadline detected — please verify the date in the original document"
  }
}
```

**UX from AI/NLP perspective:** The confidence score should be returned as a float (0.0-1.0) but the UI handles the thresholding. This keeps the LLM output simple while allowing business logic to adjust thresholds without re-prompting.

---

## Questions from Social Media Domain SME

### Question 4: Model Selection for Fast Drafts

**Answer:**

**Recommendation: MiniMax-Text-01 for Fast mode, MiniMax-M2.5 for Enhanced mode.**

| Mode | Recommended Model | Estimated Latency | Quality Level |
|------|-------------------|-------------------|---------------|
| Fast Draft (3-5s) | MiniMax-Text-01 | ~2-3 seconds | Adequate for approval |
| Standard (8-12s) | MiniMax-M2.5 | ~4-6 seconds | Good |
| Enhanced (15-20s) | MiniMax-M2.5 | ~8-12 seconds | Best |

**Rationale:**

1. **MiniMax-Text-01** is optimized for speed while maintaining reasonable quality for short-form content
2. **MiniMax-M2.5** provides the best writing quality for higher-stakes content
3. **Kimi-A3C** can serve as a fallback if MiniMax is unavailable
4. **Do NOT use Qwen2.5-0.5B** for generation — only for pre-filter. Its quality is insufficient for content generation.

**Mode Selection Logic:**
```typescript
function selectModelForDraftMode(urgency: 'critical' | 'standard' | 'enhanced'): string {
  switch (urgency) {
    case 'critical': return 'minimax-text-01'; // Fastest
    case 'standard': return 'minimax-m2.5';    // Balanced
    case 'enhanced': return 'minimax-m2.5';    // Quality
  }
}
```

---

### Question 5: Sentiment Detection Accuracy

**Answer:**

**Recommendation: Integrate into generation prompt, not separate call.**

**Why separate sentiment detection is problematic:**
- Additional latency (~500ms-1s per call)
- Token cost overhead
- Potential inconsistency between detection and generation
- Social media sentiment is context-dependent — the generation model may have better context

**Integrated approach:**
```python
SENTIMENT_INTEGRATED_PROMPT = """
Generate a response to the following social media mention.

## Detected Sentiment
Analyze the sentiment of the mention:
- positive: User is praising, complimenting, or expressing satisfaction
- negative: User is complaining, frustrated, or expressing dissatisfaction
- neutral: User is asking a question, making a statement, or being informational
- sarcastic: User appears to mean the opposite of what they say (common in social media)

## Tone Requirements
Based on detected sentiment:
- positive: Match or exceed their enthusiasm, express gratitude
- negative: Empathize first, offer solution, be sincere not defensive
- neutral: Be helpful and friendly, concise
- sarcastic: Acknowledge the concern without being defensive

## Mention Content
{user_mention}
"""
```

**Accuracy Reality Check:**
- Current models (MiniMax, Kimi, Qwen) achieve ~85-90% accuracy on explicit sentiment
- Sarcasm detection drops to ~60-70% — always human review for sarcastic mentions
- Emojis are generally well-understood by current models
- Slang varies by model — test with domain-specific examples

**Fallback:** If accuracy problems emerge in testing, add explicit sentiment detection as a pre-processing step, but start integrated.

---

### Question 6: Token Budget Optimization

**Answer:**

**Recommendation: 200-400 tokens for voice profile, maximum 500 tokens.**

| Context Budget | Tokens | Degradation Risk |
|---------------|--------|------------------|
| Minimal (voice guidelines only) | 50-100 | Low quality variation |
| Recommended (guidelines + 2 examples) | 200-300 | Optimal |
| Extended (guidelines + 5 examples) | 400-500 | Acceptable |
| Excessive (>500 tokens) | 500+ | Quality degradation |

**Why degradation happens:**
- LLM attention diffuses across long context
- Examples start to contradict each other (edge cases conflict)
- The same guidance repeated becomes noise

**Optimal voice profile structure (aim for 200-300 tokens total):**
```python
VOICE_PROFILE_PROMPT = """
## Brand Voice
- Formality: {1-10 scale}
- Tone: {adjective, adjective, adjective}
- Example: "{example sentence}"

## Do's
- {do 1}
- {do 2}

## Don'ts
- {dont 1}
- {dont 2}

## Example (Good)
"{good_example}"

## Example (Avoid)
"{poor_example}"
"""
```

**Token counting tip:** 1 token ~= 0.75 words. A 250-token profile is roughly 180-200 words.

---

## Questions from E-commerce Domain SME

### Question 7: Issue Type Extraction

**Answer:**

**Recommendation: Integrated into evaluation prompt, but with explicit classification field.**

For e-commerce issue detection, the rule evaluation should extract issue type as part of the standard extraction, not as a separate pipeline step. Here's why:

1. **Token efficiency:** The LLM is already reading the email — extracting issue type adds minimal tokens vs. separate call
2. **Context awareness:** The same words ("too small") might mean sizing for apparel, fit for furniture — the model needs context to classify correctly
3. **Latency reduction:** One call instead of two

**Implementation:**
```python
EVALUATION_PROMPT = """
## Task
Evaluate if this customer communication requires action.

## Output Schema
Return JSON:
{
  "triggered": boolean,
  "confidence": float,
  "issue_type": "sizing|damage|quality|shipping|wrong_item|other|null",
  "product_category": "apparel|electronics|home|beauty|food|other|null",
  "extracted_context": {...},
  "reasoning": "..."
}
```

**Issue Type Taxonomy for E-commerce:**
| Issue Type | Key Indicators | Product Categories |
|------------|---------------|-------------------|
| sizing | "too small", "too big", "doesn't fit", size chart | apparel, shoes |
| damage | "broken", "cracked", "damaged", "arrived damaged" | electronics, home, beauty |
| quality | "defective", "not as described", "poor quality", "stopped working" | all |
| shipping | "never arrived", "lost", "delayed", "tracking" | all |
| wrong_item | "wrong item", "received instead", "not what I ordered" | all |

---

### Question 8: Product-Specific Response Generation

**Answer:**

**Recommendation: Category label + guidelines is sufficient; few-shot examples are optional for Phase 1.**

**Evidence-based recommendation:**

1. **Category + guidelines approach works well because:**
   - Product categories have consistent patterns (sizing issues always need size exchange language)
   - Guidelines can capture 90% of the nuance without the overhead of examples
   - Fewer tokens = faster generation + lower cost

2. **When to add few-shot examples:**
   - If the category + guidelines approach produces generic responses in testing
   - For high-volume product categories where response quality directly impacts revenue
   - For complex categories (beauty with allergic reactions, electronics with troubleshooting)

**Implementation:**
```python
ACTION_GENERATION_CONTEXT = """
## Product Context
- Category: {product_category}
- Common issues: {common_issue_types}
- Resolution options: {available_resolutions}

## Response Guidelines for {product_category}
- Sizing: Acknowledge fit concern, offer exchange or refund, mention size chart
- Damage: Apologize, request photo, offer immediate replacement or refund
- Quality: Take seriously, do not dispute, offer refund or replacement

## Tone
- Professional but warm
- Solution-oriented
- Concise (under 150 words for review responses)
"""
```

**Verification in testing:** If generated responses for a category are consistently generic (e.g., "We're sorry for the inconvenience"), add 2-3 few-shot examples for that category only.

---

### Question 9: Token Optimization for Product Context

**Answer:**

**Recommendation: Use structured product context with relevance scoring, aim for 100-200 tokens.**

**Token Optimization Strategies:**

| Strategy | Tokens Saved | Tradeoff |
|----------|-------------|----------|
| Flatten to key fields only | ~50% | Lose nuance |
| Category-based templates | ~40% | Good quality |
| Include common issues only | ~30% | Minimal quality loss |

**Optimal Product Context Format (100-200 tokens):**
```python
def get_product_context_for_prompt(product: ProductEntity) -> str:
    return f"""
Product: {product.name}
Category: {product.category}
Common Issues: {', '.join(product.common_issues[:3])}
Resolution Options: {', '.join(product.available_resolutions[:3])}
Tone Notes: {product.response_tone_notes or 'Standard professional'}
"""
# ~80-150 tokens depending on product
```

**What to EXCLUDE from product context:**
- Full product descriptions (not relevant for response)
- Pricing history (irrelevant)
- Inventory levels (only relevant for stock issues, handled separately)
- Marketing copy

**Context Level by Action Type:**
| Action Type | Product Context Needed |
|-------------|----------------------|
| Review response | Category + common issues |
| Customer email | Full context |
| Refund request | Category + resolution options |
| Replacement request | Category + shipping info |

---

## Questions from UX Designer SME

### Question 10: Rule Evaluation Latency

**Answer:**

**Realistic Latency Estimates:**

| Pipeline Stage | Target | Realistic (P50) | Realistic (P95) | Notes |
|---------------|--------|-----------------|------------------|-------|
| Pre-filter | 0.5s | 200-400ms | 800ms | Qwen2.5-0.5B |
| Full evaluation | 3-4s | 1.5-2.5s | 4-5s | MiniMax-M2.5 |
| Total (per rule) | 5s | 2-3s | 5-6s | With overhead |

**Factors affecting latency:**
1. **Model selection:** MiniMax-Text-01 is ~2x faster than MiniMax-M2.5
2. **Prompt length:** Each 100 tokens adds ~100-200ms
3. **Server load:** OpenRouter latency varies by time of day
4. **Email length:** Longer emails = more tokens = slower

**UX Implications:**

1. **For live evaluation in UI:**
   - Pre-filter only: ~200-500ms — can show instant pass/fail
   - Full evaluation: 2-5s — requires loading state

2. **Recommendation:**
   - Show "evaluating..." skeleton while rules run
   - Use optimistic UI — show results as they complete
   - Cache results for 30-60 seconds to avoid re-evaluation on page refresh

3. **Latency vs. Accuracy tradeoff:**
   - If 5-second requirement is firm, use faster models (MiniMax-Text-01) for evaluation
   - If accuracy is priority, accept 6-8 second evaluation and show progressive disclosure

---

### Question 11: Parallel vs. Sequential Action Generation

**Answer:**

**Recommendation: Parallel execution with timeout handling.**

**Architecture:**

```python
async def generate_actions_for_triggered_rules(email, triggered_rules):
    # Create generation tasks for each triggered rule's actions
    tasks = []
    for rule in triggered_rules:
        for action in rule.suggested_actions:
            tasks.append(generate_action(action, email, rule))

    # Execute in parallel with timeout
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # Filter out timeouts/errors, keep successful generations
    successful = [r for r in results if isinstance(r, ActionResult)]

    return successful
```

**Latency Impact:**

| Approach | Time for N Actions | User Perception |
|----------|-------------------|-----------------|
| Sequential | N x 3-5 seconds | 10-25 seconds — feels slow |
| Parallel | ~3-5 seconds | "Instant" — good UX |

**Timeout Handling:**
```python
try:
    result = await asyncio.wait_for(
        generate_action(action, email, rule),
        timeout=8.0  # Per-action timeout
    )
except asyncio.TimeoutError:
    # Log, mark as "generation failed — tap to retry"
    # Do NOT block other actions
    results.append(GenerationError(action.id, "timeout"))
```

**Conflict Detection Post-Generation:**
- Run conflict detection AFTER all actions generated (parallel)
- Conflicts are rare enough that sequential generation for conflict check isn't needed
- Flag conflicts at display time, not generation time

---

## Questions from Integration Engineer SME

### Question 12: Tenant Context in Rule Evaluation

**Answer:**

**Recommendation: Rules evaluated within tenant's database context using SET approach.**

The tenant context should flow through the evaluation pipeline as follows:

1. **At request initialization:** Set `app.current_tenant` via the middleware (as designed in Integration Engineer analysis)

2. **Rule retrieval:** Use tenant-scoped query:
```typescript
// Rules are tenant-specific — RLS handles isolation
const rules = await db.query.rules.findMany({
  where: eq(rules.tenantId, currentTenantId)
});
```

3. **At evaluation time:** The rule configuration is already tenant-isolated because:
   - Rules are stored in tenant-scoped table
   - RLS policy ensures tenant A cannot read tenant B's rules
   - No additional tenant parameter passing needed

4. **For pre-filter optimization:** The pre-filter keywords can be cached per-tenant without cross-tenant leakage because the cache key includes tenant_id.

**Why NOT pass tenant as prompt parameter:**
- Adds tokens to every prompt (cost)
- Risk of prompt injection if tenant ID is improperly escaped
- The database-level isolation is more secure (defense in depth)

**Implementation Flow:**
```
Request arrives → Auth middleware identifies tenant
  → SET app.current_tenant = {tenant_id}
  → Fetch tenant's rules (RLS ensures isolation)
  → For each rule, run pre-filter then full evaluation
  → Actions generated are stored with tenant_id (RLS protected)
```

---

### Question 3: Entity Context for Action Generation

**Answer:**

**Recommendation: Entity lookup via tenant-scoped database connection, ensuring complete isolation.**

**Architecture:**

```typescript
async function generateActionWithEntityContext(
  actionDefinition: ActionDefinition,
  triggeringEmail: Email,
  tenantId: string
) {
  // tenantId already set via SET app.current_tenant

  // 1. Extract entity references from email/rule
  const entityRefs = extractEntityRefs(actionDefinition, triggeringEmail);

  // 2. Lookup entities via tenant-scoped connection (RLS protected)
  const entities = await Promise.all(
    entityRefs.map(ref => lookupEntity(ref.type, ref.identifier))
  );

  // 3. Build context from retrieved entities
  const entityContext = buildEntityContext(entities);

  // 4. Generate action with context
  const prompt = buildGenerationPrompt(actionDefinition, entityContext);
  const generated = await llm.generate(prompt);

  return generated;
}
```

**Why this ensures isolation:**

1. **SET + RLS at lookup time:** When `lookupEntity()` runs, the RLS policy ensures only the current tenant's entities are returned
2. **No cross-tenant joins possible:** Even if the prompt tries to reference another tenant's entity, the lookup returns nothing
3. **Defense in depth:** Two layers — application passes no tenant ID to LLM, database blocks unauthorized access

**Entity Lookup Examples by Domain:**

| Domain | Entity Types | Lookup Method |
|--------|-------------|---------------|
| Legal | clients, cases, opposing counsel | case_number pattern match, client name |
| Social Media | brands, campaigns | brand_id from configured accounts |
| E-commerce | products, orders, customers | order_id, customer_email, product_sku |

**Fallback:** If entity lookup fails (not found), generate action with placeholder "[ENTITY_NOT_FOUND]" and prompt user to verify the reference manually. Do NOT block generation.

---

## Summary of Answers

| Question | Key Recommendation |
|----------|-------------------|
| Legal context extraction | Return "unknown" rather than guess; ambiguous dates worse than no dates |
| Legal disclaimer | Append post-generation, not in prompt |
| Date confidence threshold | Three-tier UI treatment based on confidence bands |
| Fast draft model | MiniMax-Text-01 for speed, MiniMax-M2.5 for quality |
| Sentiment detection | Integrate into generation prompt, not separate call |
| Voice profile tokens | 200-400 tokens maximum |
| Issue type extraction | Integrated into evaluation prompt |
| Product response format | Category + guidelines sufficient; few-shot optional |
| Product context tokens | 100-200 tokens optimized |
| Evaluation latency | 2-3s P50, 5-6s P95 realistic |
| Action generation | Parallel with timeout handling |
| Tenant in evaluation | Use SET + RLS approach (already tenant-isolated) |
| Entity lookup | Tenant-scoped via RLS-protected connection |

---

## Architectural Implications for the Tech Stack

Given FastAPI + LangGraph backend and OpenRouter integration:

1. **LangGraph Node Design:**
   - `prefilter_node`: Runs Qwen2.5-0.5B or MiniMax-Text-01
   - `evaluate_node`: Runs MiniMax-M2.5 with structured output
   - `action_generate_node`: Parallel execution with timeout
   - All nodes receive `tenant_id` via graph state (already set at request entry)

2. **State Management:**
   - Graph state includes: email, triggered_rules, generated_actions, tenant_id
   - Confidence scores passed through for UI thresholding
   - Revision history stored in tenant-scoped table

3. **Latency Management:**
   - Pre-filter: 500ms timeout
   - Evaluation: 3s timeout per rule
   - Action generation: 8s timeout per action, parallel across actions
   - Timeout handling: Return partial results + "retry" option

This architecture respects all constraints while maintaining security isolation and performance targets.
