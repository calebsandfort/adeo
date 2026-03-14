# Phase 1 SME Analysis: Legal Domain

## Executive Summary

This analysis addresses the legal practice domain considerations for the Adeo system, focusing on the workflows, risks, and communication patterns specific to solo practitioners and small law firms. The recommendations are grounded in the operational realities of attorneys who manage caseloads without dedicated administrative support, where missed communications directly translate to malpractice risk, client harm, and professional consequences.

---

## Question 1: Most Common Email-Driven Dropped Balls in Solo/Small Firm Practice

### Direct Answer

The most dangerous email-driven dropped balls in solo/small firm practice fall into three categories:

1. **Deadline-related communications** — emails containing deadlines for filings, responses, or compliance that slip through the cracks
2. **Client communication gaps** — unresponded client inquiries that damage the relationship and create liability exposure
3. **Court/opposing counsel action items** — requests, demands, or notices requiring a response or procedural step

### Detailed Analysis

#### Highest-Risk Scenarios (By Malpractice Impact)

**A. Missed Filing Deadlines**
The most consequential dropped balls involve statutory deadlines, court-imposed filing deadlines, and discovery response deadlines. In legal practice, a missed deadline often cannot be cured — the case may be dismissed, a motion may be deemed uncontested, or evidence may be excluded.

*Real-world patterns:*
- Emails from court clerks with new deadlines attached to orders
- Emails from opposing counsel with discovery responses due
- Emails from mediation services with agreement deadlines
- Emails with embedded calendar invitations for filing deadlines (easily missed among routine calendar items)

*Risk assessment:* Catastrophic. Missed deadlines are the leading cause of legal malpractice claims.

**B. Court Scheduling Changes**
When a hearing is cancelled, rescheduled, or moved to a different modality (in-person to Zoom), the attorney must adjust their schedule, notify the client, and potentially file a continuance if conflicts arise.

*Real-world patterns:*
- Emails from courtrooms announcing hearing cancellations
- Emails from opposing counsel proposing alternative dates
- Emails from mediators indicating scheduling conflicts
- Emails from judicial assistants with room or time changes

*Risk assessment:* High. Missing a hearing can result in default judgments, dismissed cases, or contempt findings.

**C. Client Inquiries Going Unanswered**
Clients interpret non-response as neglect. In family law especially, clients are anxious and may interpret delayed responses as signs of abandonment or incompetence. In personal injury, clients may have urgent questions about medical treatment or settlement status.

*Real-world patterns:*
- Client emails asking "what's happening with my case?"
- Client emails with questions about next steps
- Client emails expressing concern or frustration
- Client emails with new information relevant to their case (address changes, new witnesses, etc.)

*Risk assessment:* Moderate to high. While not immediately catastrophic, client communication failures erode trust, lead to bar complaints, and are a significant source of malpractice exposure.

**D. Opposing Counsel Communications**
Requests for extensions, demands for discovery, notices of deposition, and proposals for settlement all require timely attention.

*Real-world patterns:*
- Emails requesting extension of time (requires response even if granted)
- Emails with settlement proposals (silence may be deemed rejection in some contexts)
- Emails scheduling depositions or examinations
- Emails withRule 26(f) disclosures or discovery requests

*Risk assessment:* High. Failure to respond can result in sanctions, default judgments, or procedural disadvantages.

#### Practice-Area-Specific Considerations

**Family Law**
- Client anxiety and high communication volume — spouses in contentious divorces send frequent, emotional emails
- Deadlines tied to court orders with specific compliance dates (e.g., "within 30 days of service")
- Parenting plan modifications and custody exchanges requiring coordination
- Financial disclosure requirements with strict deadlines

*Recommended rules specific to family law:*
- Detect emails containing custody, visitation, or parenting plan keywords
- Identify court orders with specific compliance deadlines
- Flag client emails expressing frustration or suggesting dissatisfaction with representation

**Personal Injury**
- Client medical updates that affect settlement value
- Insurance company communications about policy limits or coverage
- Expert witness reports and medical records
- Statute of limitations awareness (though typically tracked in case management)

*Recommended rules specific to personal injury:*
- Detect emails from insurance adjusters with settlement offers
- Identify client emails about new medical treatment or prognosis
- Flag emails about medical records or records requests

**Criminal Defense**
- Court appearance reminders (most critical)
- Client status inquiries (family members often contacting about loved ones in custody)
- Discovery production from prosecutors
- Plead offer deadlines

*Recommended rules specific to criminal defense:*
- Detect court emails with trial dates, sentencing dates, or probation violation notices
- Identify emails from prosecutors with plea offers or discovery
- Flag family member inquiries about client status

**Estate Planning/Probate**
- Court deadlines for probate filings
- Beneficiary inquiries about estate administration
- Documents requiring client signature
- Tax filing deadlines for estates

**General Civil Litigation**
- Discovery requests and responses
- Court orders requiring compliance
- Settlement conference or mediation notices
- Case management conference reminders

### Risk Priority Matrix

| Scenario | Missed Consequence | Frequency | Detection Priority |
|----------|-------------------|-----------|---------------------|
| Filing deadline | Case dismissal/default | Low-medium | Critical |
| Hearing change | Adverse ruling, contempt | Medium | Critical |
| Court order | Sanctions, procedural loss | Medium | Critical |
| Client inquiry | Bar complaint, malpractice | High | High |
| Opposing counsel request | Sanctions, adverse outcome | Medium-high | High |
| Client status update | Client dissatisfaction | High | Medium |

---

## Question 2: Handling Urgency in Legal Communications

### Direct Answer

The legal preset **must** include urgency classification within rules. Attempting to handle urgency as a separate cross-cutting concern will fail because urgency in legal communications is fundamentally context-dependent and tied to the substantive nature of the communication, not just surface-level indicators.

### Analysis

#### Why Urgency Cannot Be a Separate Layer

Legal urgency is not simply a matter of keywords or patterns. An email that says "please respond at your convenience" might actually be urgent (e.g., a court order with a 10-day response deadline), while an email marked "URGENT" might be routine (e.g., opposing counsel's cover letter). The urgency is determined by:

1. **The nature of the underlying legal matter** — a filing deadline is always urgent; a courtesy copy is never urgent
2. **The procedural posture** — deadlines in pending cases matter; deadlines in closed cases do not
3. **The specific jurisdiction's rules** — some courts have shorter response times than others
4. **The case status** — a deadline in an active case matters; the same deadline in a dormant case may not

Therefore, urgency classification must be embedded within each rule's evaluation prompt, not applied as a post-processing layer.

#### Recommended Implementation

Each legal rule should return urgency classification as part of its structured output:

```json
{
  "triggered": true,
  "confidence": 0.92,
  "urgency": "critical|high|medium|low",
  "extracted_context": {
    "deadline_date": "2024-03-15",
    "deadline_type": "filing",
    "court": "Superior Court",
    "case_reference": "Smith v. Jones"
  },
  "reasoning": "Email contains court order with March 15 filing deadline for defendant's response to motion"
}
```

#### Urgency Levels for Legal Domain

**Critical** — Must be addressed within hours or by end of next business day:
- Court-ordered deadlines (filing, response, compliance)
- Statute of limitations approaching
- Emergency hearing notifications
- Client emergencies (arrest, immediate danger)
- Time-sensitive settlement offers with deadline

**High** — Should be addressed within 1-2 days:
- Scheduling changes for hearings
- Opposing counsel discovery requests
- Court orders requiring attention
- Client inquiries with specific questions about case status

**Medium** — Response expected within a week:
- Routine client status updates
- Bar association or continuing education communications
- General case-related inquiries

**Low** — Can be addressed at attorney's convenience:
- Newsletters, bar updates
- Marketing materials
- CC'd documents for file maintenance
- Non-substantive correspondence

#### Interaction with Rule Evaluation

The urgency classification should influence:
1. **Dashboard sorting** — critical items bubble to top regardless of timestamp
2. **Notification approach** — critical items warrant push notification (future phase)
3. **Action generation tone** — urgent matters may require more direct language in drafts

#### False Positive Risk

Be cautious about over-classifying as urgent. Common false positive patterns:
- Emails from courts that are courtesy copies (for informational purposes only)
- Emails from opposing counsel that are informal or non-binding
- Client emails that are anxious but not time-sensitive
- Newsletter-type updates from legal services

The system should be designed to err on the side of surfacing items for review rather than missing urgent ones. A false positive that the user quickly dismisses is far less harmful than a missed deadline.

---

## Question 3: Tone and Format Conventions in Legal Communications

### Direct Answer

Yes, tone and format must vary by communication type. The legal preset should define distinct conventions for:
- **Client communications** — conversational, accessible, reassurance-focused
- **Court-directed documents** — formal, structured, compliant with local rules
- **Opposing counsel communications** — professional, strategic, minimal
- **Internal notes** — factual, complete, case-file focused

Jurisdiction-specific formatting should be handled via **templates**, not hardcoded into prompts, allowing users to select their jurisdiction's requirements.

### Detailed Analysis

#### A. Client Communications

**Tone:** Warm but professional. Clients are often stressed, scared, or frustrated. Communication should:
- Acknowledge their situation and emotions
- Explain next steps in plain language
- Provide realistic timelines without over-promising
- Avoid legal jargon or explain it when necessary

**Format:**
- Subject line: Clear and descriptive ("Update on Your Case — Smith v. Jones")
- Opening: Personalized greeting with client name
- Body: Action-oriented with clear next steps
- Closing: Contact information and availability reminder

**Example generated draft:**
> Dear Maria,
>
> I wanted to update you on the status of your divorce case. As you know, we filed our response to David's motion last week.
>
> The court has scheduled a hearing on April 15 at 9:00 AM. I'll be there to represent your interests. I'll call you the day before to review what to expect.
>
> Please don't hesitate to reach out if you have questions between now and then.
>
> Best regards,
> [Attorney Name]

**Key conventions:**
- Use client's name naturally
- Explain legal terms (parenting plan, discovery, etc.)
- Give realistic expectations
- Include next steps clearly
- Sign off warmly

#### B. Court-Directed Documents (Pleadings, Motions, Letters)

**Tone:** Formal, precise, respectful. The court is the audience, and the document must comply with procedural rules.

**Format varies by jurisdiction, but common elements:**
- Case caption at top (court name, case number, parties)
- Formal subject line or title
- Numbered paragraphs for factual allegations
- Signature block with bar number
- Certificate of service (required in most jurisdictions)

**Example generated draft (motion):**
> IN THE SUPERIOR COURT OF THE STATE OF ARIZONA
> IN AND FOR THE COUNTY OF MARICOPA
>
> JOHN SMITH,
> Plaintiff,
> v.
> JANE SMITH,
> Defendant.
>
> Case No. CV2024-012345
>
> DEFENDANT'S MOTION FOR CONTINUANCE
>
> COMES NOW the Defendant, Jane Smith, by and through undersigned counsel, and respectfully moves this Court for an Order continuing the hearing scheduled for March 15, 2024...

**Key conventions:**
- Follow local court rules for formatting
- Use proper legal citation format
- Number paragraphs consecutively
- Include required procedural headings
- Maintain professional, non-inflammatory language
- Never make assertions without evidentiary support

#### C. Opposing Counsel Communications

**Tone:** Professional, measured, strategic. These communications may be read by the court if disputes arise, so tone matters.

**Format:**
- Letter format typically
- Clear statement of purpose
- Professional but not overly friendly (maintains appropriate boundaries)
- Strategic — don't reveal strategy, but don't be adversarial unnecessarily

**Example generated draft:**
> Dear [Counsel]:
>
> Re: Smith v. Jones, Case No. CV2024-012345
>
> I am writing in response to your March 1 correspondence regarding discovery responses.
>
> We are in the process of compiling the requested documents and expect to serve our responses by the March 15 deadline. Please advise if an extension would be acceptable should we encounter difficulties in gathering certain medical records.
>
> Please do not hesitate to contact me if you have questions.
>
> Very truly yours,
> [Attorney Name]

**Key conventions:**
- Reference case by name and number
- State purpose clearly
- Avoid emotional language
- Keep tone professional but not hostile
- Document everything (these may become evidence)

#### D. Internal Case Notes

**Tone:** Factual, complete, objective. The note becomes part of the case file and may be reviewed by the attorney later or potentially discovered.

**Format:**
- Date and time
- Matter reference
- Subject line summarizing content
- Chronological detail of what occurred
- Action items or follow-ups needed

**Example generated draft:**
> DATE: March 10, 2024 2:30 PM
> MATTER: Smith v. Jones (CV2024-012345)
> SUBJECT: Telephone conference with opposing counsel re: discovery dispute
>
> Spoke with [Counsel] for opposing party re: their document production deficiency. [Counsel] indicated they are waiting on medical records from third-party provider.
>
> Agreed to extend production deadline by 14 days, pending their written confirmation.
>
> ACTION: Follow up by March 25 if written extension not received.

**Key conventions:**
- Be objective — don't editorialize
- Include all relevant details
- Note deadlines and follow-ups
- Use consistent format for easy retrieval

#### Jurisdiction-Specific Considerations

Rather than hardcoding jurisdiction requirements into prompts, the system should support:
1. **Template selection** — user selects their jurisdiction's preferred format for court documents
2. **User-editable templates** — attorney can customize the format to match their local practice
3. **Field mapping** — the LLM populates template fields appropriately

Key variations by jurisdiction:
- Caption format (varies significantly by state)
- Margins and spacing requirements
- Font requirements
- Signature block format
- Certificate of service format

For Phase 1 demo, include templates for a representative jurisdiction (e.g., California or Arizona) with notes that users can customize.

---

## Questions for Other SMEs

### For AI/NLP Architecture SME

- **Prompt accuracy for legal context:** For legal rules, how much precision is needed in extracted context (e.g., exact filing deadlines vs. "sometime next week")? Are there scenarios where ambiguous extraction is worse than no extraction? In legal practice, missing a specific date entirely is less dangerous than having a wrong date — the attorney will check the actual deadline. However, if the system suggests an incorrect date with confidence, it could lead to missed deadlines. Should extraction prompts be designed to return "unknown" rather than guess?

- **Legal disclaimer generation:** Should the action generation for legal communications include a disclaimer that the draft is AI-generated and requires attorney review? Is this a prompt-level instruction or a post-generation append?

- **Extraction confidence for dates:** For court-ordered deadlines, the date extraction is critical. Should the system have a confidence threshold below which it surfaces the item differently (e.g., "Court deadline detected — please verify date manually") rather than presenting an extracted date with low confidence?

### For UXDesigner

- **Urgency visualization:** Given that legal urgency classification is critical (deadlines, hearings), how should the dashboard visually distinguish critical, high, medium, and low urgency items? The dual accent system (amber/blue) seems designed for urgency vs. routine — should legal items use a different scheme, or is the existing scheme sufficient?

- **Email viewer for legal review:** For attorneys reviewing triggered rules, what is the optimal way to present the full email alongside the extracted context and suggested actions? Should the email be shown in a collapsible panel, a side-by-side view, or something else? Attorneys need to see the original to verify extracted context.

- **Legal deadline calendar integration:** Future phases may include calendar integration. How should we visualize detected deadlines in the dashboard — as a simple date, or with countdown ("3 days remaining") or both?

### For IntegrationEngineer

- **Entity linking for legal:** The HLRD mentions entity linking (email to case/client). For legal practice, the matching criteria should include case number patterns, client name matching, and possibly opposing counsel identification. What approach would you recommend for the matching logic — deterministic (exact match on case number) or probabilistic (fuzzy matching on names)? Legal data has specific patterns (case numbers like "CV2024-012345") that could enable deterministic matching.

- **Data retention for legal:** Client communications, case files, and court documents may have specific retention requirements. Should the system track evaluation history and action approvals differently for legal domain (longer retention, audit trail for malpractice defense) compared to other domains?

### For EcommerceDomain SME

- **Cross-domain patterns:** Are there any patterns in how e-commerce sellers handle urgent customer communications (returns, complaints) that have parallels in legal client communication? Specifically, the tension between responding quickly and responding correctly seems common to both domains.

---

## Summary of Recommendations

1. **Define rules by practice area** — include family law, personal injury, criminal defense, and general civil litigation variants for common rule types
2. **Classify urgency within rules** — critical, high, medium, low classifications embedded in each rule's evaluation
3. **Implement distinct tone templates** — client, court, opposing counsel, and internal note templates with different conventions
4. **Support jurisdiction via templates** — allow users to select court document formats rather than hardcoding
5. **Prioritize deadline detection** — this is the highest-stakes missed communication
6. **Design for false-positive tolerance** — better to surface and dismiss than miss
7. **Include client communication tracking** — unresponded client emails are a significant risk

---

## File Paths

Analysis output: `/home/csandfort/Documents/source/repos/adeo/planning-docs-output/phase-1/legal-domain-sme-analysis.md`