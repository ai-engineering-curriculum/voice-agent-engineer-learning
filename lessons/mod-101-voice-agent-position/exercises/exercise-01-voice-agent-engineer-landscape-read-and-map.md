# exercise-01: Voice Agent Engineer Landscape Read and Map

**Estimated effort:** 2 hours · **Output:** one map (table or diagram) + a one-page synthesis

## Objective

Internalise the six categories of employer from Chapter 1 by mapping a current set of real job postings and public product pages to the landscape — and defending your placements.

The deliverable is not a web crawl of vacancies. It is **your** reading of who is doing what, so that when a recruiter contacts you about "Voice AI Engineer" you can place the job in 30 seconds.

## Prerequisites

- [Chapter 1](../01-the-voice-agent-engineer-landscape.md).
- A browser and ~90 minutes of focused reading.

## Steps

1. **Collect raw material.** Find at least **eight** currently-posted job descriptions with a title in the family: "Voice AI Engineer", "Voice Agent Engineer", "Conversational Voice AI", "Speech AI Engineer", "Realtime Voice Engineer", or "Multimodal Agent Engineer". Spread them across at least **four of the six categories** in Chapter 1 (voice-AI pure-plays, orchestration platforms, telephony providers, frontier labs, applied-voice product companies, vertical-voice applications). Save a snapshot or archive link for each.

2. **Build the map.** Produce a table (minimum) or a diagram (preferred) with columns / lanes for the six categories. For each job you collected:
   - Place it in a category.
   - Note the one-line "centre of gravity" from the job description (what it asks for most insistently: pipeline latency, framework depth, telephony, model research, compliance, domain expertise…).
   - Note the stack mentioned (ASR, TTS, LLM, orchestration framework, transport).

3. **Justify ambiguous placements.** Pick the **two** postings that were hardest to categorise. Write one paragraph each on why — what would have to be true to move the posting to a different category.

4. **Write your one-page synthesis.** Answer three questions:
   - Which category currently generates the most volume of postings, as a share of the eight you collected?
   - Which category's postings most emphasise latency and which most emphasise compliance? Does that match the archetype interview questions in Chapter 1?
   - If you were applying today, which category would you preferentially target, and why?

5. **Sanity-check against the role.** Pick one posting and highlight (a) one requirement that belongs to this role (voice-agent engineer, L35), (b) one requirement that should arguably belong to an adjacent role (ASR research, conversation design, inference engineering…), and (c) one requirement that is a *red flag* — i.e. so broad that the hiring team has not thought the role through.

## Starter guidance

- Pure-plays' developer-advocacy or solutions-engineering roles are often voice-agent-engineer work with a different title.
- "Conversational AI Engineer" at a contact-centre company is usually in category 5 (applied-voice).
- A posting that lists DSP / signal-processing fundamentals, training infra, and PyTorch heavily is probably ASR / TTS research, not this role.
- Postings that focus on SIP, carriers, and five-nines reliability are IVR / CTI roles, not this one.
- Beware titles like "AI Engineer — Voice" at companies whose main product is *not* audio — they may be hiring an applied-AI engineer who happens to need to call a speech API.

## Acceptance criteria

- [ ] At least **eight** real, archived job postings, spanning at least **four** of the six categories.
- [ ] A map (table or diagram) that places each posting and labels its centre of gravity and stack.
- [ ] Two paragraphs justifying hard-to-categorise postings.
- [ ] One-page synthesis answering the three questions.
- [ ] One posting broken down into "this is us / this is adjacent / this is a red flag".
- [ ] All placements defensible from the posting text, not from brand assumptions.

## Stretch goals

- Repeat the exercise with the companies named in Chapter 1 that you did *not* see postings from. What does their absence suggest — team already built, hiring under a different title, platform not yet hiring externally?
- Add a second axis: experience level (IC / staff / lead). Does any category skew more junior or senior than the others?
- Correlate against public releases — which companies shipped a realtime voice product in the last six months, and does that predict who is hiring now?

## Keep it

Save the map. You'll reuse it in Module 112 when you evaluate what a well-run voice-agent team is measuring, and in the project-104 architectural-brief capstone when you position your design against industry norms.
