# Social Media Domain SME Analysis

## Executive Summary

This analysis addresses four critical questions about the social media management preset for the Adeo system. The recommendations are grounded in the operational realities of solo social media managers and small agencies (2-5 people) who juggle multiple brand accounts with limited time and resources.

---

## Question 1: Missed Moment Scenarios and Response Speed

**Question:** What are the most common "missed moment" scenarios for social media managers and agencies? Is trend response speed (minutes) realistic as a use case, or is the real value in not missing overnight/weekend activity?

### Analysis

**Common Missed Moment Scenarios:**

1. **Overnight/Weekend Negative Mentions** — The most damaging missed moments are negative mentions that sit for 12+ hours before being seen. A customer complaint posted at 11 PM that isn't seen until 9 AM the next day has already been viewed by hundreds of potential customers and potentially amplified. The damage compounds when the complaint is justified — silence reads as indifference.

2. **Influencer Engagement Opportunities** — When a mid-tier influencer (10K-500K followers) mentions a brand, there's a narrow window (2-4 hours) for engagement before the post loses momentum. Missing this window means losing free exposure that competitors could have capitalized on.

3. **Crisis Amplification Detection** — A single negative mention rarely spirals into a crisis. But when 5-10 similar complaints surface within hours, that's an early warning signal. Missing this pattern means the first response comes too late — after the narrative has already shifted.

4. **Client Campaign Requests** — Agency managers frequently miss time-sensitive client emails requesting content changes, approvals, or new campaign launches because they're buried in inboxes alongside routine correspondence.

5. **Review Site Notifications** — New reviews on Google, Yelp, Trustpilot, or G2 that need responses. A 1-star review sitting for 48+ hours damages local SEO and scares off potential customers.

6. **Competitive Activity** — Competitors launching campaigns, changing pricing, or having a viral moment that creates a response opportunity.

**Speed Reality Check:**

| Scenario | Realistic Response Window | Notes |
|----------|---------------------------|-------|
| Negative complaint response | 1-4 hours | Critical for damage control |
| Influencer engagement | 2-4 hours | Before post momentum fades |
| Positive trend riding | 30 min - 2 hours | Viral moments decay fast |
| Review response | 24 hours | Platform algorithms favor quick responses |
| Client email | 24-48 hours | Business hours acceptable |

**The key insight:** True "minutes" response is rarely realistic for solo managers or small agencies unless they have dedicated weekend coverage or notification systems. The real value proposition is:

- **Not missing overnight/weekend activity entirely** — knowing it exists in the morning
- **Prioritization** — knowing which items need immediate attention vs. can wait until Monday
- **Draft generation** — having a response ready to go rather than starting from scratch

A system that flags overnight activity at 7 AM with a pre-written draft is more valuable than one that promises minute-level response but only works during business hours.

### Recommendations

1. **Urgency classification should be automatic** — The system should distinguish between "needs response NOW" (negative mentions, influencer tags) and "needs response today" (routine mentions, review alerts)

2. **Time-window monitoring is essential** — The trend detection should explicitly track when content was posted and surface time-decay information (e.g., "Posted 6 hours ago — engagement window closing")

3. **Weekend/overnight alerts need special handling** — Consider a "critical alert" threshold that could trigger push notifications vs. a "morning review" batch for non-urgent items

---

## Question 2: Keyword Configuration for Trend Monitoring

**Question:** How should keyword configuration work for trend monitoring? Should users provide a flat keyword list, or should the system support boolean logic (e.g., "AI AND regulation NOT crypto")? Should the system suggest related keywords based on the brand's industry?

### Analysis

**Recommendation: Start Simple, Add Complexity as Needed**

For solo managers and small agencies, complexity is a barrier. The system should support a graduated complexity model:

**Tier 1: Simple Keyword List (Default)**
- Flat list of keywords/phrases to monitor
- "AI, machine learning, automation, chatbot"
- System matches any mention containing these terms
- Easiest to configure, understand, and maintain

**Tier 2: Basic Boolean Logic (Power Users)**
- AND, OR, NOT operators
- "AI AND regulation" — must contain both
- "AI NOT crypto" — contains AI but not crypto
- For managers who understand their industry well enough to filter noise

**Tier 3: Industry-Suggested Keyword Clusters**
- System provides suggested keywords based on industry vertical
- User selects/deselects from recommendations
- Reduces setup friction for new users

### Industry Keyword Suggestion Examples

| Industry | Suggested Keyword Clusters |
|----------|----------------------------|
| Tech/SaaS | "AI, machine learning, automation, API, integration, software, cloud, SaaS" |
| Legal | "court, filing, lawsuit, settlement, deposition, litigation, attorney" |
| Healthcare | "patient, treatment, clinical, provider, insurance, HIPAA, medical" |
| Food/Restaurant | "recipe, ingredients, dining, restaurant, food safety, chef" |
| Real Estate | "mortgage, interest rate, listing, buyer, seller, market, inventory" |

### Boolean Logic Implementation Considerations

Boolean logic adds meaningful capability but introduces:

1. **User education burden** — Many solo managers don't think in boolean terms
2. **Syntax errors** — "AI AND regulation NOT" is invalid
3. **Unexpected results** — "brand" AND "launch" might match "doesn't launch" as easily as "launches"

**Recommendation:** Implement boolean support but:
- Provide a simple UI that hides the syntax (checkboxes/toggles)
- Show live preview of how many tweets would match
- Offer templates for common patterns

### Keyword Suggestions Should Include

- **Related terms** — Not just exact matches
- **Spelling variations** — "colour" vs "color"
- **Hashtag alternatives** — Monitor both #AI and "AI" (with hashtag)
- **Competitor names** — Optional, for competitive intelligence

---

## Question 3: Tone and Voice Conventions for Generated Content

**Question:** What tone and voice conventions should the system follow when generating social media content? Should the preset include a configurable brand voice profile (formal, playful, edgy, etc.) that shapes all generated posts? How do we prevent generated content from sounding generic or bot-like?

### Analysis

**The Critical Problem with Generic AI Content**

Generic AI-generated social media content has several failure modes:

1. **Corporate void language** — "We're excited to announce..." "Thank you for reaching out..." that says nothing
2. **Missing brand personality** — A playful snack food brand sounds like a law firm
3. **Safe but forgettable** — Technically correct but no one remembers it
4. **Tone deafness** — Responding to a complaint with "We're so glad you shared your experience!"

### Brand Voice Profile Structure

**Recommendation: Configurable Brand Voice Profile with Multiple Dimensions**

The preset should include a brand voice configuration with these dimensions:

| Dimension | Range | Example |
|-----------|-------|---------|
| Formality | Casual ←→ Formal | "Hey folks!" vs "Dear valued customers" |
| Humor | Serious ←→ Playful | Direct response vs. witty comeback |
| Personality | Reserved ←→ Bold | understated claims vs. enthusiastic declarations |
| Expertise | Beginner-friendly ←→ Expert | Explains concepts vs. assumes knowledge |
| Empathy | Cool ←→ Warm | Acknowledges vs. over-promises |

### Pre-Built Voice Profiles

Provide templates that map to common brand positioning:

- **Professional/Corporate** — Formal, warm, expert, empathetic
- **Startup/Tech** — Casual, playful, bold, expert
- **Lifestyle/Consumer** — Casual, playful, warm, beginner-friendly
- **Authority/Expert** — Formal, serious, expert, reserved
- **Edgy/Youth** — Casual, playful, bold, irreverent

### Preventing Bot-Like Content

The most important factor is **specificity**. Generic content is vague; good content references specific details.

**Action prompts should require:**
- Specific details from the mention (product name, issue, user's tone)
- Brand-specific language (inside jokes, preferred terms, brand catchphrases)
- Platform-native formatting (short for Twitter, visual focus for Instagram)

**Prompt engineering approach:**
```
You are writing as [BRAND NAME], a [INDUSTRY] brand with a [VOICE DESCRIPTION].
The user's mention is: "[EXACT TEXT]"

Write a response that:
- References something specific they said
- Uses our brand voice: [SELECTED PROFILE]
- Is [APPROXIMATE LENGTH] words
- Includes [OPTIONAL: relevant hashtag]
- Matches the user's sentiment: [DETECTED SENTIMENT]
```

**Tone per Sentiment (Non-Negotiable):**

| Mention Sentiment | Required Tone | Example |
|-------------------|---------------|---------|
| Positive/Enthusiastic | Match or exceed their energy | "Thanks for the love! We're thrilled to be part of your journey" |
| Neutral/Informational | Warm but brief | "Thanks for the tag! Here's the link: [url]" |
| Negative/Complaining | Empathetic, solution-oriented | "We're sorry to hear this. DM'ing you now to make it right" |
| Question | Helpful, direct | "Great question! Here's the answer..." |

### Platform-Specific Considerations

- **Twitter/X** — Shorter, can use thread for longer responses, quote tweets add context
- **Review sites (Google, Yelp)** — More formal, never defensive, offer to take offline
- **Trustpilot/G2** — B2B tone, more formal, reference product specifics
- **Instagram** — Visual focus, stories for quick responses, DMs for issues

---

## Question 4: Speed vs. Quality Tradeoff for Trend Response

**Question:** How should the system handle the speed/quality tradeoff for trend response posts? A trend response that takes 30 minutes to approve is often worthless. Should there be a "fast draft" mode that prioritizes speed over polish?

### Analysis

**The Decay Rate of Trend Relevance**

Trends follow a predictable decay pattern:
- **Hour 0-2:** Peak engagement window — response here maximizes reach
- **Hour 2-6:** Still valuable but diminishing returns
- **Hour 6-24:** Response likely seen as "jumping on the bandwagon"
- **Hour 24+:** Generally too late for meaningful engagement

A response drafted in 5 minutes and approved in 10 is worth more than a perfect response approved in 35 minutes.

### Recommended Approach: Tiered Draft Modes

**Mode 1: Fast Draft (Default for Trending)**
- Generates response in 3-5 seconds
- Simpler prompt, less refinement
- Focuses on: correct sentiment, key point, brand voice, relevant hashtag
- Approval expected within 2-5 minutes
- Appropriate for: trend responses, time-sensitive engagements

**Mode 2: Standard Draft**
- Generates response in 8-12 seconds
- More nuanced prompt with platform conventions
- Includes: specific reference, value-add, call-to-action
- Approval expected within 15-30 minutes
- Appropriate for: routine mentions, review responses

**Mode 3: Enhanced Draft**
- Generates response in 15-20 seconds
- Highest quality with multiple options
- Includes: main response, alternative angles, suggested visuals
- Approval expected within 1-2 hours
- Appropriate for: client campaign content, strategic posts

### How the System Should Decide

**Auto-select mode based on:**
1. **Trend velocity** — If trending volume is spiking, suggest Fast Draft
2. **Time since post** — Older posts default to Standard/Enhanced since the window is already lost
3. **Follower count** — High-follower accounts (influencers) get Fast Draft priority
4. **Sentiment** — Negative mentions get Fast Draft with escalation path

### Speed Optimizations

1. **Pre-populated approval** — Fast Draft should have "One-tap approve" option
2. **Minimal editing UI** — Fast Draft shows raw text with 2-3 quick-edit buttons (tone adjust, lengthen, shorten)
3. **Template fallback** — If LLM is slow, offer 3 pre-approved templates the brand has used before
4. **Smart defaults** — Most users will just approve; optimize for that path

### The "Morning Review" Alternative

For overnight/weekend activity:
- Flag items with "Review when available" rather than "Respond NOW"
- Generate Standard Draft that can be reviewed in the morning
- Send notification at start of business day with summary

This respects that most solo managers aren't actually checking dashboards at 2 AM, but they do want to see what they missed and have drafts ready to go.

---

## Questions for Other SMEs

### For AI/NLP Architecture SME:

1. **Model selection for social media content generation:** Given the requirement for fast drafts (3-5 seconds), which of the available model families (Kimi, Qwen, MiniMax, GLM) offers the best speed/quality tradeoff for generating short-form social media responses? Should we use different models for Fast vs. Enhanced modes?

2. **Sentiment detection accuracy:** How reliably can the available models detect sentiment in short, informal social media text (slang, emojis, sarcasm)? Should sentiment detection be a separate model call or integrated into the generation prompt?

3. **Token budget optimization:** For brand voice injection, what's the maximum context size we should dedicate to voice profile examples before it degrades generation quality vs. simply repeating the same guidance?

### For UX Designer SME:

1. **Dashboard scan patterns:** When a social media manager opens the dashboard in the morning, what's the optimal information hierarchy? Should urgent items be first (amber border), or should time-based grouping show overnight activity separately?

2. **Mobile responsiveness:** Given that managers often check mentions on phones between meetings, what's the minimum viable mobile experience? Can approval happen on mobile, or is it review-only?

3. **Notification design:** Should the system send push notifications for certain triggers (negative mentions, influencer tags), or is in-app notification sufficient? What's the notification fatigue risk for a tool checked multiple times daily?

### For Integration Engineer SME:

1. **Twitter/X API simulation:** For the demo, how should simulated Twitter data be structured to realistically represent API responses? Should we include engagement metrics (likes, retweets) as part of the trigger evaluation?

2. **Historical trend data:** The system needs to detect "spikes" in mentions. What's the appropriate time window for spike detection (1 hour? 6 hours? 24 hours)? Should this be configurable per brand?

---

## Summary of Recommendations

| Feature | Recommendation |
|---------|----------------|
| Trend response speed | Prioritize overnight/weekend detection over minute-level response |
| Keyword configuration | Start with flat list, add boolean support as tier 2, industry suggestions as tier 3 |
| Brand voice | Configurable multi-dimensional profile with pre-built templates |
| Speed vs. quality | Three-tier draft mode (Fast/Standard/Enhanced) with auto-selection based on urgency |
| Sentiment handling | Non-negotiable tone mapping per sentiment category |
| Platform conventions | Separate prompt variants for Twitter, review sites, email |

These recommendations prioritize the reality that solo social media managers need **practical help** managing volume, not a tool that promises enterprise-level speed but requires enterprise-level setup.
