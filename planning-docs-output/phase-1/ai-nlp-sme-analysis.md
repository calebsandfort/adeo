# Phase 1 SME Analysis: AI/NLP Architecture

## Executive Summary

The Adeo system requires a carefully designed AI pipeline that balances detection accuracy, cost efficiency, and latency constraints. Given the tech stack (FastAPI + LangGraph backend, OpenRouter for LLM access, constrained to Kimi/Qwen/MiniMax/GLM families), I provide the following architecture recommendations for the rule evaluation and action generation pipeline.

**Key constraints driving recommendations:**
- **5-second latency** for rule evaluation (pre-filter + full prompt)
- **10-second latency** for action generation and revision
- **Cost optimization** via pre-filter two-pass approach
- **OpenRouter** with Kimi/Qwen/MiniMax/GLM model families only

---

## Question 1: Optimal Structure for Rule Evaluation Prompts

**Recommendation:** Use structured output schemas with explicit extraction fields, include 2-3 representative few-shot examples, and avoid chain-of-thought reasoning in production prompts (for latency/cost reasons).

### Prompt Structure

```python
SYSTEM_PROMPT = """
You are a rule evaluation engine. Given an email, determine if it matches the rule condition.

## Rule Definition
{rule_description}

## Output Schema
Return a JSON object with:
- "triggered": boolean - does the rule match?
- "confidence": float - 0.0 to 1.0 confidence score
- "extracted_context": object - relevant dates, parties, case references
- "reasoning": string - brief justification (for audit)

## Examples
{examples_section}

## Email Content
{email_content}

## Instructions
- Be precise but avoid over-triggering
- Only trigger if action is genuinely needed
- Extract specific entities when present
"""
```

### Why This Structure Works

1. **Structured output** enables deterministic parsing and downstream processing without ambiguity
2. **Few-shot examples** (2-3) dramatically improve consistency without adding significant token cost
3. **Explicit schema** prevents the LLM from returning free-form text that requires additional parsing
4. **Extracted context** field allows downstream actions to reference specific dates, case numbers, etc.

### Tradeoff Analysis

| Approach | Pros | Cons |
|----------|------|------|
| Few-shot (2-3 examples) | Good accuracy, low token overhead | Requires curated examples per rule |
| Chain-of-thought | Better reasoning on edge cases | +30-50% tokens, slower inference |
| Zero-shot | Lowest cost | Inconsistent output, requires robust prompting |

**Recommendation for Phase 1:** Use few-shot with 2 examples (one positive trigger, one clear negative). Skip CoT unless accuracy problems emerge in testing. The latency constraint (5 seconds) favors simpler prompts.

### Example Format

```json
{
  "examples": [
    {
      "input": "The hearing scheduled for March 15 has been cancelled...",
      "output": {"triggered": true, "confidence": 0.95, "extracted_context": {"hearing_date": "2024-03-15"}}
    },
    {
      "input": "Thanks for your help with the filing last week.",
      "output": {"triggered": false, "confidence": 0.98, "extracted_context": {}}
    }
  ]
}
```

---

## Question 2: Pre-Filter LLM Model Tier

**Recommendation:** Use a lightweight/fast model (e.g., Qwen2.5-0.5B or MiniMax-Text-01) with a very short prompt (under 100 tokens). Cost savings: ~90% vs. full evaluation.

### Model Selection for Pre-Filter

Given OpenRouter availability, prioritize:

| Model | Context | Speed | Cost Tier | Recommended For |
|-------|---------|-------|-----------|-----------------|
| Qwen2.5-0.5B | 8K | ~100ms | $0.01/1M | Pre-filter (yes/no) |
| MiniMax-Text-01 | 32K | ~200ms | $0.02/1M | Pre-filter with extraction |
| Kimi-A3C | 32K | ~200ms | $0.03/1M | Pre-filter if context needed |
| GLM-4-9B | 128K | ~300ms | $0.10/1M | Fallback pre-filter |

### Pre-Filter Prompt Design

```python
PRE_FILTER_PROMPT = """
Briefly answer: Does this email potentially contain a {rule_keyword}?
Reply with only YES or NO.

Email: {email_preview}
"""
```

This should run in under 500ms and cost ~$0.0001 per email.

### Cost/Accuracy Tradeoff

| Configuration | Est. Cost/email | False Negative Risk | Notes |
|---------------|----------------|---------------------|-------|
| No pre-filter | $0.02-0.05 | 0% | All emails go to full evaluation |
| Fast pre-filter | $0.0001 + $0.02 | ~5-10% | Some relevant emails slip through |
| Keyword-only | $0 | ~15-20% | Simple pattern matching |

**Recommendation:** Implement fast pre-filter. The 5-10% false negative rate is acceptable because:
1. Pre-filters are conservative (designed to avoid false positives, not catch everything)
2. Lower-priority rules can still catch what high-priority pre-filters miss
3. Cost savings (90%+) fund more thorough evaluation on passed emails

**Implementation Note:** If pre-filter latency adds unacceptable overhead, run it in parallel with other pre-filters rather than sequentially.

---

## Question 3: Synchronous vs. Asynchronous Rule Evaluation

**Recommendation:** Use **asynchronous evaluation with progressive disclosure** — surface high-priority actions immediately while lower-priority rules are still evaluating.

### UX/Architecture Rationale

| Approach | User Experience | Complexity | When It Matters |
|----------|-----------------|------------|-----------------|
| Synchronous | Blocks until all rules evaluated | Simple | Low email volume |
| Async with progressive | Shows results as they complete | Moderate | High email volume |
| Async batched | Shows all after threshold (e.g., 3s) | Moderate | Real-time triage |

### Implementation Approach

```python
# Pseudo-code for async pipeline
async def evaluate_email(email):
    tasks = []
    for rule in sorted_rules_by_priority():
        task = asyncio.create_task(evaluate_rule(rule, email))
        tasks.append((rule.priority, task))

    results = {}
    for priority, task in tasks:
        try:
            result = await asyncio.wait_for(task, timeout=3.0)
            results[rule.id] = result
            # Early exit: if high-priority rule triggered and no conflict expected
            if priority == "critical" and result.triggered:
                await task_cancel_remaining_lower_priority()
        except TimeoutError:
            # Log but don't block
            pass

    return results
```

### UX Design

1. **High-priority triggers (urgent)** — Show immediately with "evaluating more rules..." indicator
2. **All rules complete** — Update UI to show complete set
3. **Stale evaluation** — If rules take >5 seconds, surface what's available and mark others as "still analyzing"

This balances the need for quick response on urgent items against the thoroughness of evaluating all rules.

---

## Question 4: Handling Conflicting Rules

**Recommendation:** Flag conflicts for explicit user resolution when contradictory actions are generated. Use priority ordering for non-contradictory triggers.

### Conflict Detection Logic

```python
def detect_conflict(actions_a, actions_b):
    # Define action categories that cannot coexist
    mutually_exclusive = {
        "flag_for_urgent_review": ["draft_response", "draft_reschedule"],
        "dismiss_as_noise": ["draft_response", "draft_client_update"],
        "flag_for_legal_action": ["dismiss_as_noise"]
    }

    for cat_a in actions_a:
        for cat_b in actions_b:
            if cat_b in mutually_exclusive.get(cat_a, []):
                return True
    return False
```

### UX for Conflicts

When conflicts detected:
1. Show both actions to the user with a clear "Conflict detected" indicator
2. Display side-by-side comparison of the contradictory recommendations
3. User selects which action to proceed with
4. Log conflict for future rule tuning (conflict patterns inform rule refinement)

### Why Not Automatic Resolution

Priority ordering implicitly resolves conflicts, but:
- Rules with equal priority would require arbitrary tie-breaking
- User trust is higher when they see what triggered and why there was a conflict
- Conflicts are relatively rare (good indicator of rule overlap that needs tuning)

---

## Question 5: Twitter/X API Rate Limits and Polling Strategy

**Recommendation:** Implement tiered polling with keyword priority levels. High-priority keywords poll every 30 seconds; standard keywords every 2 minutes; low-priority every 5-10 minutes.

### Tiered Approach

```python
POLLING_CONFIG = {
    "tier_1_critical": {
        "keywords": ["brand_crisis", "viral_complaint"],  # User-defined
        "interval_seconds": 30,
        "max_results_per_poll": 50
    },
    "tier_2_important": {
        "keywords": ["brand_mention", "review_alert"],
        "interval_seconds": 120,
        "max_results_per_poll": 100
    },
    "tier_3_standard": {
        "keywords": ["industry_trend", "competitor_mention"],
        "interval_seconds": 300,
        "max_results_per_poll": 200
    }
}
```

### Cost Management

- Track API usage per tier and alert when approaching limits
- Implement exponential backoff on rate limit errors
- Consider webhooks (if Twitter X API Enterprise) as alternative to polling for tier 1

### Practical Constraints

- Twitter/X API (Basic tier): 500K tweets/month, ~1000/day
- At 30-second polling, that's ~2880 API calls/day — feasible for tier 1 only
- Most social media managers don't need real-time (minutes matter more than seconds)

---

## Question 6: Structured API Data vs. Email Prompts

**Recommendation:** Use hybrid approach — deterministic threshold checks for structured API fields + LLM evaluation for semantic interpretation.

### Hybrid Architecture

```python
def evaluate_api_payload(rule, payload):
    # Step 1: Deterministic field checks (fast, no LLM)
    if rule.has_threshold_field:
        if payload[rule.threshold_field] < rule.threshold:
            return {"triggered": True, "method": "deterministic"}

    # Step 2: Semantic interpretation via LLM (for complex conditions)
    if rule.requires_nlp:
        llm_result = await evaluate_with_llm(rule, payload)
        return llm_result

    return {"triggered": False, "method": "deterministic"}
```

### When to Use Each

| Condition Type | Method | Example |
|---------------|--------|---------|
| Numeric threshold | Deterministic | inventory < reorder_threshold |
| Boolean flag | Deterministic | order.status == "payment_failed" |
| Semantic interpretation | LLM | "Does this complaint require response?" |
| Context extraction | LLM | "Extract customer sentiment and key issue" |

### E-commerce Specific Rules

- **Order issue detection**: Deterministic check on `status` field + LLM for context
- **Negative review alert**: Deterministic check on `rating` field + LLM for response appropriateness
- **Inventory warning**: Deterministic check on `quantity` vs. threshold
- **Customer email requiring response**: LLM evaluation (unstructured text)

**Estimated impact:** 60-70% of e-commerce rules can use deterministic checks, reducing LLM costs significantly.

---

## Question 7: Action Generation Prompt Structure

**Recommendation:** Use structured templates with fill-in-the-blank sections, but allow LLM creative freedom within sections. Enforce consistency via system prompts and output schemas.

### Action Generation Prompt Template

```python
ACTION_GENERATION_PROMPT = """
You are generating a {action_type} for review.

## Context
- Client/Entity: {entity_name}
- Case Reference: {case_reference}
- Extracted Dates: {extracted_dates}
- Rule That Triggered: {rule_name}
- Triggering Content: {triggering_content}

## Action Type
{action_description}

## Output Format
Return a JSON object:
{{
  "subject": "...",
  "body": "...",
  "tone": "formal|professional|casual",
  "includes": ["date_confirmation", "next_steps", "contact_info"]
}}

## Tone Guidelines
{domain_tone_guidelines}

## Constraints
- Do not make up facts not in context
- Use placeholder [BRACKETS] for information you don't have
- Keep under {max_words} words
"""
```

### Template vs. Creative Freedom

| Aspect | Template Enforced | LLM Freedom |
|--------|------------------|--------------|
| Structure | Section headings, required fields | None |
| Tone | Domain-specific guidelines | Can adjust within constraints |
| Content | Must include extracted context | How to present it |
| Formatting | Professional conventions | Sentence structure |

### Consistency Across Generations

1. **Domain presets include tone guidelines** — legal communications vs. social media have explicit style guides
2. **Output schema validation** — ensure all required fields are present
3. **User can provide style examples** — "Write in the style of these sample emails"
4. **Action type versioning** — changes to prompts track explicit versions

---

## Question 8: Revision Prompt Structure

**Recommendation:** Use multi-turn conversation approach rather than single-prompt reconstruction. Include full context (original email + draft + feedback) but structure as conversation.

### Revision Architecture

```python
# Store revision history as conversation turns
revision_thread = [
    {"role": "system", "content": original_prompt},
    {"role": "user", "content": triggering_email},
    {"role": "assistant", "content": original_draft},
    {"role": "user", "content": feedback_1},
    {"role": "assistant", "content": revised_draft_1},
    {"role": "user", "content": feedback_2},
    # ... additional rounds
]
```

### Alternative: Single Prompt Reconstruction

```python
REVISION_PROMPT = """
Original email:
{original_email}

Current draft:
{draft}

User feedback: {feedback}

Generate a revised draft that incorporates the feedback while maintaining:
- Original intent and facts
- Professional tone
- {domain_specific_constraints}

Return only the revised content.
"""
```

### Comparison

| Approach | Pros | Cons |
|----------|------|------|
| Multi-turn conversation | Maintains context, natural for LLM | More complex state management, higher token cost |
| Single prompt reconstruction | Simpler, deterministic context | May lose nuance, higher prompt engineering requirement |

**Recommendation:** Use multi-turn approach but cap at 5 rounds to prevent degradation. After 5 rounds, prompt user to make direct edits.

### Preventing Degradation

1. **Cumulative context window limit** — after 5 revisions, summarize earlier feedback and start fresh context
2. **Quality check** — if revision is significantly shorter or loses key information, flag for human review
3. **Reference original** — always include original email in context to prevent drift

---

## Question 9: Example-Based Rule Creation — Example Count

**Recommendation:** Minimum 3 positive examples, 2 negative examples. Actively request negative examples from users.

### Example Requirements

| Example Type | Minimum | Optimal | Purpose |
|--------------|---------|---------|---------|
| Positive (should trigger) | 3 | 5-7 | Define the signal |
| Negative (should NOT trigger) | 2 | 3-5 | Define boundaries |

### Active Collection Flow

```
User: "I want to detect when opposing counsel files a motion to compel"

System:
1. "Show me 3-5 emails that represent motions to compel"
2. "Show me 2-3 emails that look similar but should NOT trigger (e.g., routine discovery requests)"
3. Generate prompt from examples + "Any rules you want to add as pre-filter keywords?"
```

### Prompt Generation from Examples

```python
def generate_rule_prompt(examples_positive, examples_negative):
    return f"""
Based on the following examples, create a rule evaluation prompt.

## Should Trigger (Positive Examples)
{format_examples(examples_positive)}

## Should NOT Trigger (Negative Examples)
{format_examples(examples_negative)}

## Pattern to Extract
Analyze what distinguishes positive from negative examples.
Write an evaluation prompt that captures this distinction.

## Output
Return the evaluation prompt text that would correctly classify:
- All positive examples as triggered
- All negative examples as not triggered
"""
```

### Validation

- Run generated prompt against provided examples — should achieve 100% accuracy
- If not, ask for more discriminating examples
- User can edit generated prompt before saving

---

## Question 10: Prompt Quality Validation

**Recommendation:** Implement "test rule" feature that runs against historical emails and shows trigger results. Add basic static analysis for common issues.

### Test Rule Interface

```python
async def test_rule(rule, test_emails):
    results = []
    for email in test_emails:
        pre_filter_passed = await run_prefilter(rule, email)
        if pre_filter_passed:
            evaluation = await evaluate_rule(rule, email)
        else:
            evaluation = {"triggered": False, "reason": "pre-filter failed"}

        results.append({
            "email_id": email.id,
            "triggered": evaluation.triggered,
            "confidence": evaluation.confidence,
            "extracted_context": evaluation.extracted_context
        })

    return results
```

### Static Analysis Checks

| Issue | Detection | Recommended Action |
|-------|-----------|-------------------|
| No examples in prompt | Regex match for "example" | Warn: likely to be inconsistent |
| Ambiguous language | "may", "might", "possibly" overuse | Suggest more definitive language |
| Missing output schema | No JSON structure specified | Warn: output may be unparseable |
| Conflicting instructions | "Be strict" + "Capture everything" | Flag contradiction |
| Too long (>2000 tokens) | Token count | Warn: may timeout or be inconsistent |

### Validation Workflow

1. User creates/edits rule → runs static analysis
2. Warnings shown inline with suggestions
3. User runs test against sample inbox
4. Results show: triggered count, false positive rate (based on user-labeled test set)
5. If acceptable → deploy; if not → iterate

---

## Question 11: Synthetic Data Generation — LLM vs. Hand-Crafted

**Recommendation:** Hybrid approach — hand-crafted templates for structured API data, LLM-generated for emails to capture natural language variety.

### Approach by Data Type

| Data Type | Generation Method | Rationale |
|-----------|-------------------|-----------|
| Email content | LLM-generated | Natural language variety critical for testing rule prompts |
| API payloads | Hand-crafted templates + Faker | Structure is consistent; LLM adds little value |
| Entity data (names, dates) | Faker + hand-crafted | Deterministic makes entity linking testable |

### LLM Email Generation Strategy

```python
def generate_email_batch(domain, scenario, count):
    prompts = [
        f"Generate {count} realistic emails about {scenario} in the {domain} domain.
         Include: sender, subject, body. Vary tone and detail level.
         Make some borderline (close to triggering but not quite)."
    ]

    emails = llm.generate(prompts)

    # Human review a sample to validate realism
    # Store as test data with labels
```

### Hybrid Implementation

```python
# Email: LLM-generated with domain-specific prompts
email_generator = EmailLLMGenerator(domain="legal", scenarios=legal_scenarios)

# API: Template-based with Faker
api_template = OrderNotificationTemplate(status="payment_failed")
api_generator = FakerBasedGenerator(template=api_template)

# Entity data: Faker
entity_generator = FakerGenerator(
    names=names,
    addresses=addresses,
    case_numbers=case_number_formats
)
```

### Validation

- Run generated emails through existing rules — should produce realistic trigger distribution
- Manually review a sample (10%) for realism
- For Phase 1 demo: 50-100 emails per domain is sufficient

---

## Question 12: Entity Context Injection in Prompts

**Recommendation:** Use targeted context injection — include only relevant entity data, not full history. For Phase 1, inject summary (not raw history).

### Context Strategy

| Context Scope | Tokens | When to Use |
|---------------|--------|-------------|
| Full history | 2000-5000+ | Rarely — only if explicitly needed |
| Summary | 200-500 | Default for rule evaluation |
| Minimal | 50-100 | Pre-filter stage |
| None | 0 | Noise detection rules |

### Implementation

```python
def get_entity_context(entity, context_level="summary"):
    if context_level == "minimal":
        return f"{entity.name} (ID: {entity.id})"

    elif context_level == "summary":
        return f"""
        Entity: {entity.name}
        Type: {entity.type}
        Status: {entity.status}
        Key dates: {entity.key_dates}
        Last activity: {entity.last_interaction}
        """

    elif context_level == "full":
        return serialize_full_entity_history(entity)
```

### Rule Evaluation Context

```python
# Rule evaluation prompt includes:
RULE_PROMPT = """
## Relevant Context
{entity_context_summary}

## Email Content
{email_body}
"""
```

### Tool Call Alternative (Future Phase)

For Phase 2+ consideration: Let LLM retrieve entity context via tool calls:
- "Get recent cases for client X"
- "Check if case Y has upcoming deadlines"

This is more token-efficient but adds latency and complexity. Phase 1 should use summary approach.

---

## Question 13: Model Selection for Pipeline Stages

**Recommendation:** Use tiered model selection based on capability/price tradeoffs.

### Model Recommendations by Stage

| Pipeline Stage | Requirements | Recommended Model | Est. Cost/email | Latency |
|---------------|--------------|-------------------|-----------------|---------|
| Pre-filter | Fast, yes/no | Qwen2.5-0.5B | $0.0001 | ~100ms |
| Rule evaluation | Instruction following | MiniMax-M2.5 | $0.02 | ~1-2s |
| Action generation | Writing quality | MiniMax-M2.5 | $0.03 | ~2-3s |
| Revision | Context-aware | Kimi-A3C | $0.02 | ~1-2s |

### OpenRouter Model Availability (Estimated)

Based on current OpenRouter catalog for Kimi/Qwen/MiniMax/GLM families:

| Model | Context | Capabilities | Pricing (est.) |
|-------|---------|---------------|----------------|
| **MiniMax-M2.5** | 32K | Strong reasoning, good writing | ~$0.50/1M input |
| **MiniMax-Text-01** | 32K | Fast, efficient | ~$0.20/1M input |
| **Qwen2.5-7B** | 32K | Good instruction following | ~$0.30/1M input |
| **Qwen2.5-0.5B** | 8K | Very fast, basic | ~$0.05/1M input |
| **Kimi-A3C** | 32K | Strong at analysis | ~$0.60/1M input |
| **GLM-4-9B** | 128K | Long context capable | ~$0.50/1M input |

### Pipeline Cost Estimates

| Scenario | Pre-filter | Evaluation | Action | Total/email |
|----------|------------|------------|--------|-------------|
| No pre-filter | — | $0.02 | $0.03 | $0.05 |
| With pre-filter | $0.0001 | $0.02 | $0.03 | $0.0501 |
| Pre-filter filtered (90%) | $0.0001 | $0.002 | $0.003 | $0.005 |

**Note:** These are estimates. Validate actual pricing via OpenRouter API. Cost savings depend on pre-filter catch rate.

### Fallback Models

If primary models unavailable:
- Pre-filter: GLM-4-9B (longer context, slower but available)
- Rule evaluation: Qwen2.5-7B (good instruction following)
- Action generation: GLM-4-9B (can handle longer outputs)

---

## Question 14: Batching Rule Evaluations

**Recommendation:** Keep rules separate for auditability and error isolation. Use parallel execution for performance.

### Batching Considerations

| Approach | Pros | Cons |
|----------|------|------|
| Separate prompts | Clean audit trail, isolated errors | Higher API overhead |
| Batched single prompt | Lower API cost, simpler | Single failure kills all, harder to attribute |

### Implementation: Parallel Execution

```python
async def evaluate_email_parallel(email, rules):
    # Pre-filter in parallel
    prefilter_tasks = [run_prefilter(rule, email) for rule in rules]
    prefilter_results = await asyncio.gather(*prefilter_tasks)

    # Full evaluation for passed rules (parallel)
    evaluate_tasks = []
    for rule, passed in zip(rules, prefilter_results):
        if passed:
            task = asyncio.create_task(evaluate_rule(rule, email))
            evaluate_tasks.append((rule, task))

    # Collect results as they complete
    results = {}
    for rule, task in evaluate_tasks:
        try:
            result = await asyncio.wait_for(task, timeout=5.0)
            results[rule.id] = result
        except TimeoutError:
            results[rule.id] = {"error": "timeout"}

    return results
```

### Batch Alternative (Not Recommended for Phase 1)

```python
# Only use if latency/cost problems emerge
BATCH_PROMPT = """
Evaluate each rule against the email. Return a list of results.

Rules:
1. Rule: {rule1_description}, Keywords: {rule1_keywords}
2. Rule: {rule2_description}, Keywords: {rule2_keywords}
...

Email: {email_content}

Output JSON array:
[{"rule_id": 1, "triggered": true/false, "confidence": 0.x}, ...]
"""
```

### Why Not Batching

1. **Auditability** — need to know which rule triggered, confidence, extracted context
2. **Error isolation** — if one rule prompt fails, others should still process
3. **Priority handling** — high-priority rules can short-circuit without evaluating all
4. **Token limits** — combined prompts may exceed context window for complex rules

**Exception:** Consider batching for pre-filters only — they're short, simple, and have uniform output format.

---

## Summary of Recommendations

1. **Rule prompts:** Structured output + 2-3 few-shot examples, no CoT in production
2. **Pre-filter:** Fast model (Qwen2.5-0.5B), <100 token prompt, saves ~90% cost
3. **Evaluation:** Asynchronous with progressive disclosure for UX
4. **Conflicts:** Flag for explicit user resolution
5. **Twitter polling:** Tiered approach (30s / 2min / 5min) based on keyword priority
6. **API rules:** Hybrid (deterministic threshold + LLM for semantic)
7. **Action prompts:** Structured templates with LLM freedom in content sections
8. **Revision:** Multi-turn conversation with 5-round cap
9. **Examples:** 3+ positive, 2+ negative, actively request negatives
10. **Validation:** Test rule feature + static analysis
11. **Synthetic data:** LLM for emails, templates for API payloads
12. **Entity context:** Summary approach, not full history
13. **Models:** MiniMax-M2.5 for evaluation/actions, Qwen2.5-0.5B for pre-filter
14. **Batching:** Keep separate, run parallel

---

## Questions for Other SMEs

### For IntegrationEngineer (Database/Multi-tenancy)

- **RLS policy testing:** How should we test Postgres RLS policies in CI to ensure tenant isolation? Should we use transaction-level isolation tests or dedicated RLS test fixtures?
- **Tenant context in connection pool:** When using connection poolers (like PgBouncer), how do we ensure `SET app.current_tenant` persists correctly per request without session state leaking between requests?

### For UXDesigner

- **Action item density:** How should we handle the case where 10+ rules trigger on a single email? Should there be a "collapse to priority" view or always show all triggered actions?
- **Conflict UI:** For conflicting rules with contradictory actions, what's the preferred presentation: side-by-side comparison, sequential with "choose one" prompt, or something else?

### For LegalDomain

- **Prompt accuracy:** For legal rules, how much precision is needed in extracted context (e.g., exact filing deadlines vs. "sometime next week")? Are there scenarios where ambiguous extraction is worse than no extraction?

### For SocialMediaDomain

- **Trend response timing:** For viral moments, is the 30-minute approval window realistic, or should we design a "fast track" mode that surfaces the raw LLM output with minimal formatting for faster approval?

---

## Architectural Implications for Tech Stack

Given the established stack (FastAPI + LangGraph), here are specific implementation recommendations:

1. **LangGraph Node Design:**
   - `prefilter_node`: Fast model call, returns pass/fail
   - `evaluate_node`: Full rule evaluation, returns triggered + context
   - `action_select_node`: Context-aware selection if multiple actions
   - `generate_action_node`: Action generation with domain prompts
   - `revision_node`: Multi-turn revision with history management

2. **State Management:**
   - Store evaluation results in graph state for audit
   - Track triggered rules and confidence scores per email
   - Maintain revision history as conversation turns

3. **CopilotKit Integration:**
   - Use CopilotKit's `State` for user feedback on revisions
   - HITL primitives for approve/reject/revise flow
   - Custom UI layer for revision feedback (as noted in HLRD)

4. **Latency Management:**
   - Pre-filters: Run in parallel, timeout at 500ms
   - Full evaluation: Timeout at 3s per rule, continue with available results
   - Action generation: Timeout at 8s, surface draft-in-progress indicator

This architecture respects the latency constraints while maintaining flexibility for the two-pass evaluation approach and future extensibility.