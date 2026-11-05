# exercise-03: Voice Agent Engineer vs. Adjacent Roles

**Estimated effort:** 1 hour · **Output:** one structured comparison + a boundary-breakage postmortem

## Objective

Produce a concrete comparison between the voice-agent engineer and the six adjacent roles in Chapter 3 — not in general, but on a single specific voice feature. Then write a short postmortem for a (plausible, fictional-or-real) incident where the boundary was broken and work went wrong.

## Prerequisites

- [Chapter 3: Adjacent Voice-Ecosystem Roles and Sharp Boundaries](../03-adjacent-voice-ecosystem-roles.md).

## Steps

1. **Pick a feature.** Choose one specific voice feature to anchor the comparison. Examples:
   - An ambient-scribe clinical dictation feature that inserts structured notes into an EHR.
   - A sales-call live-coaching assistant that whispers prompts to a rep during a customer call.
   - An inbound-call triage agent for a bank that handles routine balance queries and routes everything else to a human.
   - An outbound appointment-reminder caller for a clinic.

   Pick one and *name the product's one most uncomfortable requirement* — e.g. medical terminology, HIPAA, STIR/SHAKEN, two-party consent, barge-in during a coaching whisper.

2. **Build the comparison table.** For each of the six adjacent roles from Chapter 3 — ASR research scientist, TTS research scientist, conversation designer, voice-UX designer, IVR/CTI developer, contact-centre operations engineer — fill three columns:
   - **Their centre of gravity on this feature** — what question would they be answering?
   - **Your centre of gravity on this feature** — what question would you be answering?
   - **The handoff artefact** — the single document / interface / contract that passes between you.

   Keep each cell to two sentences. The exercise is in forcing the compression.

3. **Boundary-breakage postmortem (300–500 words).** Write a short postmortem (markdown, headings `# Incident`, `# What happened`, `# Root cause`, `# What should have happened`, `# Action items`) describing a realistic scenario where the boundary with **one** of the six roles was broken — in either direction — and the feature suffered. Examples of the kind of thing to write about:
   - A voice-agent engineer tuned the wake word's acoustic model without involving ASR research and shipped a regression on child voices.
   - A conversation designer rewrote the system prompt to sound more human, which broke an existing tool-calling contract.
   - The IVR team changed the SIP trunking vendor without telling the voice-agent team and inbound codec negotiation silently started falling back to PCMU, blowing the latency budget.

   You can use a scenario you have seen or a clearly-labelled fictional one.

## Starter guidance

- The point of the comparison table is to name *the actual question each role is answering*, not a generic job description. "Trains the model" is a job-description sentence; "Which pre-training data distribution best covers paediatric accented English?" is the question.
- The postmortem should name the role (not the person), the artefact that failed, and the organisational fix (not just a code fix).
- Do not pick an easy postmortem (e.g. "the engineer wrote bad code"). The point is to practise naming boundary issues, which are organisational, not individual.

## Acceptance criteria

- [ ] One named feature with its one most uncomfortable requirement called out.
- [ ] A 6-row comparison table covering all six adjacent roles, each cell two sentences or fewer, each handoff artefact named concretely (not "collaboration").
- [ ] A postmortem of 300–500 words covering incident, what happened, root cause, what should have happened, and action items.
- [ ] The postmortem's action items are process / artefact fixes, not individual blame.

## Stretch goals

- Repeat the exercise with a feature from a *different* category of employer in Chapter 1 (e.g. first clinical, then sales). Does the comparison table shift noticeably? If so, which rows?
- Extend the comparison table with a seventh row for a role not in Chapter 3 (e.g. localization engineer, accessibility specialist, data-labelling vendor). Where does the boundary fall?

## Keep it

The comparison table feeds Module 108 (safety / compliance), where many of the adjacent roles become mandatory reviewers, and Module 111 (orchestration), where the handoff artefacts become the actual integration surfaces.
