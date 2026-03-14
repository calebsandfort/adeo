# Cross-SME Questions: UX Designer

## Questions from AI/NLP Architecture SME

### Context
AI/NLP Architecture SME provides expertise in LLM prompt design, model selection, latency optimization, and AI pipeline architecture.

### Question 1: Action Item Density
**Source:** AI/NLP SME Analysis (Section: Questions for Other SMEs > For UXDesigner)

How should we handle the case where 10+ rules trigger on a single email? Should there be a "collapse to priority" view or always show all triggered actions?

### Question 2: Conflict UI for Contradictory Actions
**Source:** AI/NLP SME Analysis (Section: Questions for Other SMEs > For UXDesigner)

For conflicting rules with contradictory actions, what's the preferred presentation: side-by-side comparison, sequential with "choose one" prompt, or something else?

---

## Questions from Legal Domain SME

### Context
Legal Domain SME provides domain expertise in legal practice workflows, communication patterns, and risk assessment for solo practitioners and small law firms.

### Question 3: Urgency Visualization for Legal Items
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner)

Given that legal urgency classification is critical (deadlines, hearings), how should the dashboard visually distinguish critical, high, medium, and low urgency items? The dual accent system (amber/blue) seems designed for urgency vs. routine — should legal items use a different scheme, or is the existing scheme sufficient?

### Question 4: Email Viewer for Legal Review
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner)

For attorneys reviewing triggered rules, what is the optimal way to present the full email alongside the extracted context and suggested actions? Should the email be shown in a collapsible panel, a side-by-side view, or something else? Attorneys need to see the original to verify extracted context.

### Question 5: Legal Deadline Calendar Integration
**Source:** Legal Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner)

Future phases may include calendar integration. How should we visualize detected deadlines in the dashboard — as a simple date, or with countdown ("3 days remaining") or both?

---

## Questions from Social Media Domain SME

### Context
Social Media Domain SME provides expertise in social media management workflows, trend monitoring, brand voice preservation, and response timing for solo managers and small agencies.

### Question 6: Dashboard Scan Patterns
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

When a social media manager opens the dashboard in the morning, what's the optimal information hierarchy? Should urgent items be first (amber border), or should time-based grouping show overnight activity separately?

### Question 7: Mobile Responsiveness
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

Given that managers often check mentions on phones between meetings, what's the minimum viable mobile experience? Can approval happen on mobile, or is it review-only?

### Question 8: Notification Design
**Source:** Social Media Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

Should the system send push notifications for certain triggers (negative mentions, influencer tags), or is in-app notification sufficient? What's the notification fatigue risk for a tool checked multiple times daily?

---

## Questions from E-commerce Domain SME

### Context
E-commerce Domain SME provides expertise in e-commerce operations, inventory management, marketplace policies, and customer communication for small sellers.

### Question 9: Multiple Rules on Same Email
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

When multiple rules trigger on the same email (e.g., both a negative review alert and a customer service response action), how should the dashboard present these? Should they be combined into a single action item with multiple suggested responses, or separated?

### Question 10: Review Response Preview Format
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

For review response actions specifically, what preview format would help a seller quickly approve vs. edit? Is a character count or preview truncation helpful?

### Question 11: Marketplace Context in UI
**Source:** E-commerce Domain SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

How should the system indicate marketplace-specific context (Amazon vs. Shopify) in the rule and action configuration UI without overwhelming users who only use one marketplace?

---

## Questions from Integration Engineer SME

### Context
Integration Engineer SME provides expertise in database architecture, multi-tenancy, and system integration patterns.

### Question 12: Multi-Tenant Dashboard Indicator
**Source:** Integration Engineer SME Analysis (Section: Questions for Other SMEs > For UXDesigner SME)

The dashboard shows action items from triggered rules. How should we indicate tenant context in the UI for users who might have access to multiple tenants in future phases? Should there be a tenant switcher component in the sidebar?
