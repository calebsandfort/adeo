# Cross-SME Questions: Legal Domain

## Questions from AI/NLP Architecture SME

### Context
AI/NLP Architecture SME provides expertise in LLM prompt design, model selection, latency optimization, and AI pipeline architecture.

### Question 1: Prompt Accuracy for Extracted Context
**Source:** AI/NLP SME Analysis (Section: Questions for Other SMEs > For LegalDomain)

For legal rules, how much precision is needed in extracted context (e.g., exact filing deadlines vs. "sometime next week")? Are there scenarios where ambiguous extraction is worse than no extraction?

**Background:** AI/NLP SME notes that in legal practice, missing a specific date entirely is less dangerous than having a wrong date — the attorney will check the actual deadline. However, if the system suggests an incorrect date with confidence, it could lead to missed deadlines. The recommendation is that extraction prompts should return "unknown" rather than guess.

**Reference:** This is a clarification request on the legal domain's requirements for precision in extracted dates, parties, and case references from the evaluation prompts.
