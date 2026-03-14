# Social Media Domain SME Review — Phase 4

**Review of:** `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-3/requirements-draft.md`
**Date:** 2026-03-13
**SME Domain:** Social Media Management

---

## Executive Summary

The requirements draft captures the core social media management use case well, with appropriate attention to the solo-manager and small-agency context. However, there are significant gaps in platform-specific conventions, escalation workflows, and crisis detection that should be addressed before finalization. Several requirements conflict with real-world social media management best practices.

---

## 1. Accuracy Assessment

### 1.1 Requirements Correctly Represented

**FR-8.1 (Pre-configured Rules)** — Mostly accurate. The six rule types cover the primary detection scenarios:
- Trending topic keyword match
- Brand mention spike detection
- Negative sentiment mention
- Influencer/high-follower engagement
- Review site alert
- Client campaign request

**FR-8.4 (Brand Voice Profiles)** — The five dimensions (Formality, Humor, Personality, Expertise, Empathy) align with how social media managers actually describe brand voice. This is a strong approach.

**FR-8.5 (Sentiment-Aware Tone Mapping)** — The tone-by-sentiment framework is sound:
- Positive: Match or exceed enthusiasm
- Negative: Empathize first, offer solution
- Neutral: Helpful and concise
- Sarcastic: Acknowledge concern without being defensive

### 1.2 Requirements Requiring Correction

**FR-8.2 (Tiered Polling Intervals)** — **This requirement is technically unrealistic for Twitter/X.** The X API does not support configurable polling intervals in the manner described. API access is rate-limited, not interval-configurable.

**Recommendation:** Reframe this as a simulation parameter for demo purposes, not an API capability. The system simulates Twitter/X behavior, so this is acceptable for Phase 1 demonstration, but the requirement should explicitly acknowledge it's simulating API behavior.

**FR-4.8 (Review Response Character Limit)** — The 100-character preview with "185/200 chars" indicator is confusing and unrealistic. Review platform responses have varying limits:
- Google: ~400 characters
- Yelp: ~5,000 characters
- Trustpilot: ~1,000 characters
- G2: ~500 characters

**Recommendation:** Make character limits configurable per review platform, not hardcoded.

---

## 2. Completeness Assessment

### 2.1 Missing Critical Scenarios

**A. Crisis/Viral Detection**
The requirements do not distinguish between a normal negative mention and a viral crisis situation. This is a critical gap for social media managers.

**Gap:** No rule type for "crisis signal detection" — rapid velocity increase in negative mentions, potentially from a single viral post.

**Proposed addition:**
- FR-8.X: The system SHALL detect crisis signals when negative mention velocity exceeds 5x baseline within a 15-minute window.

**B. Platform-Specific Content Conventions**
Social media content varies significantly by platform, but the requirements treat it generically.

**Gap:** No mention of:
- Twitter/X thread formatting requirements
- Instagram hashtag placement conventions
- LinkedIn professional tone requirements
- TikTok caption conventions (limited to ~2,200 characters)

**Proposed addition:**
- FR-8.Y: The system SHALL inject platform-specific formatting conventions into generation prompts, including: hashtag placement rules, @mention conventions, thread formatting for Twitter/X, and character limit awareness.

**C. Review Platform Response Conventions**
FR-8.1 includes "Review site alert" but FR-8.6 doesn't specify review response actions with platform-specific formatting.

**Gap:** Each review platform has different response conventions:
- Google: Public response, concise, thank-then-address pattern
- Yelp: Semi-formal, can be longer, never argumentative
- Trustpilot: Professional, GDPR-aware (no personal data)
- G2: B2B-focused, technical tone appropriate

**Proposed addition:**
- FR-8.Z: The system SHALL apply platform-specific response templates for review sites, including character limits, tone guidelines, and GDPR compliance for public responses.

**D. Escalation Workflow**
FR-8.6 mentions "Draft escalation brief" but there's no definition of what triggers escalation.

**Gap:** No clear criteria for when an action should escalate vs. be handled directly.

**Proposed addition:**
- FR-8.W: The system SHALL escalate to client notification when: (1) mention sentiment is strongly negative AND account has >10K followers, (2) velocity spike exceeds 10x baseline, (3) legal/risk keywords detected, or (4) influencer (/>50K followers) posts negative brand content.

### 2.2 Missing Agency Workflow Considerations

**Multi-Account Context:** A small agency managing 10+ brands needs to know *which* brand triggered the alert.

**Gap:** No requirement for brand identification in the dashboard card. The entity context includes brand name, but there's no explicit requirement that this be prominent.

**Proposed addition:**
- FR-4.3.X: Each action item card SHALL display the associated brand name prominently when the user manages multiple brand accounts.

---

## 3. Conflicts with Best Practices

### 3.1 Timing Conflicts

**FR-2.5 (Auto-select Draft Mode)** — The auto-selection logic prioritizes "Fast Draft" for high-velocity trends and high-follower accounts. While this is directionally correct, the assumption that faster is always better conflicts with brand safety best practices.

**Conflict:** Fast Draft (3-5 seconds) may not allow sufficient context review for high-follower accounts where a misstep has greater impact.

**Recommendation:** Add a brand safety check that requires Standard Draft minimum for accounts over a configurable follower threshold, regardless of trend velocity.

### 3.2 Tone Convention Conflicts

**FR-8.5 (Sarcastic Tone Handling)** — The current guidance says "acknowledge concern without being defensive." This is good advice, but the requirements don't address a common scenario: **obvious troll/hostile mentions**.

**Conflict:** Engaging with obvious trolls can amplify the negative. The best practice is often to ignore, not respond.

**Proposed addition:**
- FR-8.XA: The system SHALL detect troll/hostile patterns (no specific grievance, inflammatory language, account with history of hostile behavior) and recommend "No response needed" as an action option.

### 3.3 Review Response Conflicts

**FR-4.8 Character Limit Display** — The example "185/200 chars" implies a 200-character limit, which doesn't match any major review platform. This creates user confusion.

**Conflict:** Displaying incorrect character limits could lead to truncated responses or user distrust.

---

## 4. Feasibility Assessment

### 4.1 Realistic for Solo Managers

The requirements are appropriately scoped for solo managers managing 3-5 accounts:

- **FR-2.4 (Draft Generation Speed):** 3-5 seconds for Fast Draft is realistic with current LLM latency.
- **FR-8.1 (Rule Types):** All six rule types represent real pain points for solo managers.
- **FR-5.1-5.3 (Custom Rule Creation):** The three creation paths (manual, LLM-assisted, example-based) are appropriate for non-technical users.

### 4.2 Realistic for Small Agencies

The requirements appropriately address small agency needs:

- **FR-6.1 (Multi-tenancy):** Agency workflow requires isolation between client accounts.
- **FR-4.9 (Filtering):** Agency users need to filter by brand/entity.
- **FR-2.7 (Parallel Generation):** Agencies handling multiple accounts need non-blocking action generation.

### 4.3 Potential Overreach

**FR-8.2 (30-second polling):** While acceptable for simulation, this sets unrealistic expectations for real Twitter/X API integration in future phases.

---

## 5. Specific Recommendations

### 5.1 High-Priority Changes

| FR | Issue | Recommendation |
|----|-------|----------------|
| FR-4.8 | 100-char preview, 200-char example limit | Make configurable per platform |
| FR-8.2 | Polling interval framing | Clarify as simulation parameter only |
| FR-8.X | Missing crisis detection | Add crisis signal rule type |
| FR-8.Y | Missing platform conventions | Add platform-specific formatting injection |

### 5.2 Medium-Priority Changes

| FR | Issue | Recommendation |
|----|-------|----------------|
| FR-2.5 | Fast Draft for high-follower accounts | Add brand safety minimum |
| FR-8.XA | Missing troll/hostile detection | Add "No response" recommendation |
| FR-4.3.X | Multi-brand display | Ensure brand name prominent on cards |

### 5.3 Low-Priority Enhancements

- Add "Competitor mention" as a rule type (common monitoring scenario)
- Add "Hashtag tracking" as a rule type
- Add direct message (DM) notification as a data source type

---

## 6. Questions for Other SMEs

### For Product Manager SME:
How should the system handle conflicting guidance between auto-selected draft mode and brand safety minimums? Should user-configurable overrides be permitted?

### For AI/NLP Architecture SME:
Can sentiment detection and content generation be reliably combined in a single prompt (FR-2.8), or does this create prompt conflict that reduces accuracy? What's the recommended token allocation for combined prompts?

### For Integration Engineer SME:
For review platform APIs (Google Business Profile, Yelp, Trustpilot), what authentication mechanisms should the system support in Phase 2 for real data source integration?

---

## Summary

The social media domain requirements are **75% complete** with strong foundational elements. The core framework (brand voice profiles, sentiment-aware tone mapping, tiered keyword configuration) is sound. However, critical gaps in crisis detection, platform-specific conventions, and escalation workflows should be addressed before Phase 5 technical specification.

**Recommendation:** Accept with modifications — address the four high-priority changes before proceeding to implementation.
