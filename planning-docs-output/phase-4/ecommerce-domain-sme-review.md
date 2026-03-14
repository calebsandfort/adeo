# E-commerce Domain SME Review — Requirements Draft

**Review Date:** 2026-03-13
**Reviewed By:** E-commerce Domain SME
**Phase:** 4 — SME Review

---

## Summary

The e-commerce domain preset (FR-9.0) captures core seller workflows well, with appropriate attention to marketplace-specific distinctions between public review responses and private customer communications. However, there are significant gaps in account health monitoring, chargeback/dispute management, and FBA-specific inventory handling that are critical for small sellers. Additionally, several requirements lack the specificity needed for realistic small seller operations.

**Recommendation:** Major revisions needed before acceptance.

---

## 1. Accuracy Issues

### FR-9.1 — Rule Completeness

**Current:**
- Order issue detection
- Negative product review alert
- Inventory low-stock warning
- Customer email requiring response
- Refund/return request
- Marketplace policy/listing change

**Issues Identified:**

1. **Missing: Account Health Alerts** (HIGH PRIORITY)
   - Amazon sellers track ODR (Order Defect Rate), late shipment rate, and cancellation rate
   - A drop below Amazon's thresholds triggers account suspension risk
   - Missing requirement for alerts when metrics approach warning thresholds
   - **Revenue Impact:** Account suspension halts all sales immediately

2. **Missing: A-to-Z Guarantee Claims** (HIGH PRIORITY)
   - Amazon A-to-Z claims directly affect Buy Box eligibility and account health
   - Small sellers often miss these within the 7-day response window
   - **Revenue Impact:** A-to-Z claims can result in chargebacks plus a 5-10% account health penalty

3. **Missing: Chargeback/Dispute Detection** (HIGH PRIORITY)
   - Chargebacks disproportionately impact small sellers ($15-$20 per chargeback plus increased processing fees)
   - Early detection allows seller to contest before final decision
   - **Revenue Impact:** $15-$20 direct cost plus potential account termination on repeated chargebacks

4. **Missing: Buy Box Loss Alerts** (MEDIUM PRIORITY)
   - Buy Box loss means lost sales even when listing is visible
   - Often caused by price, fulfillment speed, or account health issues
   - **Revenue Impact:** Can reduce orders by 30-70% for affected products

5. **Missing: MAP (Minimum Advertised Price) Violations** (MEDIUM PRIORITY)
   -重要 for brand-registered sellers on Amazon
   - Violations can result in listing suppression

**Suggested Addition to FR-9.1:**
```
- Account health metric alerts (ODR, late shipment, cancellation rates)
- A-to-Z guarantee claim detection
- Chargeback/dispute early warning
- Buy Box loss notification
- MAP violation detection (for brand-registered sellers)
```

### FR-9.4 — Issue Type Extraction

**Current:** sizing, damage, quality, shipping, wrong_item

**Issues:**

1. Missing: **"not as described"** — Extremely common Amazon return reason, different handling than "wrong_item"
2. Missing: **"defective/ malfunctioning"** — Triggers different resolution (replacement vs refund) and affects inventory disposition
3. Missing: **"missing parts/ accessories"** — Common issue with multi-piece products

**Suggested Revision:**
```
The system SHALL extract issue type (sizing, damage, quality, shipping, wrong_item, not_as_described, defective, missing_parts) as part of the rule evaluation output.
```

---

## 2. Completeness Gaps

### FBA/3PL-Specific Inventory Handling

The requirements assume self-fulfilled inventory, but many small e-commerce sellers use FBA or third-party logistics (3PL). This creates gaps:

1. **Missing: FBA Inbound Shipment Tracking**
   - Need alerts when shipments are delayed, lost, or received with quantity discrepancies
   - Affects inventory availability and restock timing

2. **Missing: Restock Lead Time Alerts**
   - FBA sellers need warnings when inventory won't arrive before stockout
   - Should consider: current stock level, daily velocity, lead time, in-transit quantity

3. **Missing: Aged Inventory Warnings**
   - Amazon charges long-term storage fees after 365 days
   - Need alerts when inventory approaches age thresholds

**Suggested New Requirement (FR-9.X):**
```
The e-commerce preset SHALL support FBA-specific inventory rules including:
- Inbound shipment delay/loss detection
- Stockout prediction based on lead time and velocity
- Aged inventory threshold warnings (270+, 365+ days)
```

### Marketplace-Specific Field Mapping

FR-9.3 mentions "overrideable field mappings" but doesn't specify what fields differ between platforms. This is critical for implementation:

| Field | Shopify | Amazon |
|-------|---------|--------|
| Order ID | "order_id" / "name" | "amazon_order_id" |
| Return Window | "applying" / "requested" | "return_window_days" |
| Customer Name | "customer.first_name" + "customer.last_name" | "buyer_name" |
| Product ID | "variant.id" | "asin" |
| Fulfillment | "fulfillment_status" | "fulfillment_channel" (AFN/MFN) |
| Refund Amount | Calculated from "refund_items" | "amount" from Refund API |

**Suggested Addition to FR-9.3:**
```
The system SHALL document marketplace-specific field mappings in a configuration file accessible to users, with default mappings for Shopify and Amazon Seller Central APIs.
```

### Customer Service Response Windows

Different marketplaces have different response time expectations. This affects urgency classification:

- **Amazon:** 24-hour response window for buyer messages (policy requirement)
- **Shopify:** No strict requirement, but same-day is best practice
- **Etsy:** 48-hour response window expected

**Suggested Addition to FR-9.1:**
```
- Customer message response urgency (24hr for Amazon, 48hr for Etsy)
```

---

## 3. Conflicts with E-commerce Best Practices

### Review Response Permanence (FR-9.7)

The requirement states: "The system SHALL differentiate tone and format between review responses (public, professional, concise) and customer emails (private, warm, detailed)."

**Issue:** This is accurate, but incomplete. Should add:
- Review responses are **public and permanent** — cannot be edited after posting
- First response is critical for SEO and customer perception
- Should NOT include: shipping details, troubleshooting steps, or requests for removal (violates platform policy)
- Should include: thank the customer, address specific concern, invite direct contact for resolution

**Suggested Enhancement to FR-9.7:**
```
The system SHALL differentiate tone and format between review responses (public, professional, concise, no shipping/troubleshooting details, no requests for removal) and customer emails (private, warm, detailed).
```

### Refund/Return Action Tone (FR-9.6)

Current requirement: "Draft refund/return approval"

**Issue:** Tone should vary by refund type:
- **Refund without return** (AmazonKeep) — Requires less formal tone, can include sympathy
- **Return + refund** — More standard, includes return instructions
- **Partial refund** — Most sensitive, needs explanation of reasoning

**Suggested Enhancement to FR-9.6:**
```
- Draft refund/return approval (with type-specific tone: no-return, full refund, partial refund, store credit option)
```

### Financial Materiality of Actions

Small sellers must approve actions that have real financial consequences. The current requirements don't address:
- Refunds reduce profit margin — some orders may be worth accepting the negative review rather than refunding
- Dispute responses have time limits — missing deadlines compounds the problem
- Inventory decisions affect cash flow — over-ordering ties up capital

**Suggested Addition (consider as NFR or Integration consideration):**
```
The system SHALL surface financial context in action previews where applicable:
- Refunds: Show order value and profit margin context (if available)
- Reorders: Show current stock value and supplier MOQ
- Disputes: Show response deadline and escalation risk
```

---

## 4. Missing Operational Realities for Small Sellers

### Seller Account Health Metrics (Amazon-Specific)

Small Amazon sellers live by account health metrics. Missing:
- ODR (Order Defect Rate) warning thresholds
- Late shipment rate tracking
- Pre-fulfillment cancellation rate
- Valid tracking rate
- Policy violation alerts

### Product Listing Health

- Missing: "Listing suppressed due to policy violation" alerts
- Missing: "Buy Box lost due to price" alerts
- Missing: "Category restriction applied" alerts

### Customer Lifetime Value Considerations

The current approach treats all customers equally. Small sellers know:
- Repeat customers are more valuable — issues with them deserve higher priority
- First-time buyers with issues may not return — resolution quality affects this
- VIP/Big spenders need faster response

**Not suggesting to add this to rules** — too complex for Phase 1. But noting this as a gap for future phases.

---

## 5. Positive Observations

### FR-9.2 — Hybrid Evaluation Approach
Excellent design decision. Using deterministic threshold checks for structured API fields (like order status, inventory count) plus LLM for semantic interpretation (email content, review text) is the right balance for small seller operations.

### FR-9.3 — Marketplace-Specific Sub-configurations
Correctly identifies that the same rule logic applies across platforms but field names differ. The overrideable field mapping approach will work well.

### FR-9.5 — Product Category Metadata
Smart approach to inject category-specific context into action generation. This addresses the real problem that electronics returns are different from apparel returns.

### FR-9.7 — Public vs. Private Tone Differentiation
Critical distinction that many automation tools get wrong. Public review responses must be concise, professional, and policy-compliant. Private emails can be warmer and more detailed.

---

## 6. Questions for Other SMEs

### For AI/NLP Architecture SME:
- How should the system handle mixed-language customer emails (e.g., English customer writing about a product that uses Chinese terminology)? Should this affect confidence scores differently for e-commerce vs. legal domain?

### For Integration Engineer SME:
- The field mapping approach (FR-9.3) assumes we have accurate API documentation for Shopify and Amazon. What happens when API fields change or new fields are added? Should there be a schema versioning mechanism?

### For Product Manager:
- The "Flag for seller action" action type (FR-9.6) is a placeholder for situations where no draft response is appropriate. Should this include suggested next steps (e.g., "Contact supplier," "Review Amazon policy," "File claim")?

---

## 7. Recommended Changes

### Priority 1 (Must Fix):
1. Add account health alerts to FR-9.1
2. Add A-to-Z claim detection to FR-9.1
3. Add chargeback early warning to FR-9.1
4. Add "not_as_described" and "defective" to FR-9.4 issue types

### Priority 2 (Should Fix):
5. Add FBA inventory rules (inbound, restock prediction, aged inventory)
6. Add marketplace field mapping documentation requirement
7. Enhance FR-9.7 with review response permanence and policy constraints

### Priority 3 (Nice to Have):
8. Add MAP violation detection
9. Add Buy Box loss alerts
10. Add customer message response window urgency

---

## 8. Verdict

**Acceptable with Major Revisions**

The e-commerce domain preset captures approximately 60% of the critical workflows for small marketplace sellers. The gaps around account health monitoring and chargeback management represent significant revenue risk that small sellers face daily. These should be addressed before Phase 1 implementation.

The hybrid evaluation approach (FR-9.2) and marketplace sub-configuration strategy (FR-9.3) demonstrate good understanding of the technical realities of multi-marketplace selling. These architectural decisions are sound.

---