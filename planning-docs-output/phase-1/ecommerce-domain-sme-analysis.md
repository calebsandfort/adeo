# Phase 1 SME Analysis: E-commerce Domain

## Executive Summary

This analysis addresses the e-commerce operations domain considerations for the Adeo system, focusing on the workflows, risks, and communication patterns specific to solo sellers and small e-commerce operations (1-5 people). The recommendations are grounded in the operational realities of small sellers on Shopify and Amazon, where missed notifications directly translate to lost revenue, negative review cascades, inventory crises, and account health problems that can result in marketplace suspension.

---

## Question 1: Highest-Impact Missed Notifications for Small E-commerce Sellers

### Direct Answer

For small e-commerce sellers, **operational pain points outweigh customer-facing ones** in terms of immediate revenue impact, but **customer-facing issues compound over time** and represent the broader threat to sustainability. The hierarchy of impact is:

1. **Inventory stockouts** — immediate lost sales, ranking penalties, and potential buy box loss
2. **Marketplace policy violations** — account health warnings, listing removals, or suspension
3. **Negative reviews without response** — compound damage to conversion rates and long-term brand reputation
4. **Payment/fulfillment issues** — chargebacks, late shipment penalties, and customer complaints
5. **Customer service gaps** — relationship erosion with repeat customers

### Detailed Analysis

#### Operational vs. Customer-Facing: Quantifying the Impact

**Operational Issues (Higher Immediate Revenue Impact)**

| Scenario | Revenue Impact | Detection Urgency |
|----------|---------------|-------------------|
| Stockout on best-seller | 3-7 days of lost revenue per day delayed; ranking decay begins within 48 hours | Must act within hours |
| Listing removal (policy violation) | Immediate sales stoppage; recovery takes days | Must act within 24-48 hours |
| Payment failure on order | 100% revenue loss on that order; potential chargeback | Must resolve within days |
| Address verification issue | Order frozen until resolved; customer may abandon | Must resolve within 24 hours |
| FBA/3PL inventory alert | Loss of Prime eligibility, increased fees | Must act within days |

For a solo seller doing 50 orders/day with a $30 average order value, a 3-day stockout on a best-seller costs approximately $4,500 in lost revenue — often the difference between profitability and loss for the month.

**Customer-Facing Issues (Higher Long-Term Cumulative Impact)**

| Scenario | Revenue Impact | Detection Urgency |
|----------|---------------|-------------------|
| Negative review (1-2 stars) without response | 2-5% conversion rate decrease per unanswered negative review | Respond within 24-48 hours |
| High-value customer complaint | Customer lifetime value (LTV) loss; potential for chargeback | Must respond within hours |
| Return request from repeat customer | Lost sale + return logistics cost + potential for feedback manipulation | Must respond within 24 hours |
| Product defect complaint | Signals need for quality control; public response affects future buyers | Respond within 24 hours |

A single unanswered 1-star review can reduce conversion by 2-5% — for a listing with 100 daily visitors and 3% conversion, that's 1-2 lost orders per day, compounding over weeks.

#### Priority Ranking for E-commerce Preset Rules

Based on impact analysis, the e-commerce preset rules should be prioritized as follows:

**Tier 1: Critical (Must Have)**
1. **Inventory low-stock warning** — stockouts are the most costly operational failure
2. **Marketplace policy/listing change** — account health is existential
3. **Order issue detection (payment/address)** — frozen orders = lost revenue

**Tier 2: High Priority**
4. **Negative product review alert** — review damage compounds
5. **Refund/return request (high-value or repeat customer)** — LTV protection

**Tier 3: Important**
6. **Customer email requiring response** — prevents relationship erosion

#### Solo vs. Small Team Considerations

For a **solo seller doing 50 orders/month**, the pain points are concentrated differently:
- They likely fulfill orders themselves, so shipping issues are visible quickly
- They cannot monitor everything simultaneously — stockouts happen because they sell out without noticing
- Customer emails are personal — every complaint feels significant
- Account health monitoring is reactive — they discover problems when Amazon/Shopify notifies them

For a **small team doing 500 orders/month**:
- More automated processes but higher volume of issues to track
- Need systematic handling — cannot personalize every response
- Account health metrics become more visible (ODR, late shipment rate)
- Inventory management becomes a serious constraint

---

## Question 2: Handling Shopify vs. Amazon Seller API Differences

### Direct Answer

The e-commerce preset should define rules **generically with marketplace-specific sub-configurations** — a hybrid approach. Generic rules that work across marketplaces reduce maintenance burden and complexity, while marketplace-specific configurations handle the meaningful differences in API structure, terminology, and policy environments.

### Detailed Analysis

#### Key Differences Between Shopify and Amazon Seller APIs

| Dimension | Shopify | Amazon Seller Central |
|-----------|---------|----------------------|
| **API Structure** | RESTful GraphQL available; webhook-based notifications | SP-API with paginated responses; notification categories |
| **Data Available** | Orders, inventory, customers, products, fulfillment | Orders, inventory, feedback, FBA metrics, advertising |
| **Notification Types** | Order created/updated, product/out-of-stock, fulfillment | Order notifications, inventory alerts, review alerts, policy alerts |
| **Terminology** | Customer, Order, Product, Fulfillment | Buyer, ASIN, SKU, FNSKU, A-to-Z claims |
| **Policy Environment** | Less prescriptive; Shopify Payments rules | Highly prescriptive; Amazon strict on performance metrics |
| **Performance Metrics** | Basic (cancellation rate, late shipment) | Complex (ODR, late shipment rate, valid tracking rate, refund rate) |

#### Recommended Approach: Generic Rules with Marketplace Overrides

**Generic Rule Definition:**
```
Rule: Inventory Low-Stock Warning
- Generic Trigger: Inventory level < reorder threshold
- Generic Action: Generate reorder email to supplier
- Marketplace Override: Different threshold values per marketplace
```

**Specific Implementation:**

1. **Rule: Order Issue Detection**
   - Generic: Detect payment failures, address verification, fulfillment errors
   - Shopify config: Look for `payment_gateway_declined`, `address_verification`, `inventory_reservation`
   - Amazon config: Look for `PaymentMethodDeclined`, `InvalidAddress`, `ASINInventoryNotAvailable`

2. **Rule: Negative Review Alert**
   - Generic: Detect reviews at or below threshold
   - Amazon: Pull from Product Review API, trigger on 1-2 star
   - Shopify: Pull from Product Reviews API (if available) or semantically evaluate customer feedback

3. **Rule: Marketplace Policy Alert**
   - Generic: Detect policy violations, listing removals, account warnings
   - Amazon: More granular — IP infringement, product safety, category restrictions, ODR warnings
   - Shopify: Less granular — typically basic policy violations, fee changes

4. **Rule: Inventory Low-Stock Warning**
   - Generic: Inventory < threshold
   - Amazon: Distinguish FBA (warehouse count) vs. merchant-fulfilled (available inventory)
   - Shopify: Total inventory available across locations

#### Implementation Recommendation

The system should define:
- **Base rule templates** with generic prompts that work for both marketplaces
- **Marketplace configuration objects** that map marketplace-specific API fields to generic entity types
- **Action generation prompts** with marketplace-specific tone/tterminology variations

This allows adding new marketplaces (eBay, Walmart) in the future without duplicating rule logic.

---

## Question 3: Tone Conventions for Marketplace Review Responses vs. Direct Customer Emails

### Direct Answer

**Review responses and customer emails require fundamentally different approaches:**

- **Review responses** — Public-facing, permanent, influence future buyers' decisions. Tone: professional yet approachable, solution-oriented, concise, and non-defensive. Structure: acknowledge + apologize (if applicable) + offer resolution (when appropriate) + invite private communication.
- **Direct customer emails** — Private, relationship-building, can be more conversational. Tone: warm, helpful, personal, empathetic. Structure: address specific concern + provide clear next steps + close with availability for follow-up.

### Detailed Analysis

#### Marketplace Review Response Conventions

**Amazon:**
- Keep responses under 200 words
- Never argue or be defensive
- Do not include personal information (order numbers are acceptable)
- Take conversation offline: "Please contact us directly at..."
- Thank positive reviewers briefly
- For negative reviews: acknowledge, apologize, offer to make it right

**Shopify (public product reviews):**
- Similar to Amazon but slightly more flexibility in tone
- Can reference order details more freely since on your own domain
- More opportunity to showcase brand personality

**Key conventions:**
- **Never delete or flag negative reviews** — this draws more attention
- **Respond within 24-48 hours** — shows active customer service
- **Be brief** — future buyers scan reviews, not read every response
- **Use keywords naturally** — future buyers search for solutions in reviews
- **Never promise what you cannot deliver**

#### Customer Email Tone

**Direct customer emails should:**
- Use the customer's name
- Reference their specific order (when applicable)
- Be warm and empathetic — acknowledge their frustration
- Provide clear, actionable next steps
- Sign off personally
- Be slightly longer and more detailed than review responses

**Example comparison:**

*Review Response (public):*
> "We're sorry to hear about your experience. We always aim for perfect orders and apologize that this one fell short. Please reach out to us directly so we can make this right."

*Customer Email (private):*
> "Hi Sarah, I'm so sorry about the issues with your order #1042. I completely understand your frustration — receiving a damaged item is disappointing. I've already issued a full refund and we'd love to send you a replacement at no cost. Would you prefer that, or should we keep the refund? Please let me know either way and I'll take care of it immediately. Please don't hesitate to reach out if you have any other questions."

#### Revenue Impact of Review Response

- **Unanswered negative review**: 2-5% conversion rate decrease
- **Good response to negative review**: Can recover 30-50% of lost conversion
- **Response time**: Responding within 24 hours shows "active" seller status
- **Brand perception**: Consistent professional responses build trust

#### Implementation in Action Generation

The action generation prompts should:
1. **Include marketplace context** in the prompt (Amazon vs. Shopify)
2. **Specify word count limits** (shorter for reviews, longer for emails)
3. **Include explicit tone instructions** (public vs. private)
4. **Reference entity data** (customer name for email, can omit for review)
5. **Include appropriate sign-off** (direct contact info for email, generic for review)

---

## Question 4: Handling Product-Specific Context in Responses

### Direct Answer

**Yes, product category metadata should influence action generation prompts.** A sizing complaint requires fundamentally different language than a shipping damage complaint, which is different from a quality concern. The system should inject product category, common issue types, and resolution pathways into the action generation prompt to produce relevant, specific responses rather than generic templates.

### Detailed Analysis

#### Product Category Impact on Response Tone

| Product Category | Common Issues | Response Approach |
|------------------|---------------|-------------------|
| Apparel | Sizing, fit, color accuracy | Empathize with fit concerns; offer size exchange; reference size chart |
| Electronics | Defective, not working, missing parts | Request order/details; offer replacement or refund; troubleshoot |
| Home goods | Damaged on arrival, missing parts | Acknowledge shipping damage; offer replacement; ask for photos |
| Beauty/skincare | Allergic reaction, skin irritation, product quality | Empathize; do not dispute; offer refund; ask for batch/lot info |
| Food/supplements | Expired, wrong item, allergic concerns | Take very seriously; immediate refund; do not argue |
| Books/media | Wrong item, damaged, not as described | Straightforward replacement or refund |

#### How Product Context Improves Responses

**Without product context:**
> "We're sorry to hear about your experience. Please contact us for a resolution."

**With product context (apparel):**
> "We're sorry to hear the Medium didn't fit as expected — sizing can definitely vary! We'd be happy to send you a Large in exchange, or if you'd prefer a different style entirely, we can do a full refund. Many customers find our sizing runs slightly small, so if you'd like to exchange, we recommend sizing up. Let us know which you prefer!"

**With product context (electronics):**
> "We're sorry your product isn't working properly. This is rare — we'd like to get a replacement out to you immediately. Could you please confirm which model you received and whether any lights or indicators came on when you plugged it in? This helps us determine if it's a defect or a one-time issue."

#### Recommended Implementation

1. **Product entity schema should include:**
   - Product category (clothing, electronics, home, beauty, etc.)
   - Common issue types (sizing, defects, shipping damage, quality)
   - Standard resolution options (refund, exchange, replacement, store credit)
   - Keywords for issue detection

2. **Action generation prompts should:**
   - Include product category in context
   - Specify issue type if detected by rule
   - Reference resolution options available for that product
   - Include product-specific empathy language

3. **Rule evaluation should extract:**
   - Issue type from customer communication (sizing, damage, quality, shipping)
   - Product category from order/product context
   - Customer history (new vs. repeat, high-value vs. standard)

#### Edge Cases

- **Multi-item orders**: Each item may need different language
- **Bundled products**: More complex — which item is the issue?
- **Subscription products**: Different tone — emphasize continuity and retention
- **Custom/personalized products**: Refund policies differ; response should reflect this

---

## Questions for Other SMEs

### For AI/NLP Architecture SME:
- How should the system handle extracting issue type (sizing vs. damage vs. quality) from customer communications? Should this be a separate classification step in the rule evaluation pipeline, or integrated into the evaluation prompt?
- For product-specific response generation, should we use few-shot examples in the action generation prompt showing correct language for each product category, or is a category label + guidelines sufficient?
- How can we optimize token usage when injecting product context into action generation prompts without losing the specificity that makes responses useful?

### For UX Designer SME:
- When multiple rules trigger on the same email (e.g., both a negative review alert and a customer service response action), how should the dashboard present these? Should they be combined into a single action item with multiple suggested responses, or separated?
- For review response actions specifically, what preview format would help a seller quickly approve vs. edit? Is a character count or preview truncation helpful?
- How should the system indicate marketplace-specific context (Amazon vs. Shopify) in the rule and action configuration UI without overwhelming users who only use one marketplace?

### For Integration Engineer SME:
- How should the product entity schema handle multi-marketplace listings? A single product might have different SKUs on Shopify vs. Amazon, different inventory levels, and different review profiles. Should the entity model support marketplace-specific sub-records?
- For the simulated data sources, how should we represent the difference between Shopify and Amazon API payloads? Should we have separate simulated providers for each, or a unified provider with marketplace configuration?

---

## Summary Recommendations

1. **Prioritize operational rules first** — inventory and policy alerts prevent existential threats; customer-facing rules address long-term sustainability
2. **Use generic rules with marketplace overrides** — reduces maintenance, enables future expansion to new marketplaces
3. **Differentiate public vs. private tone explicitly** — review responses need different structure and constraints than customer emails
4. **Inject product context into action generation** — category and issue type significantly improve response relevance and reduce generic "sorry for the inconvenience" responses
5. **Design for solo sellers first** — 50 orders/month seller has different detection needs than 500 orders/month operation; build for the smaller scale and scale up features as needed

---

*Analysis prepared for Adeo Phase 1 requirements elaboration. All recommendations assume simulated data sources and single-tenant operation for Phase 1.*
