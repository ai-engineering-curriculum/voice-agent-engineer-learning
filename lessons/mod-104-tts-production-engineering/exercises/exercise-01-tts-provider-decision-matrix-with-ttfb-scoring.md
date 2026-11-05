# exercise-01: TTS Provider Decision Matrix With TTFB Scoring

**Estimated effort:** 2 hours · **Output:** a filled-in provider scorecard with real TTFB measurements + a one-page recommendation

## Objective

Produce a defensible TTS provider decision for a **named deployment** of your choice — real or hypothetical, but with concrete constraints. By the end you will have a scorecard that includes **measured TTFB numbers** (not vendor marketing claims) for the providers you shortlist, and a one-page recommendation that says which provider, under which assumptions, for which phases.

This is Chapter 1 applied to a specific context. Where the Module 103 STT decision matrix put WER front-and-centre, this one puts TTFB front-and-centre — the latency metric that decides whether the agent feels alive.

## Prerequisites

- [Chapter 1 — The TTS Provider Landscape and the TTFB-First Scoring Decision](../01-tts-provider-landscape-and-scoring.md).
- Browser access to the provider documentation pages linked in Chapter 1.
- A developer account at 2–3 shortlisted vendors with a generous free tier (OpenAI TTS, ElevenLabs, Cartesia, PlayHT, and Google Cloud all qualify as of this writing).
- Python 3.10+ with the vendor SDKs you choose + a way to measure wall-clock reliably.

## The deployment brief

Pick one of these three. Each is deliberately under-specified to force you to make (and name) your own assumptions:

- **Brief A — EU-regulated health-insurance renewal agent.** Outbound and inbound phone calls, English + German, must speak drug and policy terminology clearly, must store generated audio in the EU, HIPAA-equivalent compliance posture. Latency ceiling: 300 ms TTFB p95. Volume: 1M synthesised characters per day.
- **Brief B — Consumer companion app.** In-app voice (WebRTC), US + UK English, expressive and empathic (the voice is part of the brand), latency ceiling: 250 ms TTFB p95. Users expect a consistent recognisable voice; one brand voice across all turns. Volume: 300k characters per day.
- **Brief C — IVR + in-app field-service assistant.** Mixed PSTN and WebRTC ingress, English only, numbers and work-order IDs read correctly, compliance: SOC 2. Latency ceiling: 400 ms TTFB p95 (PSTN tolerates more). Volume: 2M characters per day. Must handle a vocabulary of ~500 custom equipment names, model numbers, and site codes.

Write a one-paragraph expansion of your chosen brief that names any assumptions you're adding (specific locales, specific voice-identity requirements, specific SSML tags needed). Post this as the top of your deliverable.

## Steps

### Part A — Shortlist (20 minutes)

1. From the Chapter 1 landscape, shortlist **4–6 providers** plausibly in scope for your brief. Include at least one latency-first managed vendor (Cartesia, ElevenLabs Flash, OpenAI `tts-1`), one enterprise managed vendor (Azure, Google Chirp 3 HD, Polly), and one you would not pick but want to score as a sanity check.
2. For each shortlisted provider, cite one dated provider doc page that establishes they can plausibly serve the brief (correct locales, correct deployment posture, correct feature surface).
3. Run the Chapter 1 deployment-posture decision tree against your brief. Which row applies? Which providers it eliminates upfront.

### Part B — Measure TTFB (45 minutes)

4. For each shortlisted provider **you have account access to**, write a minimal streaming client that:
   - Sends a representative text payload (15–20 utterances drawn from realistic LLM outputs for your brief — greetings, numbers, a longer apology, a drug/product-name passage, a date/amount read-out).
   - Records the wall-clock timestamp **when the first request byte is sent** and **when the first audio byte is received**.
   - Repeats each utterance 20 times (warm-connection — same WS session or same keep-alive'd HTTP pool) and 20 times (cold-connection — new session per utterance).
   - Logs the samples to a CSV.

5. Report, per provider and per utterance class:
   - TTFB p50, p95, and p99 (cold and warm separately).
   - Realtime factor (audio-seconds-produced / wall-clock-synthesis-seconds) on at least one longer utterance.
   - Whether the provider underruns your player on a realistic jitter profile (if you ship the player, pipe through 50 ms + 30 ms-RMS artificial jitter).

6. **Where you cannot get account access**, write "requires benchmark" in the cell. Do not invent numbers.

### Part C — Fill the scorecard (30 minutes)

7. Use the scorecard template from Chapter 1. Add any axes specific to your brief (e.g., "German medical-vocabulary quality" for Brief A; "sustained voice-identity across registers" for Brief B).
8. Mark compliance and deployment-posture cells **first**. Red / fail / blocking cells eliminate providers before anything else matters.
9. Populate your measured-TTFB cells with actual numbers (not "fast"). Flag the 2–3 utterance classes where any provider missed your latency target by more than 20%.
10. Populate SSML support cells by **actual test**: synthesise a short clip with and without `<break>`, `<phoneme ipa>`, `<say-as interpret-as>`, and the one or two tags your brief most needs. Record "honoured" / "silently dropped" / "errored."

### Part D — Recommendation (25 minutes)

11. Write a **one-page recommendation** answering:
    - **Primary provider**, under what assumptions, with what deployment posture.
    - **Fallback provider** and the specific axis that would trigger the switch.
    - **Phasing.** "MVP with Y; migrate the EU workload to X in VPC once we exceed 500k char/day." Include a trigger, not a date.
    - **POC plan.** Which 2–3 "requires benchmark" or "needs listening panel" items you would resolve before committing. Methodology sketch for each.
    - **Open risks.** 3–5 bullets you cannot resolve from documentation + the TTFB probe alone (e.g., sustained voice-identity under load, mid-stream cancel latency in production, cost at steady state).

## Starter guidance

- **Measure TTFB from where your service will run, not your laptop.** If you cannot colocate a probe in the deployment region, say so explicitly and caveat the numbers.
- **Warm the connection.** Vendor TTFB numbers are almost always for warm-connection conditions. Report cold and warm separately; the delta tells you whether you need session-reuse engineering.
- **Use realistic text.** "Hello" is not a benchmark. Pull actual example LLM outputs for your brief, or a reasonable facsimile.
- **Test the SSML tags, don't trust the docs.** The shortest path to a nasty integration surprise is assuming `<phoneme>` works and discovering during UAT that your chosen voice silently drops it.
- **Barge-in latency belongs on the matrix.** If mid-stream cancel takes 400 ms to actually stop, that's a Module 106 regression; find it now.
- **Treat vendor marketing TTFB as a working hypothesis to disprove, not a datum.** Vendors publish numbers in favourable conditions; your conditions will differ.

## Acceptance criteria

- [ ] Brief expansion paragraph at the top, with assumptions named.
- [ ] Deployment-posture decision-tree row identified and justified.
- [ ] 4–6 providers shortlisted; each with a dated doc link.
- [ ] Measured TTFB p50 / p95 / p99 (cold + warm) per provider per utterance class — for every provider you have account access to.
- [ ] Realtime factor measured on at least one longer utterance per provider.
- [ ] SSML tag support tested empirically on the 3–4 tags your brief depends on, with results recorded.
- [ ] Scorecard filled; compliance / deployment-posture cells used as blocking filters.
- [ ] 2–3 unresolved gaps flagged with a POC methodology sketch.
- [ ] One-page recommendation: primary + fallback + phasing (with trigger) + POC plan + open risks.
- [ ] Every factual claim backed by a dated URL or a benchmark run ID.

## Stretch goals

- **Benchmark cost at steady state** — characters/day × rate card × any add-ons (cloning seats, HD tier, PCU reserved capacity). Produce a monthly bill estimate per shortlisted provider.
- **Add a mid-stream cancel probe.** Script a request that starts synthesis, sends a cancel 200 ms in, and records the latency until audio actually stops. This is the barge-in metric Module 106 will consume.
- **Do a 20-listener MOS panel on 20 utterances** across two provider × two voice combinations, P.808 shape, stratified by register (greeting / number / apology / long-form). Report preference rates with a clustered bootstrap CI. (Preview for exercise-04.)
- **Run the matrix for all three briefs.** Different compliance posture + different latency regime + different volume changes the rankings; filling it three times is where the Chapter 1 skills consolidate.

## Keep it

The scorecard template with your measured TTFB probe code becomes your reusable artefact. Every time a vendor releases a new voice line, re-run the probe (half an hour) and update the dated cells. Over a quarter this is how you keep the recommendation honest.
