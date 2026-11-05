# exercise-03: EU AI Act Article 5 and 50 Mapping for Voice Agents

**Estimated effort:** 2 hours · **Output:** a feature-by-feature classification sheet against Article 5 and Annex III + draft Article 50 disclosure copy + an FRIA skeleton for the retained in-scope feature.

## Objective

Translate [Chapter 3](../03-eu-ai-act-voice.md) into a concrete classification exercise for a specific voice-agent product. You will (a) classify each product feature against the Article 5 prohibitions and the Annex III high-risk categories, (b) author the Article 50 opening-disclosure and synthetic-audio provenance language, and (c) draft the skeleton of a Fundamental Rights Impact Assessment (FRIA) for a feature you determine is high-risk.

## Scenario

You are shipping "EuroCall," a multi-tenant voice-agent platform marketed to EU customers. Three specific customer use cases are under design:

- **A — Retail customer-support bot.** Answers customer questions about orders, initiates returns, offers a discount code for churn risk. Routes elderly / accessibility callers to a "simplified mode" by inferring age from voice (current proposal).
- **B — Interview-screening bot for a staffing agency.** Conducts first-round phone interviews for seasonal retail roles. Scores candidates on "enthusiasm" and "communication" from voice features.
- **C — Clinical post-discharge follow-up bot for a hospital network.** Calls patients 48 hours after discharge to ask structured health questions ("Any fever? Any pain in the surgical site?"). Flags concerning answers to a nurse.

All three customers are EU-established. The platform runs on EU-region endpoints for all audio processing.

## Prerequisites

- [Chapter 3 — The EU AI Act for Voice Systems](../03-eu-ai-act-voice.md).
- Primary sources: [Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) (the AI Act). Especially Articles 5, 50, 27, and Annex III.
- Primary source: [AI Act Recitals 18, 44, 132–136](https://artificialintelligenceact.eu/) for Article 50 interpretation.

## Steps

### Part A — The classification sheet (45 minutes)

1. Produce a classification table with one row per use case (A, B, C) and one column per Article 5 prohibition + Annex III category. Cells:
   - `prohibited` — cite the specific Article 5 subparagraph if yes.
   - `high-risk-annex-iii` — cite the specific point.
   - `not-applicable` — with a one-sentence reason.
2. For Use Case A, address the "age-from-voice" feature *specifically*. Which Article 5 subparagraph does it hit? Produce the one-line reasoning. Then propose a replacement feature that achieves the same business goal (accessibility routing) without inferring age from voice.
3. For Use Case B, address "enthusiasm" and "communication" scoring. Which Article 5 subparagraph(s) do these hit? Is any carve-out available? Produce the one-line reasoning.
4. For Use Case C, address whether the bot is high-risk under Annex III. Cite the specific point. Determine whether this is in-scope for Article 14 (human oversight), Article 15 (accuracy / cybersecurity), and Article 9 (risk management).

### Part B — Remediation plan (30 minutes)

5. For each of Use Cases A and B, produce a *go / no-go / modify* recommendation. If *modify*, specify the exact feature changes that remove the prohibition (or that move the use case out of Annex III). Keep the recommendations in terms the product manager can understand:
   - "Remove the age-from-voice inference entirely. Replace with a self-service accessibility toggle in the caller's account."
   - "Replace 'enthusiasm' and 'communication' scoring with task-specific signal scoring (e.g., 'did the candidate answer the required question'). Have a human read the transcript."
6. For Use Case C, assume you proceed. List the five compliance artefacts you need to produce before launch per Chapter 3 + Annex III:
   - Risk management system (Art. 9).
   - Data governance (Art. 10).
   - Technical documentation (Art. 11).
   - Logging / record-keeping (Art. 12).
   - Human oversight plan (Art. 14).

### Part C — Article 50 disclosure copy (20 minutes)

7. Draft the Article 50(1) chatbot-disclosure opening for Use Case C. Produce:
   - The exact words the agent speaks at the start of the call.
   - A fallback if the patient is confused and asks "are you a person?" mid-call.
   - A localisation note — this is an EU-wide deployment; produce the English version and list the languages that will need localisation (for the EU, at least the 24 official EU languages).
8. Draft the Article 50(2) synthetic-audio-provenance plan:
   - What watermarking standard the TTS pipeline uses (point to Chapter 6: AudioSeal / SilentCipher).
   - What C2PA assertions are attached to the output container.
   - What the public documentation page will say about how a recipient can verify the synthetic-audio label.

### Part D — The FRIA skeleton for Use Case C (25 minutes)

9. Draft a one-page FRIA skeleton for Use Case C. Required sections per Article 27:
   - Description of the deployer's process in which the AI system will be used.
   - Period of time and frequency of use.
   - Categories of natural persons affected.
   - Specific risks of harm to those persons.
   - Measures for human oversight.
   - Measures in case of risk materialising.
10. Populate each section with concrete content (not placeholder text) specific to Use Case C.

## Starter guidance

- Article 5(1)(f) prohibits emotion-inference in workplace and education with a safety/medical carve-out. A post-discharge follow-up bot that asks "does it hurt?" and routes based on the *answer* is not inferring emotions from voice features — it is doing standard question-answering on transcribed content. Different use case B is in scope because it infers *affect* from voice features of the candidate.
- Annex III Point 4(a) covers "AI systems intended to be used in the areas of recruitment or selection of natural persons" — Use Case B is explicitly in scope regardless of the Article 5(1)(f) question.
- Annex III Point 5(a)-(c) covers access to essential public services, credit, and emergency triage. A post-discharge follow-up bot is on the edge — healthcare triage is in scope; benign scheduling probably is not. Argue the edge.
- The 24 official EU languages are the Union's working languages (Regulation 1/1958 as amended). Localised consent and disclosure obligations under GDPR apply too (Chapter 4).

## Acceptance criteria

- [ ] Classification table for Use Cases A, B, C against every Article 5 prohibition and every relevant Annex III point.
- [ ] A one-line rationale for each classified cell, grounded in the specific regulation text.
- [ ] Remediation recommendations for A and B that eliminate the prohibition.
- [ ] A five-item compliance artefact list for Use Case C naming Articles 9, 10, 11, 12, 14.
- [ ] Article 50(1) chatbot-disclosure opening text for Use Case C, in English, with fallback and localisation plan.
- [ ] Article 50(2) synthetic-audio plan naming the watermark and the C2PA assertions.
- [ ] FRIA skeleton for Use Case C with six populated sections.

## Stretch goals

- **GDPR layering.** For Use Case C, identify the GDPR lawful basis (likely Article 9(2)(h) for health data processing under medical professional's responsibility) and the DPIA elements that overlap with the FRIA. Produce a one-page DPIA + FRIA cover sheet showing where the two assessments share content and where they diverge.
- **Code of Practice mapping.** If the AI Office's General-Purpose AI Code of Practice is published at the time you do this exercise, map Use Case C's upstream model dependencies to the Code's provider obligations. <!-- needs-research: confirm Code of Practice is finalized at time of use -->
- **Cross-product standardisation.** Across all three use cases, what Article 50 disclosure text is the *same* at platform level (one text that all tenants inherit)? Draft it.
- **Penalty exposure.** For a hypothetical breach, compute the maximum penalty under Article 99 for (a) a prohibited-practice violation (A or B, worst-case) and (b) an Article 50 violation. State the turnover-vs.-fixed-amount computation.

## Keep it

The classification exercise is a template you will rerun for every new voice-agent feature. Keep the Article-5-prohibition and Annex-III decision trees in a reusable form.
