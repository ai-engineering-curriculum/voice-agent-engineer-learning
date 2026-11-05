# exercise-01: Modality Interleaving Contract Design

**Estimated effort:** 2 hours · **Output:** a one-page modality-interleaving contract for a specific product scenario

## Objective

Produce a written, reviewable **modality-interleaving contract** for a multimodal voice-agent feature of your choice. The contract is the artefact you would bring to a design review *before* writing any code, so that product, privacy, and engineering reviewers can ratify the shape of the agent before implementation choices lock it in.

This is a paper deliverable. No wire-up.

## Prerequisites

- [Chapter 1 — The Modality Interleaving Contract](../01-the-modality-interleaving-contract.md).
- Enough product context to pick a scenario (yours, or one of the reference scenarios below).

## Reference scenarios

Pick one or propose your own (if proposing, justify in one paragraph why it is worth building).

1. **Consumer electronics support agent.** Voice-first. User may upload a screenshot or photo of an error. Rare tool-returned diagrams. US consumer traffic.
2. **Field technician co-pilot.** Technician in the field, camera on the workbench, intermittent screen share from a tablet. Continuous video is the norm.
3. **Clinical ambient scribe with reference imagery.** Clinician dictating; occasionally shows a wound photo or a form on screen for the scribe to transcribe into the note.
4. **Shopping assistant on mobile.** User asks for help choosing a product; may attach a photo of a product they already own, a picture of a room, or a screenshot from a competitor's site.

## Steps

1. **Pick the archetype.** State explicitly whether this is **voice-first-with-vision-on-demand**, **true multimodal**, or a product that spans both as explicit modes. Justify in one paragraph.

2. **Write the modality manifest.** List every input modality the agent accepts. For each:
   - Arrival pattern (streaming vs. discrete).
   - Who initiates (user vs. agent tool call).
   - Transport (WebRTC audio track, HTTPS multipart, screen-share surface, etc.).
   - Expected size / rate.
   - Max size / rate limits you will enforce.

3. **Pick the interleaving policy.** Choose one of: modality-independent turns, sticky visual context, deictic binding, fully multimodal turn. Justify against the archetype and the expected traffic shape.

4. **Document ordering guarantees.** State explicitly what happens when voice and vision arrive together or out of order. At minimum:
   - If vision arrives during an active voice turn, does the agent wait, process in order, or merge?
   - If voice arrives while vision is still being uploaded, what is the hold-off window (ms)?
   - What happens on timeout?

5. **Write the response contract.** A short policy table covering:
   - Required citation shape (bounding box, OCR excerpt, DOM reference, none).
   - Behaviour on low vision confidence.
   - Behaviour on missing vision for a vision-dependent question.
   - Modality-specific refusals (medical, PII, PCI, your domain's specifics).

6. **Call out your top three risks.** For each: what is the failure, how would you detect it in production, and what is the mitigation.

7. **Draft the one-page contract.** Suitable to hand to a product manager and a privacy reviewer. Format is up to you — the examples in Chapter 1 are a starting point, not a template to copy.

## Starter guidance

- **Avoid "all of the above" interleaving policies.** A policy that tries to do all four at once is a policy without a decision; the implementation will be buggy and the reviewers will not know what to ratify.
- **Write the contract in product terms, not provider terms.** "Accepts uploaded images up to 20 MB" is a product decision; "uses OpenAI `image_url` with `detail: high`" is an implementation detail that comes later.
- **The privacy row for continuous video is the sharpest.** If you chose scenario 2 or 3, spend the time there — the rest of the modules will assume your contract is already specific about consent and retention.
- **Separate consent gestures for voice and vision.** Voice-only consent at the start of a call does not cover screen share that starts mid-call. Reflect this in the contract.

## Acceptance criteria

- [ ] One archetype chosen with a one-paragraph justification.
- [ ] A modality manifest covering every input modality with arrival pattern, initiator, transport, and size/rate.
- [ ] One interleaving policy chosen with justification.
- [ ] Explicit ordering guarantees for voice/vision collisions and timeouts.
- [ ] A response contract covering citation shape, low-confidence behaviour, missing-vision behaviour, and at least two refusal categories.
- [ ] Top three risks with detection and mitigation.
- [ ] One-page document suitable to hand to a product + privacy reviewer.

## Stretch goals

- **Second-scenario comparison.** Do the same work for a second scenario and compare the contracts side by side. The exercise is to see which pieces are stable across products and which change radically.
- **Consent flow walkthrough.** Add a storyboard of the user-facing consent flow — screens, prompts, revoke affordances — and trace how each consent decision maps to specific transport / retention behaviour.
- **State-machine diagram.** Draw the archetype-mode transition state machine for a product that spans both voice-first and true-multimodal modes.

## Keep it

The contract is the foundation for exercises 02–05 in this module. If you later build the full multimodal agent in project-102, you will reuse this document as the design brief. Save it in a form you can hand to a reviewer without reformatting.
