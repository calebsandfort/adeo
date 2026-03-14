# AI/NLP SME Review — Requirements Draft

**SME Domain:** AI/NLP Architecture
**Phase:** 4 — SME Review
**Date:** 2026-03-13
**Status:** REVIEW COMPLETE

---

## 1. Overall Assessment

The requirements draft demonstrates strong AI/ML architecture decisions with well-reasoned tradeoffs. The pre-filter pattern, structured output schemas, and cost optimization strategies align with current best practices. A few areas require clarification or enhancement.

---

## 2. Accuracy Issues

### Issue 1: Reasoning Field Ambiguity (FR-1.3 vs FR-1.9)

**Location:** FR-1.3 requires `reasoning` field, but FR-1.9 states "The evaluation prompt SHALL NOT include chain-of-thought reasoning in production"

**Analysis:** These requirements conflict. FR-1.3 requires a `reasoning` string "providing brief justification for audit purposes," while FR-1.9 prohibits chain-of-thought in production.

**Recommendation:** Clarify that `reasoning` should be a concise justification (1-2 sentences) derived from the evaluation, not full CoT. Alternatively, specify that `reasoning` is optional and should be omitted if latency/cost constraints are critical.

**Proposed FR-1.9 revision:**
> The evaluation prompt SHALL NOT include chain-of-thought reasoning in production to minimize latency and token costs, except when the `reasoning` field is required for audit purposes, in which case it SHALL be limited to 1-2 sentences.

---

### Issue 2: Pre-Filter Latency Target May Be Unrealistic

**Location:** NFR-1.4

**Analysis:** Targeting sub-500ms latency for a pre-filter LLM call is aggressive, especially for remote API calls via OpenRouter. Network latency alone typically adds 100-300ms. The Qwen2.5-0.5B model is small enough, but OpenRouter routing introduces variability.

**Recommendation:** Add a note acknowledging network variability, or specify that the target applies to model inference only, with a separate network SLA of ~300ms.

**Proposed NFR-1.4 revision:**
> Pre-filter LLM calls SHALL use lightweight models (Qwen2.5-0.5B or MiniMax-Text-01) targeting sub-500ms inference latency. Network latency to OpenRouter is excluded from this target but SHOULD be monitored.

---

## 3. Completeness Gaps

### Gap 1: LLM Output Validation

**Location:** FR-1.3 (no validation specified)

**Analysis:** The requirements specify structured output schemas but do not mandate validation of LLM responses. If the LLM returns malformed JSON or misses required fields, the system has no defined behavior.

**Recommendation:** Add a new requirement:
> **FR-1.10** The system SHALL validate LLM outputs against the required schema, with retry logic (max 1 retry) for parse failures, and a fallback to log the failure and surface the item for manual review.

---

### Gap 2: Model Availability Fallback Behavior

**Location:** NFR-2.4

**Analysis:** The requirement specifies fallback models but not the behavior when all models fail.

**Recommendation:** Add:
> **NFR-2.5** When all primary and fallback models are unavailable, the system SHALL log the failure, surface the communication for manual review, and NOT block the pipeline.

---

### Gap 3: Confidence Threshold Handling

**Location:** FR-1.3, FR-7.3

**Analysis:** FR-7.3 specifies confidence thresholds for legal domain (85% for deadlines, 90% for case numbers), but FR-1.3 only specifies the confidence field exists without thresholds. No requirement specifies what happens when confidence falls below threshold.

**Recommendation:** Add a cross-cutting requirement:
> **FR-1.11** The system SHALL apply domain-specific confidence thresholds to determine if extracted context requires manual verification, with results flagged but not blocked when thresholds are not met.

---

### Gap 4: Pre-Filter Model Selection Criteria

**Location:** NFR-2.1

**Analysis:** Specifies Qwen2.5-0.5B or MiniMax-Text-01, but MiniMax-Text-01 may not be available or may have different capabilities. No criteria for choosing between them.

**Recommendation:** Add guidance:
> **NFR-2.1.1** Model selection SHALL prioritize availability and latency, defaulting to Qwen2.5-0.5B for pre-filter unless MiniMax-Text-01 demonstrates superior performance on yes/no classification benchmarks.

---

## 4. Prompt Engineering Gaps

### Gap 5: Evaluation Prompt Template

**Location:** FR-1.0

**Analysis:** The requirements specify that evaluation prompts exist (FR-1.1) and should include few-shot examples (FR-1.8), but no requirement specifies what the base prompt template should contain.

**Recommendation:** Add:
> **FR-1.12** The evaluation prompt SHALL follow a consistent template structure:
> - Role definition
> - Input data description with placeholders
> - Output schema definition
> - Few-shot examples (if applicable)
> - Any domain-specific constraints

---

### Gap 6: Prompt Injection Prevention

**Location:** Missing

**Analysis:** No requirement addresses prompt injection attacks where incoming communications contain malicious content designed to manipulate the LLM.

**Recommendation:** Add:
> **NFR-X** The system SHALL implement input sanitization for LLM prompts, stripping or escaping user-controlled content that could inject instructions. Pre-filter prompts SHALL be evaluated for injection resistance.

---

### Gap 7: Action Generation Prompt Template

**Location:** FR-2.3

**Analysis:** Specifies "structured templates with LLM creative freedom" but no template structure.

**Recommendation:** Add:
> **FR-2.3.1** Action generation prompts SHALL follow the template:
> - Context summary (entity data, extracted info)
> - Tone/style guidelines from domain preset or brand voice
> - Required sections with format guidance
> - Constraints (word count, platform limits)

---

## 5. Conflicts with Best Practices

### Conflict 1: Sentiment Integration vs. Latency Tradeoff

**Location:** FR-2.8, Synthesis Notes line 409

**Analysis:** The synthesis notes indicate this was "resolved based on token/latency tradeoff analysis." However, integrating sentiment into the generation prompt adds complexity to the prompt and may reduce generation quality. This is a reasonable tradeoff, but the requirements should acknowledge it.

**Recommendation:** Add a note in Synthesis Notes acknowledging the tradeoff:
> Sentiment integration in generation prompt was chosen over separate calls for cost/latency, accepting slight prompt complexity increase. Quality should be monitored in production.

---

### Conflict 2: Brand Voice Profile Token Cap

**Location:** NFR-3.3

**Analysis:** Caps brand voice injection at 200-400 tokens, but FR-2.3 requires "domain-specific tone guidelines" AND FR-8.4 specifies five dimensions (Formality, Humor, Personality, Expertise, Empathy). Five dimensions may not fit in 200 tokens with meaningful detail.

**Recommendation:** Either:
1. Increase cap to 300-500 tokens, OR
2. Clarify that brand voice profiles use a structured scoring approach (e.g., "Formality: 7/10") rather than natural language descriptions

---

## 6. Minor Issues

### Minor 1: Timeout Handling Not Specified

**Location:** FR-2.6

**Analysis:** Specifies "timeout handling" but no specific timeout values. Combined with NFR-1.2 (10-second target), timeouts should be defined.

**Recommendation:** Add:
> **FR-2.6.1** Action generation timeouts SHALL default to 15 seconds, with failed generations surfaced as "tap to retry" within 20 seconds of initiation.

---

### Minor 2: Pre-Filter Bypass for High-Priority Rules

**Location:** FR-1.4

**Analysis:** Short-circuit behavior allows high-priority rules to halt lower-priority evaluations, but it's unclear if high-priority rules can bypass pre-filters entirely.

**Recommendation:** Clarify:
> **FR-1.4.1** High-priority rules MAY bypass pre-filter stages, executing full evaluation directly, when configured to do so by the administrator.

---

## 7. Positive Findings

The following requirements reflect strong AI/ML best practices:

1. **FR-1.8** — Few-shot examples requirement ensures consistency without excessive tokens
2. **FR-1.9** — CoT prohibition in production is a solid cost/latency optimization
3. **FR-2.8** — Integrated sentiment (despite tradeoff concerns, it's pragmatic for Phase 1)
4. **NFR-3.1** — Pre-filter architecture targeting 80-90% cost reduction is well-reasoned
5. **NFR-3.2-3.4** — Token caps for context injection show careful token budgeting
6. **FR-5.3** — Static analysis for prompt validation is an excellent quality gate
7. **FR-3.4-3.5** — Multi-turn revision with original context preservation prevents degradation

---

## 8. Summary of Recommended Changes

| Priority | Requirement | Change Type | Description |
|----------|-------------|-------------|-------------|
| HIGH | FR-1.9 | Clarification | Distinguish audit reasoning from CoT |
| HIGH | NFR-1.4 | Refinement | Acknowledge network latency separately |
| HIGH | FR-1.10 | New | Add LLM output validation requirement |
| MEDIUM | NFR-2.5 | New | Define behavior when all models fail |
| MEDIUM | FR-1.11 | New | Add confidence threshold handling |
| MEDIUM | NFR-X | New | Add prompt injection prevention |
| LOW | NFR-3.3 | Refinement | Clarify brand voice format (scoring vs. prose) |
| LOW | FR-2.6.1 | New | Define timeout values |

---

## 9. Questions for Other SMEs

**For Integration Engineer SME:**
- How does the timeout handling in FR-2.6 interact with the connection pooling configuration in FR-6.5? Should there be a relationship between PgBouncer timeout and LLM timeout?

**For UX Designer SME:**
- For "tap to retry" actions (FR-2.6), should there be a visual indicator showing previous failure attempts?

**For Security SME:**
- Does the RLS approach (FR-6.3-6.4) adequately protect against timing-based side channels where an attacker could infer tenant data exists based on response timing?

---

*Review completed. Requirements are substantially sound; the recommended changes above would improve robustness and reduce implementation ambiguity.*
