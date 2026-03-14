# UX Designer SME Analysis — Adeo

## Analysis Overview

The Adeo system presents a compelling dashboard design challenge: a triage tool for busy professionals who check it briefly between tasks, not a dashboard they stare at all day. The four UX questions below address the core interaction patterns that will define whether the system feels fast and trustworthy or overwhelming and slow.

**Assumed Tech Stack (per HLRD design system):**
- ShadCN components + Tailwind CSS
- Light theme with amber (urgency) / blue (routine) accent system
- CopilotKit for HITL primitives
- Card-based UI with stone-100 backgrounds and white card surfaces

---

## Question 1: Presentation of Multiple Triggered Rules and Actions

**How should multiple triggered rules and their actions be presented in the dashboard? Should they be grouped by rule, by email, by urgency, or by action type? What's the right information density for a professional who is scanning quickly between tasks?**

### Recommendation: Group by Urgency, Then by Email

**Primary grouping: Urgency level** (urgent items first, then routine), **Secondary grouping: Email thread** (each email showing all triggered rules/actions under it).

**Why this approach:**
- The amber/blue accent system already encodes urgency at a glance — urgent items have amber left borders and badges, routine items have blue. This is the most important differentiation and should drive the top-level sort.
- Grouping by email prevents the "split brain" problem where related actions from the same source appear in different locations. A cancelled hearing email might trigger both a "reschedule request" action and a "client notification" action — presenting them together lets the user see the full picture.
- For a solo practitioner scanning between calls, the cognitive load of mentally reassembling fragmented emails is too high.

**Information density recommendation: Single-card-per-action-item**

Each action item should be a compact card showing:
- Urgency badge (amber/blue pill)
- Timestamp (relative, e.g., "5 min ago" in monospace)
- Email subject line (truncated to 60 characters)
- Rule name that triggered (small, secondary text)
- Confidence indicator (optional — subtle dot or percentage)
- Associated entity (client name or case number, e.g., "Martinez v. Clark")
- Single primary CTA: "Review Actions" button

**Do not show:**
- Full email preview in the card (available on demand via expansion or navigation)
- Multiple actions pre-expanded (show the first, indicate "X more actions" with count)
- Detailed evaluation results (confidence breakdown, extracted context)

**Progressive disclosure pattern:**
1. Card shows summary (subject, entity, urgency badge)
2. Click/tap "Review Actions" opens a sheet or dialog with:
   - Full email content
   - All triggered rules with confidence scores
   - All suggested actions as individual options
   - Approve/Edit/Reject/Revise buttons per action

This keeps the main list scannable (cards are ~80-100px tall) while giving full detail exactly when needed.

**Layout structure:**

```
+--------------------------------------------------+
|  [Left Sidebar - 240px]  |  [Main Content Area] |
|  - Logo                  |  Header:             |
|  - Navigation:           |  "Action Items (12)" |
|    - Action Items*       |  [Filter by: All▼]   |
|    - Email History       |  [Sort: Urgency▼]    |
|    - Rules               |                      |
|    - Actions             |  +-----------------+  |
|    - Settings            |  | [URGENT CARD]  |  |
|                          |  | border-l-amber  |  |
|  [Domain: Legal ▼]       |  | Hearing cancel  |  |
|                          |  | Martinez v.Clark|  |
|                          |  | [Review Actions] |  |
|                          |  +-----------------+  |
|                          |                      |
|                          |  +-----------------+  |
|                          |  | [ROUTINE CARD]  |  |
|                          |  | border-l-blue   |  |
|                          |  | Client inquiry  |  |
|                          |  | Davis estate    |  |
|                          |  | [Review Actions]|  |
|                          |  +-----------------+  |
+--------------------------------------------------+
```

---

## Question 2: CopilotKit HITL vs. Custom UI for Revision

**CopilotKit's HITL components provide the approval/reject primitives, but the revision-with-feedback flow may need custom UI layered on top. What's the right balance between using CopilotKit's built-in components and building custom UI for the revision experience?**

### Recommendation: Use CopilotKit for Core Approval/Rejection, Custom Sheet for Revision Flow

**Use CopilotKit's built-in components for:**
- The simple Approve / Reject binary — this is CopilotKit's core HITL primitive and should be leveraged directly
- The state transitions: pending → approved → executed, and pending → rejected
- Integration with the CopilotKit runtime for agent communication

**Build custom UI for:**
- The "Edit" experience (in-place text editing of generated drafts)
- The "Revise with feedback" loop (where the user provides natural language feedback and receives a revised draft)

**Why this split:**

CopilotKit's HITL components handle the "machine proposes, human decides" pattern well. However, the revision-with-feedback flow is more nuanced — it requires:
1. A text area showing the current draft (editable)
2. A feedback input field for natural language instructions
3. A "Generate Revision" trigger
4. Display of the revised draft with ability to iterate further

This is fundamentally a multi-turn conversation pattern that CopilotKit's simple approval state doesn't naturally model. Building a custom revision sheet using ShadCN's `Sheet` or `Dialog` component gives full control over the experience.

**Proposed Revision Sheet UI:**

```tsx
<Sheet>
  <SheetTrigger>Review Actions</SheetTrigger>
  <SheetContent side="right" className="w-[600px]">
    <SheetHeader>
      <SheetTitle>Action: Draft Client Status Update</SheetTitle>
      <SheetDescription>
        Rule: Cancelled Hearing Detection (98% confidence)
      </SheetDescription>
    </SheetHeader>

    {/* Email context - collapsible */}
    <Accordion type="single" collapsible>
      <AccordionItem value="email">
        <AccordionTrigger>View triggering email</AccordionTrigger>
        <AccordionContent>
          <div className="bg-stone-50 p-4 rounded-md text-sm text-stone-700">
            [Email content here...]
          </div>
        </AccordionContent>
      </AccordionItem>
    </Accordion>

    {/* Draft editor */}
    <div className="mt-6">
      <label className="text-sm font-medium text-stone-700">Generated Draft</label>
      <Textarea
        className="mt-2 min-h-[200px]"
        defaultValue={generatedDraft}
      />
    </div>

    {/* Revision feedback */}
    <div className="mt-4">
      <label className="text-sm font-medium text-stone-700">
        Feedback for revision (optional)
      </label>
      <Input
        className="mt-2"
        placeholder="e.g., Make tone more formal, mention deadline is Jan 20"
      />
      <Button variant="secondary" className="mt-2">
        Generate Revision
      </Button>
    </div>

    {/* Action buttons */}
    <div className="mt-6 flex gap-3">
      <Button variant="outline" className="flex-1">
        Reject
      </Button>
      <Button className="flex-1 bg-amber-500 hover:bg-amber-600">
        Approve & Execute
      </Button>
    </div>
  </SheetContent>
</Sheet>
```

**Design rationale:**
- Using `Sheet` (side drawer) preserves context — the user can still see the dashboard behind while working on the draft
- Collapsible email context via Accordion keeps the interface clean but accessible
- Clear separation between "edit draft directly" and "provide feedback for AI revision" — two distinct workflows
- Feedback input is plain text (not a structured form) to encourage natural language
- Revision button is secondary-styled to encourage direct editing as the primary path

---

## Question 3: Rule Testing Interface

**How should the rule testing interface work? A simple "paste an email and see if it triggers" input, or a more sophisticated view that replays historical emails through the rule and shows results in a timeline?**

### Recommendation: Historical Replay View with Timeline

**The "paste an email" approach is useful for ad-hoc testing, but the historical replay view provides much more value for rule refinement.** The ideal solution offers both.

**Two-mode interface:**

**Mode 1: Quick Test (for rule creation/editing)**
- Textarea input: paste or type email content
- "Test Rule" button
- Result displayed inline:
  - Triggered: Yes/No (with confidence percentage)
  - Extracted context (dates, parties, case numbers)
  - Which actions would be suggested

**Mode 2: Historical Replay (for validation before going live)**
- Shows all emails in the simulated inbox
- Filters: All / Triggered / Not Triggered
- Click any email to see:
  - Full email content
  - Which rules were evaluated
  - Which rules triggered (highlighted)
  - What actions would be suggested
  - Side-by-side: what actually happened (if previously acted on)

**Why historical replay matters:**
- Users creating rules often have a specific scenario in mind ("I want to catch cancelled hearings"). Testing against one email tells you if it catches that one. Testing against 50 historical emails tells you if it catches the full variety of ways that scenario appears — and whether it generates false positives.
- The "dry run" mode mentioned in the HLRD maps naturally to this view: newly created rules can be run against historical data and show results without surfacing actions.
- Seeing patterns in false negatives ("it catches 'hearing cancelled' but misses 'hearing postponed'") helps users refine prompts.

**Implementation sketch:**

```
+------------------------------------------+
|  Rule: Cancelled Hearing Detection      |
|  [Quick Test] [Historical Replay]       |
+------------------------------------------+
|                                          |
|  Historical Replay Mode                  |
|  Filter: All (156) | Triggered (8) ...  |
|                                          |
|  +------------------------------------+  |
|  | Jan 12, 2025  9:30 AM              |  |
|  | From: clerk@court.gov              |  |
|  | Subject: Hearing Status — Case... |  |
|  | Status: ✓ Triggered (94%)          |  |
|  +------------------------------------+  |
|  +------------------------------------+  |
|  | Jan 10, 2025  2:15 PM              |  |
|  | From: opposing@counsel.com         |  |
|  | Subject: Motion to Continue       |  |
|  | Status: ✗ Not Triggered            |  |
|  +------------------------------------+  |
|                                          |
+------------------------------------------+
```

**Clicking an email expands details:**

```
+------------------------------------+
| Jan 12, 2025  9:30 AM   ✓ Triggered |
|                                        |
| Full email content:                   |
[collapsible text area]                |
|                                        |
| Rules evaluated:                      |
  - Cancelled Hearing: ✓ (94%)         |
  - Deadline Detection: ✗ (12%)         |
  - Client Inquiry: ✗ (8%)              |
|                                        |
| Would suggest actions:                |
  - Draft rescheduling request          |
  - Draft client status update          |
+------------------------------------+
```

---

## Question 4: Dashboard Layout for Solo Practitioner

**What's the right dashboard layout for a solo practitioner who checks this tool between client calls and court appearances? Should it be a single-page priority queue, or a multi-tab layout with separate views for action items, emails, and rule management?**

### Recommendation: Single-Page Priority Queue with Persistent Sidebar Navigation

**Single-page priority queue, not multi-tab layout.**

**Rationale:**

The usage pattern described — "checking between client calls and court appearances" — is fundamentally different from a dashboard that's open all day. The user:
- Opens the app, scans for urgent items, acts on them, closes the app
- Has 30-60 seconds per session
- Needs to see urgency hierarchy immediately without clicking tabs

Multi-tab layouts (Action Items | Email History | Rules | Actions | Settings) introduce friction:
- Each tab click is a decision point
- Tabs imply the user will spend time exploring each — but these users won't
- Moving between tabs loses context of what was just reviewed

**Proposed layout:**

```
+------------------+---------------------------------------+
| SIDEBAR (fixed)  |  MAIN AREA                            |
|                  |                                       |
| Adeo             |  [Header bar]                         |
|                  |  Action Items (urgent: 3, routine: 9)|
| [Domain: Legal]  |  [Search] [Filter] [Sort]             |
|                  |                                       |
| Navigation:      |  +---------------------------------+  |
| • Action Items * |  | [URGENT CARD]                   |  |
|   (badge: 3)     |  | Hearing cancelled - Martinez    |  |
| • Email History  |  | [Review Actions]                |  |
| • Rules          |  +---------------------------------+  |
| • Actions        |                                       |
| • Settings       |  +---------------------------------+  |
|                  |  | [URGENT CARD]                   |  |
|                  |  | Court order received - #2024-X |  |
|                  |  | [Review Actions]                |  |
|                  |  +---------------------------------+  |
|                  |                                       |
|                  |  +---------------------------------+  |
|                  |  | [ROUTINE CARD]                  |  |
|                  |  | Client follow-up - Davis        |  |
|                  |  | [Review Actions]                |  |
|                  |  +---------------------------------+  |
+------------------+---------------------------------------+
```

**Key design decisions:**

1. **Single primary view: Action Items** — This is where the user spends 90% of their time. The badge shows pending count.

2. **Sidebar navigation is persistent, not tab-based** — The sidebar is always visible. Clicking "Rules" or "Actions" navigates to a full page, but the primary experience is the Action Items queue.

3. **Urgency counts in header** — Shows "Action Items (urgent: 3, routine: 9)" so the user knows immediately how much attention is needed.

4. **Filter and sort are inline in header** — Not a separate "view" but quick adjustments to the current list.

5. **Email History is secondary** — Accessed via sidebar, but not a primary workflow. It's there for audit ("wait, what happened with that email?") rather than daily use.

6. **Rules and Actions management are tertiary** — These are setup tasks done once, not daily workflows. They live in the sidebar but the user rarely clicks them after initial setup.

**Breakpoint considerations:**
- Desktop (primary): Full sidebar + main content
- Tablet: Collapsible sidebar (hamburger menu), full-width content
- Mobile: Bottom navigation bar with 3-4 key items, stacked card layout

However, the HLRD notes mobile is out of scope for Phase 1, so desktop-first design is appropriate.

---

## Cross-Cutting UX Considerations

**Loading states:** The HLRD specifies performance requirements (5s rule evaluation, 10s action generation). However, even within those limits, loading states matter:
- Action items should have skeleton loaders when refreshing
- "Review Actions" sheet should show a spinner while generating drafts
- Rule testing should show progress ("Evaluating rule...")

**Empty states:** When no action items exist (all caught up):
- Friendly message: "All caught up — no action items pending"
- Optionally show last activity timestamp ("Last item cleared 2 hours ago")
- Do not show empty table with column headers

**Error states:** When rule evaluation fails:
- Show item with error indicator (red border, "Evaluation failed — tap to retry")
- Do not block the queue; allow user to proceed and address errors separately

---

## Questions for Other SMEs

**For AI/NLP Architecture SME:**
- How long does rule evaluation actually take in practice with the specified model families (Kimi, Qwen, MiniMax, GLM)? The 5-second requirement is a target, but what's realistic? This affects whether we can show live evaluation in the UI or must cache results.

**For Integration Engineer SME:**
- The HLRD mentions "simulated data sources" seeded with synthetic data. How realistic should the email subjects and body content be for testing the UX? Should we invest in LLM-generated synthetic emails for more varied edge cases, or is hand-crafted seed data sufficient for demonstrating the dashboard patterns?

**For AI/NLP Architecture SME:**
- When a rule triggers multiple actions, does the system evaluate them in parallel or sequentially? Action generation latency directly affects whether opening a "Review Actions" sheet feels instant or takes several seconds.

**For Legal/Social/Ecommerce Domain SMEs:**
- For your respective domains, what's the realistic volume of action items per day for a solo practitioner/small team? Is the dashboard likely to have 5 pending items or 50? This affects whether we need pagination, infinite scroll, or can simply show all items.
