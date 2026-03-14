# UX Designer SME Answers — Phase 2b Cross-SME Consultation

This document provides answers to 12 questions routed to the UX Designer SME from other domain experts. Each answer draws on my Phase 1 UX analysis, the context from other Phase 1 SME analyses, and established UX patterns for professional dashboard tools.

---

## Questions from AI/NLP Architecture SME

### Question 1: Action Item Density — Handling 10+ Rules Triggering on a Single Email

**How should we handle the case where 10+ rules trigger on a single email? Should there be a "collapse to priority" view or always show all triggered actions?**

**Recommendation: Collapse to Priority View with Expandable Summary**

When multiple rules trigger on the same email, the dashboard should:

1. **Show a single card** for that email with:
   - Primary urgency indicator (based on highest-urgency triggered rule)
   - Badge showing "X rules triggered" (not "X actions")
   - "Review Actions" button that opens the full review sheet

2. **In the review sheet**, display all triggered rules in a list with:
   - Rule name and confidence score
   - Urgency level per rule (color-coded)
   - Whether it generated an action or just an alert
   - Collapsible detail per rule showing extracted context

3. **At the action level**, group actions by type:
   - Draft communications (emails, messages)
   - Alerts (deadline warnings, notifications)
   - Status changes

The key UX principle here: **users should never need to mentally reassemble related items**. If one email triggers a deadline alert and a client communication request, those belong together in a single flow.

**Design detail:** The card should show the highest-urgency rule in the badge area, but when expanded, all rules should be visible. This handles the scenario where an email contains both a critical deadline (amber) and routine follow-up (blue) — the card takes the amber treatment.

---

### Question 2: Conflict UI for Contradictory Actions

**For conflicting rules with contradictory actions, what's the preferred presentation: side-by-side comparison, sequential with "choose one" prompt, or something else?**

**Recommendation: Side-by-Side in the Review Sheet with Explicit Conflict Indication**

When the AI/NLP system detects contradictory actions (e.g., one rule suggests "dismiss client" while another suggests "prioritize client retention"), the presentation should be:

1. **In the action card** on the dashboard:
   - Show a warning badge: "Conflict detected"
   - Both actions visible but marked as contradictory

2. **In the review sheet**, present side-by-side:
   ```
   +---------------------------+---------------------------+
   | Option A: Dismiss         | Option B: Retain         |
   |---------------------------|---------------------------|
   | Rule: Low Engagement     | Rule: High Value         |
   | Confidence: 91%          | Confidence: 87%          |
   | Generated Draft:         | Generated Draft:         |
   | "We regret to inform..." | "Thank you for your..."  |
   |---------------------------|---------------------------|
   | [Select This Option]     | [Select This Option]      |
   +---------------------------+---------------------------+
   ```

3. **The selection mechanism:**
   - User clicks one option to select it
   - Selected option expands to full editing view
   - Rejected option collapses to summary
   - Clear "Review Both" option available

**Why this approach:**
- Side-by-side comparison lets the user see the full picture at once
- Explicit conflict indication prevents accidental selection
- Sequential "choose one" creates unnecessary friction — the user might need to read both drafts to make a decision

---

## Questions from Legal Domain SME

### Question 3: Urgency Visualization for Legal Items

**Given that legal urgency classification is critical (deadlines, hearings), how should the dashboard visually distinguish critical, high, medium, and low urgency items? The dual accent system (amber/blue) seems designed for urgency vs. routine — should legal items use a different scheme, or is the existing scheme sufficient?**

**Recommendation: Extend the Amber/Blue System with Additional Visual Cues**

The existing amber (urgent) / blue (routine) system should be extended for the four-level legal urgency, not replaced. The scheme:

| Urgency Level | Visual Treatment | When to Use |
|---------------|-------------------|-------------|
| **Critical** | Amber left border + "CRITICAL" badge + pulsing dot | Filing deadlines, statute of limitations, emergency hearings |
| **High** | Amber left border + "URGENT" badge | Hearing changes, court orders, opposing counsel requests |
| **Medium** | Blue left border + "Review" badge | Client status inquiries, routine case updates |
| **Low** | Gray left border + timestamp only | Newsletters, CC'd files, bar updates |

**Do not create a separate color scheme for legal.** The dual accent system is the project's design foundation and should be treated as such. The extension adds granularity while maintaining consistency.

**Additional visual cues for legal:**
- Deadline countdown: Show "3 days left" in bold text next to the timestamp for critical/high items
- Court icon: Small courthouse icon next to court-related items
- Client avatar: Show client photo/initials for client-related items

**Why not replace:** Creating domain-specific schemes (legal=red, social media=green, etc.) creates cognitive load for users who work across domains. The amber/blue system is already established — extending it is cleaner.

---

### Question 4: Email Viewer for Legal Review

**For attorneys reviewing triggered rules, what is the optimal way to present the full email alongside the extracted context and suggested actions? Should the email be shown in a collapsible panel, a side-by-side view, or something else? Attorneys need to see the original to verify extracted context.**

**Recommendation: Collapsible Panel in the Review Sheet**

For legal review, the optimal presentation is:

1. **In the review sheet** (side drawer):
   - Top section: Extracted context in a compact summary card
   - Middle section: Collapsible "Original Email" panel
   - Bottom section: Generated actions

2. **The collapsible email panel**:
   ```
   +------------------------------------------+
   | [v] Original Email                       |
   +------------------------------------------+
   | From: clerk@court.gov                   |
   | Subject: Order — Smith v. Jones         |
   | Date: March 12, 2025 9:30 AM           |
   |------------------------------------------|
   | [Full email content displayed here]    |
   |                                          |
   | ...                                      |
   +------------------------------------------+
   ```

3. **Extracted context card** (always visible above the email):
   ```
   +------------------------------------------+
   | Extracted Information                    |
   +------------------------------------------|
   | Deadline: March 15, 2024 (3 days)      |
   | Type: Filing deadline (defendant       |
   |        response to motion)              |
   | Court: Superior Court, Maricopa County  |
   | Case: Smith v. Jones (CV2024-012345)    |
   +------------------------------------------+
   ```

**Why collapsible panel:**
- Attorneys need to verify extracted context against the original — they need full email access
- The email is not the primary focus — the extracted context and actions are
- Collapsible preserves screen real estate while keeping the email accessible
- Side-by-side creates horizontal scrolling on standard laptop screens

**Critical UX detail:** The extracted context should be clearly labeled as "Adeo extracted" with an edit icon. The attorney must be able to see what the system thought vs. what the email actually says. This is essential for building trust in the system.

---

### Question 5: Legal Deadline Calendar Integration

**Future phases may include calendar integration. How should we visualize detected deadlines in the dashboard — as a simple date, or with countdown ("3 days remaining") or both?**

**Recommendation: Show Both — Countdown Prominently, Date on Hover**

For detected deadlines, the dashboard should show:

1. **On the action card (compact):**
   ```
   Hearing — March 15 (3 days)
   ```

2. **On hover/tap (tooltip):**
   ```
   Deadline: March 15, 2025
   Filing response to Motion to Compel
   Case: Smith v. Jones
   ```

3. **In the review sheet (detailed):**
   - Large countdown: "3 days remaining" (amber for critical, blue for high)
   - Calendar icon indicator
   - Click to add to external calendar (Phase 2+)

**The countdown should be the primary display** — for attorneys, "3 days" immediately communicates urgency in a way that "March 15" does not. However, the specific date should always be accessible for verification.

**Visual treatment:**
- 1 day or less: Bold amber with "TODAY" or "TOMORROW"
- 2-3 days: Amber countdown
- 4-7 days: Blue countdown
- 7+ days: Just the date, no countdown

This creates a natural urgency gradient that aligns with the amber/blue system.

---

## Questions from Social Media Domain SME

### Question 6: Dashboard Scan Patterns

**When a social media manager opens the dashboard in the morning, what's the optimal information hierarchy? Should urgent items be first (amber border), or should time-based grouping show overnight activity separately?**

**Recommendation: Urgent Items First, with Time-Based Section Headers**

The optimal hierarchy for morning dashboard review:

1. **Sort order:** Urgency first (critical/urgent), then recency
2. **Section headers:** Group by time window within urgency tier
   - "Needs Immediate Attention" (amber section)
   - "Today" (blue section)
   - "Yesterday / This Week" (gray, collapsed by default)

**Example:**
```
[AMBER SECTION] — Needs Immediate Attention
+--------------------------------------------------+
| 9:32 AM — @angry_customer mentioned you         |
| "Your product is broken, worst service ever!"   |
| [Negative Mention — Critical] [Review Actions]  |
+--------------------------------------------------+

[BLUE SECTION] — Today
+--------------------------------------------------+
| 8:15 AM — @influencer tagged your brand        |
| "Just tried the new product, love it!"          |
| [Influencer Tag — Review] [Review Actions]      |
+--------------------------------------------------+

[Show older items]
```

**Why urgent first:**
- Social media managers need to know immediately if there's a crisis
- Time-based grouping for overnight activity ("While You Were Sleeping") is useful but secondary
- The Social Media SME analysis notes that overnight negative mentions are the most damaging — these should appear at the very top

**Section header design:**
- Amber header for urgent section: "Needs Attention" with count
- Blue header for routine section: "Today" with count
- Gray header for older items: "Earlier" (collapsed by default)

---

### Question 7: Mobile Responsiveness

**Given that managers often check mentions on phones between meetings, what's the minimum viable mobile experience? Can approval happen on mobile, or is it review-only?**

**Recommendation: Review-Only on Mobile, Approval on Desktop**

For Phase 1, the mobile experience should be **review-only** (read action items, view context, but not approve/edit):

| Feature | Desktop | Mobile |
|---------|---------|--------|
| View action items | Full card list | Compact list |
| View email context | Expandable in sheet | Tap to read in modal |
| Approve action | Full editing + approval | Disabled — "Open on desktop" |
| Reject action | Full rejection flow | Disabled — "Open on desktop" |
| Edit draft | Full revision sheet | Disabled — "Open on desktop" |
| Manage rules | Full configuration UI | View only, no editing |

**Rationale:**
1. **Draft editing on mobile is poor UX** — The revision flow requires text editing, feedback input, and careful review. Mobile keyboards are not suited for this.
2. **Approval is a deliberate act** — The HLRD targets 10-15 second decision time on desktop. Mobile approvals are more likely to be accidental or rushed.
3. **Notification fatigue** — If mobile approval is enabled, users might approve from notification taps without full context.

**Mobile MVP for Phase 1:**
- View action items in a stacked card layout
- Tap card to see email context in a full-screen modal
- Clear "Open in Desktop App" CTA to complete actions
- Badge showing pending action count for awareness

---

### Question 8: Notification Design

**Should the system send push notifications for certain triggers (negative mentions, influencer tags), or is in-app notification sufficient? What's the notification fatigue risk for a tool checked multiple times daily?**

**Recommendation: In-App Notifications Primarily, Push for Critical Items Only**

**Notification tiering:**

| Item Type | In-App Notification | Push Notification |
|-----------|--------------------|--------------------|
| Negative mention (critical) | Always | Optional — user can enable |
| Influencer tag | Always | No |
| Deadline (legal) | Always | Critical only (Phase 2) |
| Routine action item | Always | No |
| Review response needed | In summary digest only | No |

**Design pattern:**
- In-app notifications appear as a red badge on the sidebar icon
- Clicking opens the notification panel with a list of recent items
- Push notifications are opt-in, not default

**Notification fatigue mitigation:**
1. **Batch notifications** — If multiple items arrive within 10 minutes, show one notification "3 new action items"
2. **Summary frequency** — Option for hourly digest vs. instant notification
3. **Do not notify on every action item** — Only notify when urgency is critical/high, or during "morning review" batch

**The "morning batch" pattern from Social Media SME analysis:**
- At start of business day, send one notification: "You have X items to review, including Y urgent"
- This is more valuable than instant notifications for overnight activity

**Recommendation for Phase 1:** Start with in-app only. Add push notifications in Phase 2 based on user demand.

---

## Questions from E-commerce Domain SME

### Question 9: Multiple Rules on Same Email

**When multiple rules trigger on the same email (e.g., both a negative review alert and a customer service response action), how should the dashboard present these? Should they be combined into a single action item with multiple suggested responses, or separated?**

**Recommendation: Combined into Single Action Item with Multiple Actions Listed**

Similar to my answer to Question 1, but with e-commerce-specific considerations:

1. **Single card** for the email showing:
   - Highest-urgency rule indicator (negative review = critical/amber)
   - Badge: "2 rules triggered"
   - List both actions in the review sheet

2. **In the review sheet**:
   ```
   +------------------------------------------+
   | Email: 1-Star Review from @username      |
   | Triggered by: Negative Review (94%)     |
   |                 + Customer Service (78%) |
   +------------------------------------------+
   |                                          |
   | [Action 1] Review Response Draft       |
   | Rule: Negative Review Alert            |
   | Status: Ready for approval              |
   | Preview: "We're sorry to hear..."       |
   |                                          |
   | [Action 2] Refund Processing Draft     |
   | Rule: Customer Service Request         |
   | Status: Ready for approval              |
   | Preview: "I've processed your..."       |
   +------------------------------------------+
   ```

3. **The user can approve both, edit one, or reject one** — the actions are independent

**Why combined:** The email is the unit of attention. A customer sends one email with multiple issues — the seller needs to see them together to provide a cohesive response. Separating them creates the "split brain" problem.

---

### Question 10: Review Response Preview Format

**For review response actions specifically, what preview format would help a seller quickly approve vs. edit? Is a character count or preview truncation helpful?**

**Recommendation: Show First 100 Characters + Character Count, Full in Revision Sheet**

For review response actions in the action card:

```
+------------------------------------------+
| Review Response Draft                   |
| Preview: "We're sorry to hear about..."  |
|        [85 characters] [More below]     |
+------------------------------------------+
```

In the revision sheet (when "Review Actions" is clicked):
```
+------------------------------------------+
| Generated Response                      |
+------------------------------------------+
| We're sorry to hear about your          |
| experience. We always strive for        |
| perfect orders and apologize that       |
| this one fell short. Please reach       |
| out to us directly so we can make       |
| this right.                             |
|                                          |
| Character count: 185 (Amazon limit: 200) |
|                                          |
| [Edit Draft] [Provide Feedback for AI]  |
+------------------------------------------+
```

**Key UX elements:**
1. **Preview truncation** — Show first ~100 characters so the user can gauge tone
2. **Character count** — Review platforms have hard limits (Amazon: 200 chars). Show this clearly.
3. **"More below" indicator** — If truncated, show continuation indicator
4. **Platform limit indicator** — If approaching limit, show warning color

**Why not show full in card:** Review responses are short (200 characters), so showing full in the card would be 3-4 lines — acceptable but clutters the list. Truncation keeps cards compact.

---

### Question 11: Marketplace Context in UI

**How should the system indicate marketplace-specific context (Amazon vs. Shopify) in the rule and action configuration UI without overwhelming users who only use one marketplace?**

**Recommendation: Contextual Hints, Not Upfront Configuration**

The system should not force marketplace selection upfront. Instead:

1. **At domain selection** (initial setup): User selects "E-commerce" without specifying marketplace
2. **In rule configuration** (when creating rules):
   - Show generic rule templates by default
   - When user creates a rule, offer "Add marketplace-specific variant" as an optional step
   - The variant appears as an expandable section, not a required field

3. **In action generation** (when actions are generated):
   - Show marketplace context as contextual hint: "Generated for Amazon (185/200 chars)"
   - This appears in the action preview, not as a configuration step

4. **Example UI pattern:**
   ```
   +------------------------------------------+
   | Rule: Negative Review Response          |
   |                                          |
   | Base prompt: [editable]                 |
   |                                          |
   | [+] Add Amazon variant                  |
   | [+] Add Shopify variant                 |
   |                                          |
   | Preview: [Shows generic response]       |
   +------------------------------------------+
   ```

**Why this approach:**
- Users who only sell on Amazon shouldn't be forced to think about Shopify
- Marketplace context becomes relevant at action-generation time, not setup time
- The "variant" pattern (from the E-commerce SME analysis) maps directly to this UI
- Contextual hints in action previews reinforce marketplace awareness without overwhelming

---

## Questions from Integration Engineer SME

### Question 12: Multi-Tenant Dashboard Indicator

**The dashboard shows action items from triggered rules. How should we indicate tenant context in the UI for users who might have access to multiple tenants in future phases? Should there be a tenant switcher component in the sidebar?**

**Recommendation: Domain Selector in Sidebar, Not Tenant Switcher (Phase 1)**

**For Phase 1 (single tenant per user):**
- Show domain indicator: "Domain: Legal" or "Domain: E-commerce" in the sidebar
- No tenant switching needed — each user has one domain

**For future phases (multi-tenant):**
- Use a "context switcher" pattern in the sidebar top area
- Show: Current tenant name/avatar, click to see accessible tenants
- Pattern:
   ```
   +------------------+
   | [Avatar] Smith  |
   | Law Firm ▼      |  <- Click to see: Other Firm, Personal
   +------------------+
   ```

**Why domain selector now, not tenant switcher:**
1. The HLRD specifies single-tenant operation for Phase 1
2. Domain (Legal/Social/Ecommerce) is the relevant context now, not tenant
3. "Tenant" terminology is technical — "Law Firm" or "My Business" is user-friendly
4. The tenant switcher pattern can be added in Phase 2 when multi-tenancy is implemented

**Current implementation for Phase 1:**
- Sidebar shows: "Adeo" logo, "Domain: [Legal ▼]" selector
- Domain selector allows switching between Legal, Social Media, E-commerce
- No tenant indication needed (single-tenant)

**The domain selector is the right abstraction** for Phase 1. In Phase 2+, it could expand to show "Smith Law Firm (Legal)" or "Acme Shop (E-commerce)" as the context.

---

## Summary of UX Decisions

| Question | Decision |
|----------|----------|
| 10+ rules on one email | Collapse to priority card, expand in sheet |
| Contradictory actions | Side-by-side in review sheet |
| Legal urgency levels | Extend amber/blue with 4 levels |
| Email viewer | Collapsible panel in review sheet |
| Deadline display | Countdown prominently + date on hover |
| Morning dashboard | Urgent first, then time-based sections |
| Mobile experience | Review-only, no approval on mobile |
| Notifications | In-app for all, push opt-in for critical |
| Multiple rules | Combine into single card |
| Review preview | Truncate + character count |
| Marketplace context | Contextual hints, not upfront config |
| Tenant context | Domain selector now, tenant switcher later |

---

## Notes for Phase 3 Requirements

These answers should inform specific FR/NFR wording in the requirements draft:

1. **FR: Mobile approval** — Specify that mobile is review-only for Phase 1
2. **FR: Conflict detection** — Require UI presentation of contradictory actions
3. **NFR: Notification batching** — Specify 10-minute batching window
4. **FR: Deadline countdown** — Visual display requirement with gradient by urgency

---

*Answers prepared for Phase 2b cross-SME consultation. These recommendations are based on established UX patterns for professional dashboard tools and the specific context from all Phase 1 SME analyses.*