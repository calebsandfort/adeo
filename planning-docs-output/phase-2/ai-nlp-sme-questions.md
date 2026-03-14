# Cross-SME Questions: AI/NLP Architecture

## Questions from Legal Domain SME

### Context
Legal Domain SME provides domain expertise in legal practice workflows, communication patterns, and risk assessment for solo practitioners and small law firms. Their analysis focuses on deadline detection, urgency classification, and tone conventions for legal communications.

### Question 1: Prompt Accuracy for Legal Context Extraction
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

For legal rules, how much precision is needed in extracted context (e.g., exact filing deadlines vs. "sometime next week")? Are there scenarios where ambiguous extraction is worse than no extraction?

**Background:** In legal practice, missing a specific date entirely is less dangerous than having a wrong date — the attorney will check the actual deadline. However, if the system suggests an incorrect date with confidence, it could lead to missed deadlines. Should extraction prompts be designed to return "unknown" rather than guess?

### Question 2: Legal Disclaimer Generation
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

Should the action generation for legal communications include a disclaimer that the draft is AI-generated and requires attorney review? Is this a prompt-level instruction or a post-generation append?

### Question 3: Extraction Confidence for Dates
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

For court-ordered deadlines, the date extraction is critical. Should the system have a confidence threshold below which it surfaces the item differently (e.g., "Court deadline detected — please verify date manually") rather than presenting an extracted date with low confidence?

---

## Questions from Social Media Domain SME

### Context
Social Media Domain SME provides expertise in social media management workflows, trend monitoring, brand voice preservation, and response timing for solo managers and small agencies.

### Question 4: Model Selection for Fast Drafts
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

Given the requirement for fast drafts (3-5 seconds), which of the available model families (Kimi, Qwen, MiniMax, GLM) offers the best speed/quality tradeoff for generating short-form social media responses? Should we use different models for Fast vs. Enhanced modes?

### Question 5: Sentiment Detection Accuracy
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

How reliably can the available models detect sentiment in short, informal social media text (slang, emojis, sarcasm)? Should sentiment detection be a separate model call or integrated into the generation prompt?

### Question 6: Token Budget Optimization
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

For brand voice injection, what's the maximum context size we should dedicate to voice profile examples before it degrades generation quality vs. simply repeating the same guidance?

---

## Questions from E-commerce Domain SME

### Context
E-commerce Domain SME provides expertise in e-commerce operations, inventory management, marketplace policies, and customer communication for small sellers on Shopify/Amazon.

### Question 7: Issue Type Extraction
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

How should the system handle extracting issue type (sizing vs. damage vs. quality) from customer communications? Should this be a separate classification step in the rule evaluation pipeline, or integrated into the evaluation prompt?

### Question 8: Product-Specific Response Generation
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

For product-specific response generation, should we use few-shot examples in the action generation prompt showing correct language for each product category, or is a category label + guidelines sufficient?

### Question 9: Token Optimization for Product Context
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

How can we optimize token usage when injecting product context into action generation prompts without losing the specificity that makes responses useful?

---

## Questions from UX Designer SME

### Context
UX Designer SME provides expertise in dashboard design, information hierarchy, and human-in-the-loop interaction patterns.

### Question 10: Rule Evaluation Latency
**Source:** UX Designer SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

How long does rule evaluation actually take in practice with the specified model families (Kimi, Qwen, MiniMax, GLM)? The 5-second requirement is a target, but what's realistic? This affects whether we can show live evaluation in the UI or must cache results.

### Question 11: Parallel vs. Sequential Action Generation
**Source:** UX Designer SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

When a rule triggers multiple actions, does the system evaluate them in parallel or sequentially? Action generation latency directly affects whether opening a "Review Actions" sheet feels instant or takes several seconds.

---

## Questions from Integration Engineer SME

### Context
Integration Engineer SME provides expertise in database architecture, multi-tenancy, and system integration patterns.

### Question 12: Tenant Context in Rule Evaluation
**Source:** Integration Engineer SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

The HLRD mentions pre-filter LLM calls and full evaluation prompts. How should the tenant context be passed to the rule evaluation? Should each tenant have isolated rule configurations stored in their schema, or should rules be evaluated within the tenant's database context?

### Question 13: Entity Context for Action Generation
**Source:** Integration Engineer SME Analysis (Section: Questions for Other SMEs > For AI/NLP Architecture SME)

When generating actions (e.g., drafting emails), the LLM needs access to entity context (client names, case numbers). Should this context be retrieved via the tenant-scoped database connection we've designed above, ensuring complete isolation during entity lookups?
