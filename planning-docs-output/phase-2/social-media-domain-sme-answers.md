# Phase 2b: Social Media Domain SME Answers

## Question from AI/NLP Architecture SME

### Trend Response Timing and Approval Windows

**Question:** For viral moments, is the 30-minute approval window realistic, or should we design a "fast track" mode that surfaces the raw LLM output with minimal formatting for faster approval?

**Answer:**

The 30-minute approval window is **not realistic for viral moments**. This aligns directly with my Phase 1 analysis where I addressed this exact tradeoff (Question 4: Speed vs. Quality Tradeoff for Trend Response).

**The core problem:** Trends decay rapidly. The engagement window is 0-2 hours for peak impact, diminishing quickly after that. A response approved at 30 minutes has already lost significant value compared to one approved in 5 minutes.

**My recommendation from Phase 1 was a tiered approach:**

| Mode | Generation Time | Expected Approval Window | Use Case |
|------|-----------------|--------------------------|----------|
| **Fast Draft** | 3-5 seconds | 2-5 minutes | Viral moments, crisis, influencer tags |
| **Standard Draft** | 8-12 seconds | 15-30 minutes | Routine mentions, review responses |
| **Enhanced Draft** | 15-20 seconds | 1-2 hours | Client campaigns, strategic content |

**For the specific "fast track" architecture question:**

The system should implement:

1. **Minimal-format surface for Fast Draft mode** — Raw text with basic formatting (line breaks preserved), no rich media suggestions, no multiple option generation. Just the response text and a single "One-tap approve" button.

2. **Pre-approved template fallback** — If LLM latency exceeds threshold (e.g., >5 seconds), fall back to 2-3 brand-approved templates that can be deployed immediately. Templates should be pre-loaded per brand.

3. **Auto-escalation triggers** — Fast Draft should automatically apply when:
   - Trend velocity is spiking (mention volume increasing rapidly)
   - Post is <2 hours old
   - Mention is from high-follower account (>10K followers)
   - Sentiment is negative (complaint/crisis potential)

4. **Mobile-first approval for Fast Draft** — The approval action must work with a single tap on mobile. The full revision UI is desktop-only; mobile gets approve/reject only.

**Technical implications:**

- Fast Draft should skip the revision cycle entirely — generate once, present for approval
- Standard Draft includes one auto-revision cycle (if first output needs tone adjustment)
- Enhanced Draft supports full multi-turn revision

**The key insight:** For social media, the "ship it" path is more important than the "polish it" path. Design the UX for the 80% case (approve immediately) rather than the 20% case (needs revision).

---

## Summary

Yes, implement a "fast track" mode. The 30-minute window is realistic for Standard Draft content (routine mentions, review responses), but viral moments require sub-5-minute approval. Fast Draft mode should surface raw LLM output with minimal formatting and a single-tap approval interface.
