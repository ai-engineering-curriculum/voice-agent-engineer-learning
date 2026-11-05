# exercise-01: STT Provider Decision Matrix

**Estimated effort:** 2 hours · **Output:** a filled-in provider scorecard + a one-page recommendation

## Objective

Produce a defensible provider decision for a **named deployment** of your choice — real or hypothetical, but with concrete constraints. By the end you will have a scorecard that a product org can actually decide against, and a one-page recommendation that says which provider, under which assumptions, for which phases.

This is the chapter 1 skill applied to a specific context, not a provider comparison in the abstract.

## Prerequisites

- [Chapter 1 — The STT Provider Landscape and the Scoring Decision](../01-stt-provider-landscape-and-scoring.md).
- Browser access to the provider documentation pages linked in Chapter 1.
- No code required.

## The deployment brief

Pick one of these three. Each is deliberately under-specified to force you to make (and name) your own assumptions:

- **Brief A — EU-regulated health-insurance renewal agent.** Inbound phone calls, 70 % English + 30 % German, must handle medical and policy terminology, must store transcripts in the EU, HIPAA-equivalent compliance posture required. Volume: 500k minutes/month at steady state.
- **Brief B — English-language consumer-support assistant for a US mobile app.** Mixed PSTN (existing phone number) and WebRTC (in-app voice) ingress. Volume: 50k minutes/month. Needs good number / amount recognition (users talk about dollars and order numbers often). Compliance: PCI-DSS SAQ-A for the payment turns.
- **Brief C — Field-service technician dictation.** Non-interactive. Technicians record 1–3 minute voice notes at the end of each job, offline when needed, synced when connection returns. Accuracy over latency. 30k minutes/month. On-prem preferred — field devices live in warehouses, mines, oil rigs.

Write a one-paragraph expansion of your chosen brief that names any assumptions you're adding (specific accent mix, specific languages, specific vendor-neutrality constraints). Post this as the top of your deliverable.

## Steps

### Part A — Shortlist (30 minutes)

1. From the Chapter 1 landscape, shortlist **4–6 providers** that are plausibly in scope for your brief. Do not dismiss anyone yet; include open-weights + managed.
2. For each shortlisted provider, cite one dated provider doc page that establishes they *can plausibly serve the brief* — correct languages, correct deployment posture, correct feature surface. If the page says they cannot, they're not shortlisted.
3. Before you score anything, **run the deployment-posture decision tree from Chapter 1** against your brief. Which row is this project on? Document the answer.

### Part B — Fill the scorecard (60 minutes)

4. Use the scorecard template from Chapter 1. Add any axes specific to your brief (e.g., "offline / on-device support" for Brief C; "German medical vocabulary quality" for Brief A).
5. Mark compliance and deployment-posture cells **first**. Red / fail / blocking cells eliminate providers before quality matters.
6. For every remaining axis, fill in:
   - A short factual statement from the provider's dated documentation (archive the URL and date).
   - Where the information is not publicly documented (common for latency / per-domain WER), write "requires benchmark" rather than guessing.
7. Flag the top 2–3 "requires benchmark" gaps you'd prioritise in a follow-up POC — because you cannot decide without those numbers.

### Part C — The recommendation (30 minutes)

8. Write a **one-page recommendation** answering:
   - **Primary provider**, under what assumptions, with what deployment posture.
   - **Fallback provider** (what you would switch to if the primary fails on a specific axis — name the axis).
   - **Phasing.** "MVP with Y; migrate to X once we exceed 100k minutes / month." Include a trigger, not a date.
   - **POC plan.** Which 2–3 "requires benchmark" gaps you would run before committing. Methodology sketch for each.
   - **Open risks.** 3–5 bullets you cannot resolve from documentation alone; what would move each one.

## Starter guidance

- Treat provider marketing pages as a secondary source. Use the docs portal for every factual claim, and date it. The number on a competitor-comparison PDF is almost always cherry-picked.
- If the brief is EU-regulated (A), your blocking filter kills about half the shortlist before you score anything. Good — that's the filter working.
- For latency axes where the provider doesn't publish a number, write "requires benchmark" rather than inventing one. Your acceptance criterion is "defensible," not "complete."
- Keep an eye on the SDK-stability tie-breaker. If two providers score the same on paper but one has a streaming SDK that has changed three times in a year, that's an axis you are allowed to name.

## Acceptance criteria

- [ ] Brief expansion paragraph at the top, with assumptions named.
- [ ] Deployment-posture decision-tree row identified and justified.
- [ ] 4–6 providers shortlisted; each with a dated doc link establishing basic plausibility.
- [ ] Scorecard filled; compliance / deployment-posture cells marked first and used as blocking filters.
- [ ] 2–3 "requires benchmark" gaps flagged, with a POC methodology sketch.
- [ ] One-page recommendation: primary + fallback + phasing (with trigger) + POC plan + open risks.
- [ ] Every factual claim backed by a dated URL.

## Stretch goals

- **Run the matrix for all three briefs.** The shape of the scorecard changes drastically depending on the compliance posture and the latency regime. The exercise of filling it three times is where the Chapter 1 skills actually consolidate.
- **Add a "cost per month at steady state" row** with explicit arithmetic — minutes × rate card × any add-ons (diarisation, redaction, custom-model hosting). Multiple providers have per-feature pricing that adds up differently; a steady-state cost is the number your finance team cares about.
- **Prepare the one-page recommendation as if you were presenting to a VP.** Three bullets of context, the recommendation, the trade-offs. Three slides maximum. Most L35 provider picks end up on a slide like this.
- **Identify one axis where your brief should pull in a non-obvious direction.** E.g., for Brief C, the right choice might be self-hosted Whisper despite the ops cost, because the on-prem / offline constraint rules out managed services. The ability to justify the uncomfortable choice is where the scorecard earns its keep.

## Keep it

The scorecard template, lightly specialised to your org, becomes your reusable artefact. Every time a provider releases a major version, update the cells with new dated links and re-run the recommendation.
