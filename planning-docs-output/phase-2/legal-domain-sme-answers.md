# Legal Domain SME Answers — Phase 2b

## Question 1: Prompt Accuracy for Extracted Context

**Source:** AI/NLP Architecture SME (Question from Phase 1 analysis)

### Question

For legal rules, how much precision is needed in extracted context (e.g., exact filing deadlines vs. "sometime next week")? Are there scenarios where ambiguous extraction is worse than no extraction?

### Answer

**Yes, ambiguous extraction is demonstrably worse than no extraction for legal domain rules.** This is a critical design principle that should be embedded in the rule evaluation prompts.

#### Why Ambiguous Extraction Is Dangerous

In legal practice, a misrepresented deadline is often more dangerous than an unknown deadline for the following reasons:

1. **False Confidence Effect:** When the system presents extracted information (a specific date, a named party, a case number), the attorney is psychologically primed to treat it as verified. They may act on that information without independently checking the source email or court records. An "unknown" result forces the attorney to check the original source — a specific but wrong date bypasses this safety check.

2. **Malpractice Risk Amplification:** If an attorney relies on a wrong date provided by an "intelligent" system and misses a deadline, the defense "I didn't check the source" becomes much harder to argue. The attorney was provided a specific date and acted on it. This creates actual malpractice exposure that doesn't exist when the system says "I don't know — please verify."

3. **Deterministic vs. Probabilistic Data in Legal Context:** Legal operations rely on确定性 (deterministic) data. The court file has a specific deadline. The case number is X. The opposing counsel is Y. Probabilistic guesses in this domain don't represent "help" — they represent potential liability.

#### Recommended Implementation

The extraction prompts should be designed with explicit instructions:

```
For date extraction:
- If the email contains a specific date (e.g., "March 15," "03/15/2024," "next Tuesday"), extract it precisely
- If the email contains a relative date without a specific calendar date (e.g., "sometime next week," "in a few days"), return "extracted_date": null with a note: "date_ambiguous": true
- If no date is mentioned, return "extracted_date": null

Never infer or calculate a specific date from ambiguous information.
```

#### Confidence Thresholds

For legal rules specifically, implement a **high-confidence threshold** for extracted context:

| Extraction Type | Minimum Confidence | Action if Below Threshold |
|----------------|-------------------|-------------------------|
| Court deadlines | 85% | Flag as "deadline detected — verify date manually" |
| Case numbers | 90% | Flag as "case reference unclear — verify" |
| Party names | 80% | Flag as "party identified — verify spelling" |
| Hearing dates | 90% | Flag as "hearing detected — verify date/time" |

The system should present items with low-confidence extractions differently in the UI — not as "Court deadline: March 15" but as "Court deadline detected — please verify date manually."

#### Scenarios Where Ambiguous Extraction Is Particularly Dangerous

1. **Court Order Emails:** An email stating "the response is due shortly" should NOT be extracted as a specific deadline. The attorney needs to open the attachment and read the actual order.

2. **Settlement Deadlines from Opposing Counsel:** An email saying "we need to resolve this by early next month" might mean March 1 or March 28 — the difference is material.

3. **Case Numbers in Correspondence:** Emails often reference case numbers informally ("the Smith case" vs. "Smith v. Jones, Case No. CV2024-012345"). Extracting the wrong case reference could mean the attorney applies the wrong deadline to the wrong case.

#### Recommended Prompt Language

Include this explicit instruction in legal domain evaluation prompts:

```
IMPORTANT: In legal contexts, it is better to return "unknown" than to guess.
A missed deadline can result in case dismissal, default judgment, or sanctions.
If you are not certain of the specific date, party name, or case reference,
return null and let the attorney verify from the original document.
```

#### Summary

- **Precise extraction required:** Exact dates, case numbers, party names — not approximations
- **Ambiguous = unknown:** "Sometime next week" triggers action but returns null for date
- **Confidence thresholds:** Legal rules need higher confidence thresholds (85-90%)
- **UI presentation:** Low-confidence extractions should be surfaced as "detected — verify manually" rather than as specific extracted values
- **Never calculate:** Do not derive specific dates from relative references (e.g., "two weeks from now" should not become a calculated date)

---

## Additional Context from Legal Domain Analysis

This answer aligns with the Phase 1 Legal Domain SME Analysis, which identified:

- **Missed filing deadlines** as the highest-consequence dropped ball in legal practice
- **Urgency classification** must be embedded within each rule's evaluation
- **False-positive tolerance** — better to surface and dismiss than miss entirely

The extraction accuracy question is particularly important because it intersects with the "false positive" philosophy: the system should surface items for review, but the surfaced items should not include potentially wrong information that creates false confidence.

---

## File Paths

- Questions source: `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-2/legal-domain-sme-questions.md`
- This answers file: `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-2/legal-domain-sme-answers.md`
- Related Phase 1 analysis: `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-1/legal-domain-sme-analysis.md`
- AI/NLP SME analysis: `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-1/ai-nlp-sme-analysis.md`
