# UX Designer SME Review — Requirements Draft

**Phase:** 4 — SME Review
**Date:** 2026-03-13
**Reviewer:** UX Designer SME

---

## Executive Summary

The requirements draft provides solid foundational UI/UX specifications for the Adeo dashboard and approval workflows. However, there are several gaps in interaction states, edge cases, and accessibility considerations that should be addressed before finalization. The core dashboard layout and HITL approval flow follow established professional tool patterns, but missing error/empty states and accessibility requirements could significantly impact real-world usability.

---

## Detailed Review

### 1. Accuracy of UI/UX Requirements

**Strong Points:**

- **FR-4.1** (urgency-first grouping) correctly optimizes for triage workflow — professionals scan by urgency before context
- **FR-4.3** (card content specification) is appropriately detailed with truncated subject lines (60 chars) and relative timestamps in monospace
- **FR-4.5** (side drawer Sheet component) preserves dashboard context — this is the correct pattern for HITL review
- **FR-4.9** (filtering) covers essential filter dimensions; adding "date range" would strengthen this
- **NFR-5.1** (60-second session target) is realistic for professional tool usage patterns

**Issues Found:**

- **FR-4.2**: The "amber/blue accent system" needs clarification on exact color values. If using ShadCN/ui, specify whether these map to existing semantic colors (amber-500, blue-500) or require custom tokens.
- **FR-4.8**: Character count display ("185/200 chars") should specify platform limits per domain. Different limits apply to Twitter (280), LinkedIn (3000), Instagram (2200), etc.

---

### 2. Completeness of Dashboard and Interaction Specifications

**Missing Interaction States:**

| Gap | Recommendation |
|-----|----------------|
| **Empty states** | Add FR-4.xx requirements for: (1) no pending items, (2) no rules configured, (3) no entities in database. Each needs distinct messaging and CTA. |
| **Error states** | Add FR-4.xx for: (1) LLM evaluation failure, (2) action generation timeout, (3) network error. Specify error card design and "retry" CTA placement. |
| **Loading states** | NFR-1.5 mentions skeleton loaders but doesn't specify WHERE they appear. Add to: dashboard initial load, drawer open (context loading), revision submission. |
| **Success feedback** | After approval execution, add toast notification confirming action was sent. Specify duration and dismissal behavior. |

**Missing User Flows:**

- **Undo functionality**: After "Reject" or "Approve," can user undo within a time window? Not specified. Recommend adding 30-second undo window with clear CTA.
- **Bulk operations**: For users with 10+ pending items, no bulk approve/reject capability. Consider FR-4.xx for multi-select + bulk action.
- **Keyboard shortcuts**: Power user feature not addressed. Recommend: `j/k` navigation, `a` approve, `r` reject, `e` edit, `Enter` open drawer.
- **Keyboard navigation**: Not specified — users must be able to tab through action cards and drawer actions.

---

### 3. Gaps in User Experience Flows

**Rule Creation Flow Gaps (FR-5.x):**

1. **Onboarding**: New users have zero rules. FR-5.x doesn't specify initial onboarding UI — wizard? template gallery? This is critical for time-to-value.

2. **Validation feedback**: FR-5.3 validates prompts but doesn't specify HOW feedback is presented. Inline validation in prompt editor? Modal warning? Error list?

3. **Dry run results**: FR-5.4 specifies dry run but not HOW results display. Recommended: expand inline below rule config with pass/fail badges per test email.

4. **Save vs. Publish**: Dry run mode implies rules can exist in "draft" state. Need FR-5.x for: draft rules, active rules, disabled rules — and UI to manage state.

**Approval Flow Gaps (FR-3.x):**

1. **Conflict resolution**: FR-4.6 mentions conflict detection but not HOW user resolves. Recommend explicit "choose A or B" UI, not side-by-side without guidance.

2. **Revision degradation warning**: FR-3.5 mentions flagging revisions that lose key information but not HOW. Recommend inline warning banner: "This revision appears to omit: [specific info]."

3. **5-round cap feedback**: FR-3.3 specifies cap but not exact UI. Recommend: modal at round 4 saying "One more revision allowed" with clear "Edit directly" CTA at round 5.

**Testing Mode Gaps (FR-4.10):**

1. **Quick Test results display**: Not specified. Recommend: trigger results shown inline with rule match/no-match badges, confidence scores visible.

2. **Historical Replay navigation**: How does user move between emails? Recommend: prev/next buttons, keyboard arrows, or list view with expandable details.

---

### 4. Conflicts with UX Best Practices

**Potential Conflicts:**

1. **FR-4.3 — Single CTA ("Review Actions")**: This is correct for scan-speed optimization. However, users may want quick-approve for obvious items. Consider FR-4.xx for "Quick Approve" on hover/expand for low-urgency items.

2. **FR-2.4 — Three draft generation modes**: From a UX perspective, auto-selecting draft mode (FR-2.5) removes user control. Consider: allow user to override mode selection in settings, or show mode badge on generated draft so user knows what quality to expect.

3. **NFR-5.2 — Mobile review-only**: This is correct given Phase 1 scope, but the cutoff line should be explicit. Recommend: add NFR-5.x stating "Mobile approval capability deferred to Phase 2" to prevent scope creep.

4. **FR-4.7 — Countdown display for legal**: "3 days remaining" is good. However, legal domain should also show: (1) weekday/weekend distinction for court deadlines, (2) "today" urgency escalation, (3) business hours context. This may require FR-7.x update.

---

### 5. Accessibility Considerations

**Missing Requirements:**

- **Color contrast**: Urgency colors (amber/blue) must meet WCAG AA (4.5:1 for text). Specify this in NFR-5.x.
- **Screen reader support**: Action cards need proper ARIA labels. Recommend NFR-5.x: "All interactive elements must have accessible names."
- **Focus management**: Drawer open should trap focus; close should return focus to trigger element. Recommend: specify in FR-4.5 implementation requirements.
- **Reduced motion**: For animation (pulsing dot, drawer slide), respect `prefers-reduced-motion`. Recommend: add to NFR-5.x.

---

## Summary of Recommended Additions

### New Functional Requirements

1. **FR-4.xx — Empty states**: Dashboard shows contextual empty state with CTA when no pending items.
2. **FR-4.xx — Error handling UI**: Error cards with retry CTA when evaluation/generation fails.
3. **FR-4.xx — Toast notifications**: Success feedback after approved action executes.
4. **FR-5.xx — Rule lifecycle states**: Draft, active, disabled — with UI to transition between them.
5. **FR-5.xx — Onboarding flow**: Initial rule setup wizard or template gallery for new users.
6. **FR-3.xx — Undo window**: 30-second undo after reject/approve actions.

### New Non-Functional Requirements

1. **NFR-5.x — Accessibility standards**: WCAG AA compliance, focus management, reduced motion.
2. **NFR-5.x — Keyboard navigation**: Tab order specification for all interactive elements.

---

## Questions for Other SMEs

**For Product Manager:**
- Should the mobile review-only Phase 1 decision be documented as explicit scope boundary?
- What is the expected onboarding flow for new users — guided setup or self-service template selection?

**For AI/NLP Architecture SME:**
- When "Quick Draft" mode is auto-selected (FR-2.5), can we surface a confidence indicator to the user so they know whether to scrutinize more closely?

**For Integration Engineer SME:**
- For error states (LLM failure, timeout), should failed items be retryable individually or is bulk retry supported?

---

## Conclusion

The requirements draft is strong on core dashboard layout and HITL approval flow. The recommended additions focus on completing the interaction state picture (empty, loading, error, success), accessibility compliance, and clarification of edge cases. Addressing these gaps will ensure the implemented UI feels polished and professional rather than functional but incomplete.

**Recommendation:** Approve with conditions — address empty/error states and accessibility requirements before implementation.
