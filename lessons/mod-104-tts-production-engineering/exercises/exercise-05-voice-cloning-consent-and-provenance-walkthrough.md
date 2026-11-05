# exercise-05: Voice-Cloning Consent and Provenance Walkthrough

**Estimated effort:** 2 hours · **Output:** a shippable consent form, an audio consent script, a provenance record template, a kill-switch + takedown playbook, and an escalation memo

## Objective

Walk the full voice-cloning stack for a **named use case**, producing the artefacts you would need before shipping a non-self voice clone to production. By the end you will have a consent form legal could sign off on (with their tweaks), a provenance record schema your service could store today, a takedown playbook that works at 2 AM, and a short escalation memo to your `agentic-safety-engineer` peer on at least one specific risk.

This is Chapter 5 made concrete. Do not actually clone anyone's voice for this exercise. The deliverables are the paperwork + the pipeline sketches that would let you clone responsibly if the business required it.

## Prerequisites

- [Chapter 5 — Voice Cloning: Consent, Provenance, and Knowing When to Say No](../05-voice-cloning-consent-and-provenance.md).
- Read-through of one vendor's cloning docs end-to-end (ElevenLabs Professional Voice Cloning, Resemble Professional, Azure Custom Neural Voice). Pick one as your reference vendor.
- Awareness of your deployment's jurisdiction. If you do not have a real one, pick one for the exercise (US-IL + EU + UK is a useful hard combo).

## The clone brief

Pick one of these three — again deliberately under-specified. Write the one-paragraph expansion before you start.

- **Brief A — "Brand voice" from a licensed voice actor.** The product wants a recognisable, warm, mid-30s, mid-register English voice from a paid voice actor, under a 2-year licence, used across a consumer companion app in US + UK. PVC.
- **Brief B — "Support agent voice" from a consenting employee.** A tenured customer-support rep has agreed to let their voice be cloned for use as the default agent voice in your internal support product, available to all employees across the company, until they leave the role. IVC.
- **Brief C — "Voice preservation" for a user.** The product is a healthcare app for ALS patients whose speech is deteriorating; users bank their voice early and the product generates speech on their behalf as the disease progresses. Self-cloning at user initiation. IVC with liveness.

## Steps

### Part A — The written consent form (30 minutes)

1. Draft a one-to-two-page consent form the speaker will sign. It must include the eight items Chapter 5 enumerates:
   - Speaker identity + verification method.
   - Requesting party.
   - Scope of use (use cases, geographies, platforms, languages, duration).
   - Explicit statement that AI will synthesise speech in their voice saying words they did not say.
   - Revocation right and SLA.
   - Governing law and jurisdiction.
   - Signature + witness.
   - Timestamp and non-repudiable storage note.
2. Call out the sections where your legal team will want to be involved (governing law, warranty, indemnity). Do not attempt the legal drafting — this is a working template for legal to redline.
3. Flag any clauses the vendor's own consent flow does not cover, which your form must add.

### Part B — The audio consent recording script (10 minutes)

4. Write the standardised consent script the speaker will read aloud, 30–60 seconds, matching the written form's scope.
5. Design the consent-recording flow: how the audio is captured (sampling rate, codec, environment), who witnesses, how the file is stored (encrypted, append-only, with a hash + timestamp from a trusted source — object-storage object-lock or similar).

### Part C — The provenance record (25 minutes)

6. Design the data model your service will store per voice-ID. Start from the JSON template in Chapter 5 and extend for your brief:
   - Required fields; optional fields.
   - References to the written consent (hash + storage URI) and the audio consent (hash + storage URI).
   - Model revision binding (voice-ID + vendor model revision pair — treat as one artefact).
   - Watermark posture per vendor.
   - Deployment scope metadata (geographies, languages, platforms).
7. Decide the storage shape — a dedicated `voice-identity` service table? An existing identity service? An audit-log stream? Document why.
8. Specify the retention policy and the propagation rules on revocation.

### Part D — The kill switch + takedown playbook (25 minutes)

9. Write a 1-page takedown playbook. On revocation / incident, who does what, in what order:
   - **Immediate (<1 hour):** disable the voice-ID in your TTS routing; block any synthesis calls that reference it. This is the kill switch — test that it works from the control plane, not as a vibe.
   - **Short-term (<48 hours):** notify the vendor to disable the voice-ID on their side; confirm propagation.
   - **Medium-term (<1 week):** delete the voice-ID from the vendor's active storage per the vendor's deletion SLA. Update the provenance record with the revocation reason, requester, and timestamps.
   - **Audit:** export the full synthesis log for that voice-ID over the retention window, including any downstream distributed audio artefacts.
10. Add the escalation tree: who in engineering owns each step, who in legal is informed, who in exec is informed above what severity threshold.
11. Specify what "incident" means and the three severity levels: a consent dispute (speaker says scope was exceeded), a technical breach (voice-ID used outside documented bounds by your own system), an external misuse report (someone claims your product's cloned voice was used elsewhere — covered in Module 108).

### Part E — The "when to say no" case + escalation memo (20 minutes)

12. For your brief, name one specific risk you cannot resolve at your level — one that needs `agentic-safety-engineer` or legal input. Examples depending on the brief:
    - Brief A: the voice actor leaves the agency; the agency claims renewal terms; what is your fallback?
    - Brief B: the employee changes roles; is the voice still deployable? Does leaving the company trigger revocation?
    - Brief C: a family member requests a voice-ID be created for a deceased user who banked audio pre-death; what is the policy?
13. Write a 1-page memo to your `agentic-safety-engineer` peer: the current state, the specific question, the artefacts you have, the decision you need. This is the Chapter 7 handoff shape.

### Part F — The "we are not shipping this" case (10 minutes)

14. Separately, sketch a request your product would receive that you would refuse — "clone a public figure for a parody skit," "clone a customer's voice on-signup with a click-through," "clone a voice from a 10-second YouTube clip without consent." Pick one. Write the 3–5 sentence refusal note to the requester, and the internal log entry that captures why.

## Starter guidance

- **Do not actually clone anyone.** Every artefact in this exercise is a document or a pipeline sketch. If you clone for the exercise, you have skipped the point.
- **Read your reference vendor's full consent docs end-to-end.** Half the subtlety is in which clauses the vendor's flow covers and which they punt to you.
- **Legal is a peer, not a formality.** The output of this exercise is something you would walk into legal with. "I wrote this; please tell me what you'd change" is the working mode.
- **Jurisdiction is real.** Illinois BIPA is a real cause of action in Illinois; your "oh we'll just consent" is unenforceable there without specific wording. Flag jurisdiction-specific risks in the consent form.
- **Revocation SLA is the honesty test.** A "we'll delete when we can" clause will not survive audit. Pick an SLA you can enforce (hours for kill switch, days for vendor-side deletion) and build the pipeline to it.

## Acceptance criteria

- [ ] Brief expansion paragraph at the top.
- [ ] Draft written consent form covering the eight Chapter 5 elements, with a flagged "legal to review" section.
- [ ] Audio consent script + capture-and-storage flow spec.
- [ ] Provenance record schema (JSON or equivalent), extended for the brief.
- [ ] Takedown playbook with concrete SLAs per step and an escalation tree.
- [ ] One specific `agentic-safety-engineer` escalation memo (1 page) on a real risk.
- [ ] One "we are not shipping this" refusal + internal log entry.
- [ ] A brief documentation of which Chapter 5 jurisdictional regimes apply to the brief and how the artefacts satisfy them.

## Stretch goals

- **Mock the kill switch.** Build the smallest possible toy voice-identity service — a Python app with an in-memory map `voice_id -> is_active`. Have the TTS routing layer check it before every synthesis. Flip a voice off; watch synthesis fail. This is the control plane you want.
- **Watermark probe.** If your reference vendor emits a provenance watermark, synthesise three clips and run them through the vendor's detector (Resemble's detector, ElevenLabs' AI Speech Classifier). Confirm the mark survives round-trip; see whether it survives a re-encode (MP3 bounce, Opus transcode).
- **C2PA manifest.** For a file-distribution scenario, attach a C2PA signed manifest to a generated clip. Verify with a C2PA validator (the Content Authenticity Initiative's tooling). Document the gaps (which fields are not vendor-populated yet).
- **Legal-ready package.** Combine the five documents (consent form, audio script, provenance record, takedown playbook, escalation memo) into a single PDF or wiki page. Format it the way a legal reviewer would prefer (sections, defined terms, a changelog). This is the shape counsel will ask for when you actually need the signoff.
- **Red-team your own consent flow.** Write 5 attacker scenarios: a speaker who later claims fraud, a submitter who forged the audio, a jurisdiction-shopping actor, a vendor who loses the voice-ID, a revoked-consent that didn't propagate. For each, walk the artefacts above and note the weak point.

## Keep it

The consent-form template and the provenance-record schema are reusable across voice-ID creation requests. The takedown playbook is reusable across incidents. Put all three under version control and treat them like infrastructure. When a vendor contract renews, the consent terms update; when a regime changes (NO FAKES Act enactment, UK DUAA guidance), the form updates. This stays warm.
