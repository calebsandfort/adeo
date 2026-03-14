# Legal Domain SME Review — Requirements Draft

**Phase:** 4 — SME Review
**Date:** 2026-03-13
**Reviewer:** Legal Domain SME
**Output:** `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-4/legal-domain-sme-review.md`

---

## Executive Summary

The requirements draft captures the core needs of solo/small firm legal practice reasonably well. The legal domain preset includes appropriate rule categories, urgency classifications, and action types. However, there are several significant gaps, some ethical concerns, and areas where the requirements should better reflect the realities of legal practice, particularly around malpractice risk and professional obligations.

---

## 1. Accuracy Assessment

### 1.1 Rules and Detection Scenarios (FR-7.1)

**Strengths:**
- Cancelled/rescheduled hearing detection — essential for solo practitioners who manage their own calendars
- Deadline notification (explicit and implied) — covers the primary malpractice risk area
- Client inquiry requiring response — addresses client communication obligations
- Opposing counsel action items — critical for litigation practice
- Court order/filing notification — high-stakes communications

**Issues Identified:**

1. **Missing: Retainer request/.payment urgency** — Many solo practitioners operate on retainers. Emails about running low on retainer or requests for additional funds are time-sensitive and impact case continuity.

2. **Missing: Conflict check flagging** — When a new party or opposing counsel is mentioned, the system should flag for conflict check rather than just noting it.

3. **Missing: Discovery deadline tracking** — Discovery requests (interrogatories, document requests, admissions) have specific response deadlines that differ from court-imposed deadlines. These are a common source of missed deadlines.

4. **Missing: Filed-but-not-served documents** — When opposing counsel files a motion but certifies service hasn't occurred yet, this creates a follow-up task that often gets lost.

**Recommendation:** Add FR-7.1.1 through FR-7.1.5 to cover these gaps.

---

### 1.2 Urgency Classification (FR-7.2)

**Assessment:** Generally accurate but with some overlap concerns.

- **Critical** correctly identifies court-ordered deadlines, statute of limitations, emergency hearings — these are genuinely the highest-risk items
- **High** appropriately covers scheduling changes and court orders
- **Medium** for routine client updates is reasonable
- **Low** for newsletters is appropriate

**Issue:** The line between "Critical" and "High" for court orders is unclear. A court order setting a deadline is "Critical" (the deadline itself), but a court order requiring action within a specified time is also "Critical." The current framing could cause users to under-prioritize some court orders.

**Recommendation:** Revise FR-7.2 to explicitly state that any court order containing a deadline or requiring action is "Critical" regardless of whether it is labeled a "court order" or "scheduling change."

---

### 1.3 Confidence Thresholds (FR-7.3)

**Assessment:** Appropriately conservative.

- 85% for court deadlines — acceptable but may want to be more conservative given malpractice exposure
- 90% for case numbers — appropriate; incorrect case numbers make tracking impossible
- 80% for party names — reasonable floor
- 90% for hearing dates — appropriate; wrong dates are catastrophic

**Additional Consideration:** The system should also extract the **deadline source** (which court, which judge, which document) with similar confidence requirements. A deadline without a clear source is nearly useless for an attorney to act upon.

---

### 1.4 Actions (FR-7.5)

**Strengths:**
- Client status update (conversational) — appropriate for client relations
- Rescheduling request (formal court format) — correct for procedural motions
- Opposing counsel response (professional, minimal) — appropriate for advocacy
- Internal note (factual, complete) — good for file documentation
- Flag for urgent review — appropriate for items requiring human judgment

**Gaps:**

1. **Missing: Calendar entry/deadline task** — The system should generate a calendar entry or task with the deadline rather than just a draft response. For solo practitioners, capturing the deadline as a task is often more valuable than a draft letter.

2. **Missing: Conflict check reminder** — When new parties are mentioned, generate a conflict check reminder action.

3. **Missing: Out-of-office/availability notification** — If a hearing is cancelled, generate a potential out-of-office update for the client.

**Recommendation:** Add these action types to FR-7.5.

---

## 2. Completeness Assessment

### 2.1 Legal Communication Tone Specifications

The requirements mention "domain-specific tone guidelines" in FR-2.3 but do not detail what these should be for legal communications. The legal preset should explicitly specify:

1. **Client communications:**
   - Clear, non-technical language when possible
   - Reassuring but honest about risks
   - Never promise outcomes

2. **Court filings:**
   - Formal, citation-compliant
   - Uses proper legal terminology

3. **Opposing counsel:**
   - Professional but not cordial
   - Avoids admissions
   - Minimal unnecessary communication

4. **Internal notes:**
   - Factual, chronological
   - Includes document references
   - Avoids speculation

**Recommendation:** Add FR-7.7 to specify tone guidelines for each action type in the legal preset.

---

### 2.2 Missing Legal Scenarios

| Scenario | Risk Level | Current Coverage |
|----------|------------|------------------|
| Statute of limitations expiration | Critical | Covered (implicitly in FR-7.2) |
| Court holiday/weekend deadline handling | Critical | NOT COVERED |
| Time-sensitive settlement offers | High | NOT COVERED |
| Bar complaint/ethics inquiries | Critical | NOT COVERED |
| Client documents received for review | Medium | NOT COVERED |
| Limine motions/response deadlines | High | NOT COVERED |
| Appellate deadline notifications | Critical | NOT COVERED |
| Mediation/statement deadline | High | NOT COVERED |

**Key Gap:** The system should detect deadlines and provide a countdown (FR-4.7), but it should also flag if the deadline falls on a court holiday or the deadline calculation involves complex rules (e.g., "3 business days from service").

**Recommendation:** Add FR-7.8 to address deadline edge cases including court holidays and business day calculations.

---

### 2.3 Practice Area Considerations

The requirements treat "legal" as a monolithic domain, but practice area matters significantly:

- **Family law:** High emotional content, custody concerns, rapid scheduling changes
- **Criminal defense:** Jail calls, bond modifications, speedy trial rights
- **Personal injury:** Medical records requests, insurance deadlines, liens
- **Estate planning:** Document execution requirements, notary needs
- **Corporate:** Transaction deadlines, filing requirements

**Recommendation:** While the Phase 1 preset can remain general, FR-7.9 should acknowledge that future practice-area-specific sub-presets will be needed.

---

## 3. Conflicts and Ethical Concerns

### 3.1 AI Disclaimer (FR-2.7)

**Requirement:** Append a configurable disclaimer: "This draft was generated with AI assistance and requires attorney review before use"

**Ethical Analysis:**

This requirement touches on Model Rule 1.1 (Competence) and Model Rule 1.4 (Communication). The current framing suggests the disclaimer protects the attorney, but:

1. **Professional responsibility:** A lawyer cannot delegate their professional judgment to AI. The disclaimer does not relieve the attorney of the duty to review for accuracy.

2. **Client communication ethics:** Using AI-assisted drafting without disclosure may be required under some jurisdictions. The disclaimer may be ethically necessary in some cases and insufficient in others.

3. **Competence requirement:** Under Model Rule 1.1, using AI tools requires understanding their limitations. The system should educate users on these limitations, not just append a disclaimer.

**Recommendation:**
- FR-2.7 should be expanded to include guidance on when the disclaimer is ethically required vs. optional
- The system should provide a warning that the disclaimer does not substitute for attorney review
- Consider adding a checkbox: "I have reviewed this draft for accuracy" as part of the approval workflow

---

### 3.2 Confidentiality Considerations (Cross-cutting)

**Concern:** The rules engine processes sensitive legal communications. While FR-6 addresses tenant isolation, there are additional confidentiality considerations:

1. **Prompt data retention:** If evaluation prompts and results are logged, this creates sensitive data that should have retention limits

2. **Training data concerns:** If the LLM provider uses API calls for training, this creates a potential ethics violation

3. **Client matter naming:** Entity context includes client names and case information. This should not appear in any logs that the attorney cannot access

**Recommendation:** Add NFR-4.5 addressing data retention and confidentiality requirements for legal domain specifically.

---

### 3.3 Conflict Detection vs. Generation

**Issue:** The requirements mention detecting "conflicting actions" (FR-4.6) but for legal, conflicts have specific meaning. A response to opposing counsel that contradicts a simultaneously generated client update would be a real conflict, not just a content conflict.

**Recommendation:** For legal domain, FR-4.6 should explicitly address the case where actions to different recipients would create inconsistency.

---

## 4. Gaps Summary

### 4.1 High-Priority Gaps

| Gap | Impact | Recommendation |
|-----|--------|----------------|
| Court holiday/weekend deadline detection | Missed filings | Add FR-7.8 |
| Business day calculation | Incorrect deadline dates | Add FR-7.8 |
| Calendar entry generation | Missing actionable tasks | Add to FR-7.5 |
| AI disclaimer guidance | Ethical compliance | Expand FR-2.7 |
| Practice area variation | Relevance to specific practices | Add FR-7.9 |

### 4.2 Medium-Priority Gaps

| Gap | Impact | Recommendation |
|-----|--------|----------------|
| Conflict check flagging | Ethics violations | Add to FR-7.1 |
| Discovery deadlines | Missed discovery responses | Add to FR-7.1 |
| Legal tone specifications | Inappropriate communications | Add FR-7.7 |
| Retention request/payment | Case continuity | Add to FR-7.1 |

---

## 5. Positive Aspects to Retain

The following aspects of the requirements are well-designed and should be preserved:

1. **"Return unknown rather than guess" (FR-7.4)** — Excellent principle that should be extended to all extracted entities, not just dates

2. **Urgency extension for legal (FR-4.2)** — The pulsing dot for critical items appropriately signals emergency

3. **Confidence thresholds (FR-7.3)** — Conservative approach to extraction is correct for legal

4. **Multiple action types (FR-7.5)** — Good variety covering different communication needs

5. **Jurisdiction templates (FR-7.6)** — Flexibility to handle different courts is essential

---

## 6. Specific Requirement Changes

### Suggested New Requirements

**FR-7.1.1:** The legal preset SHALL detect retainer/low-balance inquiries and flag as high-urgency when the matter is active.

**FR-7.1.2:** The legal preset SHALL detect discovery requests and extract response deadlines with court-specific calculation rules.

**FR-7.5.6:** The legal preset SHALL generate calendar entries or task items with deadlines, not just draft communications.

**FR-7.7:** The legal preset SHALL specify tone guidelines for each action type:
- Client updates: clear, accessible, honest about risks
- Court filings: formal, properly cited, procedural
- Opposing counsel: professional, minimal, no admissions
- Internal notes: factual, chronological, document-referenced

**FR-7.8:** The legal preset SHALL handle deadline edge cases:
- Court holiday detection
- Weekend deadline adjustment
- Business day calculations
- Service-date-based deadline computation
- Cross-jurisdiction variation awareness

**FR-7.9:** The legal preset SHOULD support practice-area-specific sub-configurations in future phases.

### Suggested Modifications

**FR-7.2:** Revise to explicitly state any court order containing a deadline = Critical, regardless of label.

**FR-2.7:** Expand to include ethical guidance on disclaimer use and add a "reviewed" checkbox in approval workflow.

**FR-4.7:** Add that legal deadline countdown should indicate if deadline falls on weekend/holiday.

---

## 7. Conclusion

The requirements draft provides a solid foundation for the legal domain preset. The core detection scenarios, urgency classification, and action types are appropriate for solo/small firm practice. However, the gaps identified above — particularly around deadline edge cases, AI disclaimer ethics, and practice area variation — should be addressed before implementation.

The most critical items to address are:
1. Court holiday and business day deadline calculations (malpractice risk)
2. AI disclaimer ethical guidance (professional responsibility)
3. Calendar entry generation (practical utility for solo practitioners)

With these additions, the legal domain preset will be realistic enough for a practicing attorney to recognize and valuable enough to materially assist with practice management.

---

*Review prepared by Legal Domain SME*
*Status: Ready for PM Finalization (Phase 4)*
