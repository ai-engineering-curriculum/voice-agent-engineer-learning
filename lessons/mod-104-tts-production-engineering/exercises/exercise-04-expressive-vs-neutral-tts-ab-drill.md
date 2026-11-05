# exercise-04: Expressive vs. Neutral TTS A/B Drill

**Estimated effort:** 2 hours · **Output:** a listener-panel preference report stratified by register, with CIs + a per-register voice recommendation

## Objective

Run an honest A/B between an expressive TTS and a neutral TTS for a specific voice-agent product, stratified by register, with a listener panel. By the end you will have **preference rates with defensible confidence intervals** — the kind of number a product decision can rest on — plus a per-register recommendation that says "use the expressive voice for apologies, the neutral voice for balance read-outs" or the equivalent for your product.

This is Chapter 4's A/B drill made concrete. The deliverable is a report, not a pitch.

## Prerequisites

- [Chapter 4 — Expressive vs. Neutral TTS and Voice Selection UX](../04-expressive-vs-neutral-tts-and-voice-selection-ux.md).
- Access to **both** an expressive TTS (ElevenLabs Conversational, Hume EVI, Cartesia Sonic Turbo, `gpt-4o-mini-tts` with a style prompt, Polly Generative, or Azure `<mstts:express-as>`) and a neutral TTS (OpenAI `tts-1`, Azure Neural without style, Polly Neural, Google Chirp 3 HD) that share a locale.
- Python 3.10+ with your streaming client from exercise-02 (or equivalent).
- A listener pool: 10–20 people who are not you, not your team, and (ideally) match your product's target users. Prolific, UserTesting, or an internal panel all work. Free panels (friends) are acceptable for the exercise but note the limitation.

## Steps

### Part A — Define the registers (15 minutes)

1. Enumerate the registers your product actually produces. A typical voice agent has 4–7:
   - **Greeting / opening** — first utterance of a session.
   - **Task confirmation** — "I've placed the order for…"
   - **Information read-out** — balances, numbers, statuses.
   - **Clarification / question** — "Did you mean…"
   - **Apology / error** — "I'm sorry, I couldn't…"
   - **Hold / delay** — "One moment while I look that up."
   - **Hand-off / escalation** — "Let me connect you to…"
2. For each register, draft 3–5 utterances drawn from or modelled on real LLM outputs for your product. Total: 15–25 utterances.

### Part B — Render the pairs (30 minutes)

3. Render every utterance in two conditions:
   - **Expressive** — expressive engine, style/prompt appropriate for the register (empathetic for apology, cheerful for greeting, etc.).
   - **Neutral** — neutral engine, no style modification.
4. Keep everything else constant per pair: same locale, comparable voice gender / age-coding, same text, same SSML `<say-as>` layer (if any).
5. If you can only test one register's style per engine (e.g., Azure `<mstts:express-as style="empathetic">` is per-voice), note it; stratification still works as long as expressive is expressive within each register.
6. Save the pairs as a CSV + audio files, with per-pair IDs you can quote in the panel.

### Part C — Run the listener panel (45 minutes of your time, more of theirs)

7. Build a simple A/B listening tool:
   - Random order per listener (counterbalance which side is A and which is B).
   - Each pair: "Which voice would you prefer to receive this response from?" plus a short open-text "why" box.
   - Secondary question per pair: "Which sounds more appropriate for this content?" (same scale).
8. Recruit 10–20 listeners. Give them context: "You are a customer of [product type]. Imagine you are hearing these in a real interaction."
9. Collect responses. Target ~25 pairs × 15 listeners = ~375 judgements. More is better; 10 listeners × 20 pairs is the minimum defensible.
10. ITU-T Recommendation P.808 is the industry-standard methodology for subjective speech-quality collection; match its shape (randomisation, counterbalancing, warm-up pairs). You do not have to implement the full standard; you should be able to point to where your protocol deviates.

### Part D — Analyse (30 minutes)

11. Compute per-register preference rates:
    - Pooled: fraction of judgements preferring expressive.
    - Clustered bootstrap over listener-pair preferences to get a 95% CI (clustered because the same listener judges multiple pairs — their preferences are correlated).
12. Flag registers where the CI excludes 50% — that is where you have a defensible preference.
13. Compute the overall pooled preference as a sanity check, but **report per-register as the headline finding**. An overall 55-45 split is meaningless; a 75-25 split on apologies with a 30-70 split on balance read-outs is a decision.
14. Produce a plot: y-axis registers, x-axis preference rate with CI bars crossing 50%.

### Part E — The recommendation (15 minutes)

15. Write a one-page voice-selection recommendation:
    - Per register, which engine / style wins (preference rate + CI).
    - Where to use a steerable expressive voice with per-turn style ("one voice, empathetic on apologies, neutral on numbers") versus two different voice-IDs ("sorry, hard-no, do not change voice mid-session").
    - How this interacts with the Chapter 4 consistency rule (no mid-session voice-ID changes).
    - A listening-panel schedule — this is not a one-time exercise. Vendor model revisions change the rankings; re-run quarterly.

## Starter guidance

- **Keep the listener pool honest.** If you are the listener, you already know the brief and your preferences are biased. Use at least 10 non-team listeners.
- **Open-text "why" answers are gold.** The raw preference is a number; the "I prefer the neutral one because the cheerful one feels like it's selling me something" sentence is the product insight.
- **Randomise and counterbalance.** Half your listeners should hear A first; the other half B first. A listener who always picks B is probably doing a positional heuristic.
- **Report with CIs.** Point estimates without uncertainty are a model-evaluation-engineer anti-pattern (see mod-103 Chapter 7). A 55-45 point estimate with a 48-62 CI is not a real finding.
- **Stratify, always.** The whole point is per-register insight. Pooled numbers hide the finding.
- **Keep the audio.** The artefacts are the evidence. A product decision based on A/B results without the audio clips is unauditable.

## Acceptance criteria

- [ ] 4–7 registers defined; 3–5 utterances per register; 15–25 pairs rendered.
- [ ] Panel of at least 10 non-team listeners; ~250+ judgements collected.
- [ ] Per-register preference rates with clustered-bootstrap 95% CIs, reported in a table or plot.
- [ ] At least 2 registers where the CI excludes 50% — one favouring expressive, ideally one favouring neutral.
- [ ] One-page voice-selection recommendation with a quarterly re-run cadence.
- [ ] Representative audio clips preserved; listener-response raw data retained.
- [ ] Protocol documented (randomisation, counterbalancing, question wording, listener briefing).

## Stretch goals

- **Add a trust / warmth / competence Likert scale.** Beyond raw preference, rate each clip on 5-point scales for trust, warmth, competence. Expressive often wins on warmth and loses on competence; this is a product-design insight.
- **Steerability ablation.** On a steerable engine (`gpt-4o-mini-tts`, Azure `<mstts:express-as>`), test the same utterances with the "wrong" style (cheerful on an apology, empathetic on a balance read-out). Confirm the style actually does what you want — and does not.
- **Cross-demographic stratification.** Split your listeners by age bracket or role. Preferences for expressive voice commonly differ with demographic; your product's target segment may surprise you.
- **Longitudinal consistency test.** Record a 10-turn simulated conversation using your winning configuration; play it to 5 listeners end-to-end. Does the voice sound like one person across turns? Any drift is a Chapter 4 consistency bug.
- **A/B with the real product.** If you have a staging environment, A/B test the winning configuration against the current production voice on a small user slice. Preference in the panel does not always map to retention / NPS in the product; measure both.

## Keep it

The listener-panel protocol and the clip bank are reusable. The next time a vendor releases a new voice line, re-render the pairs, re-run the panel, and update the recommendation. The CI methodology stays the same; the clips stay the same; only the voice changes. Over a year this is how you keep voice selection a data decision rather than a taste decision.
