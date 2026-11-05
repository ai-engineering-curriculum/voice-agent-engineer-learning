# 03. The EU AI Act for Voice Systems

## Why this chapter

The EU AI Act (Regulation (EU) 2024/1689, published in the Official Journal on July 12, 2024, entered into force August 1, 2024) is the first horizontal AI regulation of real consequence in the EU, and the first to directly regulate *voice*-specific AI uses. Three families of obligations matter for a voice-agent engineer:

1. **Prohibitions** (Article 5) — certain uses are outright banned. Two of them hit voice directly.
2. **Transparency** (Article 50) — synthetic audio and chatbots must disclose themselves.
3. **High-risk classification** (Annex III) — certain voice-system deployments flip the whole GPAI / provider / deployer obligation stack on.

The AI Act's application is staged. Prohibitions applied from **2 February 2025** (Article 113(a)). GPAI obligations from **2 August 2025** (Article 113(b)). High-risk + transparency from **2 August 2026**. Full application including existing high-risk systems from **2 August 2027**. If you are reading this at authoring time, check the current application state for every article you rely on. <!-- needs-research: confirm application dates and any Commission guidance updates at the time of use -->

The AI Act's penalties scale: up to **€35 million or 7% of worldwide annual turnover** for prohibited-practice violations; up to €15 million or 3% for other obligations. These are GDPR-shaped numbers. Treat them like it.

## Article 5(1)(f) — emotion inference in workplace and education

[Article 5(1)(f)](https://artificialintelligenceact.eu/article/5/) prohibits placing on the market, putting into service, or using "AI systems to infer emotions of a natural person in the areas of workplace and education institutions, except where the use of the AI system is intended to be put in place or into the market for medical or safety reasons."

For a voice agent this cuts across several common product shapes:

- **Call-centre "agent assist" with real-time sentiment scoring** on the *employee's* side of the call (nudging the human agent on tone, empathy, or escalation risk) — prohibited when the subject of inference is the employee, which it is in most nudge features.
- **Interview-screening voice assistants** that infer emotion from the candidate — prohibited (education + workplace both apply at the candidate boundary).
- **Classroom voice tutoring** that infers student engagement or frustration — prohibited.

The scope of "infer emotions" is in [AI Act Recital 18](https://artificialintelligenceact.eu/recital/18/) — it "refers to AI systems aimed at identifying or inferring emotions or intentions of natural persons on the basis of their biometric data." Simple detection of "the customer is speaking loudly" is arguably outside the inference boundary; a model that outputs "the customer is angry" / "frustrated" / "sad" is inside.

The medical / safety carve-out is narrow. A call-centre monitoring an employee for signs of acute distress to send them home sick is *not* a safety exception for the normal nudge feature; the exception contemplates genuinely safety-critical use cases (identifying a driver falling asleep).

**Engineer-shaped translation:** if your voice agent has a "sentiment pipeline" that outputs discrete emotion labels about an employee or student, remove it from EU deployments. Replace with task-signal pipelines (e.g., classifying *agent action* — "escalate," "retry" — rather than *human state*).

## Article 5(1)(g) — biometric categorisation of sensitive attributes

[Article 5(1)(g)](https://artificialintelligenceact.eu/article/5/) prohibits "biometric categorisation systems that categorise individually natural persons based on their biometric data to deduce or infer their race, political opinions, trade union membership, religious or philosophical beliefs, sex life or sexual orientation." The carve-out is for lawful categorisation of labelled datasets in law-enforcement contexts.

For voice: inferring demographic sensitive attributes *from voice features alone* is prohibited. "Gender inference from voice pitch" is the trivially-prohibited example. "Ethnicity inference from accent" is the harder case, and it is also prohibited.

Many voice-agent products once bundled a *speaker profile* feature that quietly inferred age, gender, or accent from the call audio for routing ("route elderly callers to the simplified menu"). That feature is now illegal in the EU regardless of how useful it is. Remove it; use declared preferences instead.

## Article 50 — transparency obligations

[Article 50](https://artificialintelligenceact.eu/article/50/) imposes transparency obligations on four classes of AI system. Two apply to voice:

- **Article 50(1): "AI systems intended to interact directly with natural persons"** — providers must design them so that natural persons concerned are informed they are interacting with an AI system, unless this is obvious from the circumstances. This is the **chatbot disclosure obligation**. A voice agent answering a cold call is clearly in scope.
- **Article 50(2): deepfake / synthetic content disclosure** — deployers of AI systems that generate "synthetic audio, image, video or text content" must label the content as artificially generated or manipulated. Audio is *explicitly listed*.

The chatbot disclosure (50(1)) can be satisfied by the agent opening the call with something like:

> "Hi, I'm Aria, an automated voice assistant for ExampleCo. I'm an AI, not a human. How can I help you today?"

Vendor practices converge on an opening of this shape; the FCC's December 2024 Notice of Proposed Rulemaking proposes a similar US-side requirement, which may harmonise the two.

Article 50(2) is more subtle. The regulation itself is agnostic to *how* the label is implemented — the text says "the outputs of the AI system are marked in a machine-readable format and detectable as artificially generated or manipulated." That is a *watermark* requirement. The AI Office is expected to codify acceptable standards via codes of practice; the current state of the art is C2PA Content Credentials and audio watermarking schemes like AudioSeal and SilentCipher — covered in [Chapter 6](06-spoofing-deepfake-provenance.md).

The exception in Article 50(2) for "assistive, enhancing, or other similar" purposes is narrow and does *not* cover a voice-agent TTS pipeline in general. Expect to watermark your TTS output.

## Annex III — high-risk use cases

[Annex III](https://artificialintelligenceact.eu/annex/3/) enumerates the "high-risk" use cases. Several hit voice:

- **Point 1(a):** Remote biometric identification systems — a voice-authentication agent is in scope.
- **Point 1(c):** Biometric categorisation according to sensitive attributes (prohibited under Article 5(1)(g); see above).
- **Point 4(a):** Access to employment — AI systems used for recruitment or screening, which includes voice-based interview tooling.
- **Point 5(b):** Access to essential private services and essential public services and benefits — credit scoring and public-benefit eligibility systems are in scope. A voice agent that makes an eligibility decision is in scope at the decision layer.
- **Point 6:** Law enforcement — polygraph-like systems, "similar tools to detect the emotional state of a natural person," are in scope.
- **Point 7:** Migration, asylum, and border-control management — voice analysis in asylum interviews would be in scope.

A voice system classified as high-risk inherits the Article 9–17 obligations: risk-management system, data governance, technical documentation, record-keeping, transparency to deployers, human oversight, accuracy / robustness / cybersecurity, and conformity assessment with CE marking. This is a significant engineering-adjacent governance burden that typically moves into an `ai-risk-engineer` / compliance team. From the voice engineer's seat, the obligation that bites directly is:

- **Article 15 (accuracy, robustness, cybersecurity):** the voice system's performance must be documented and the system must be designed to be resilient to errors and attempts at third-party manipulation. Chapter 7's threat model is the input to this article.
- **Article 14 (human oversight):** a high-risk voice system must allow a human to intervene and to disregard, override, or reverse the output. For an autonomous outbound agent, this means a human-in-the-loop breakpoint for high-stakes decisions.

## General-Purpose AI (GPAI) model obligations

Article 51–55 impose obligations on providers of general-purpose AI models — which includes the large speech and multimodal models voice agents stand on (OpenAI Realtime, Gemini Live, Claude, Llama multimodal, Whisper). Providers must:

- Publish a sufficiently detailed summary of training data content (Article 53(1)(d)).
- Maintain technical documentation of the model.
- Comply with Union copyright law.

For "systemic risk" GPAI (currently models trained with >10²⁵ FLOPs, Article 51), add Article 55 obligations: model evaluation, adversarial testing, serious-incident reporting, cybersecurity.

**Engineer-shaped translation:** you are usually a *deployer* of a GPAI model, not a provider. Your obligation is primarily Article 50 transparency + Article 25 "putting-on-the-market" responsibilities. But if you fine-tune a model into a product that is itself GPAI (unlikely for a voice agent), you inherit provider obligations. The [AI Office GPAI Code of Practice](https://digital-strategy.ec.europa.eu/en/policies/ai-code-practice) codifies the current compliance expectations; check for the current version. <!-- needs-research: confirm current Code of Practice version and finalization status at the time of use -->

## Putting it together — the EU deployment checklist

For a voice agent launched in the EU:

1. **Confirm the use case is not prohibited** (Article 5). Explicitly: no emotion inference about employees or students; no biometric categorisation of sensitive attributes.
2. **Classify the use case against Annex III.** If it's in Annex III, conformity assessment + registration in the EU database + full Article 9–17 compliance is on. Engage a notified body if the law requires it.
3. **Implement Article 50 transparency.** The call opening discloses the AI nature of the system. The TTS output carries a machine-readable "synthetic audio" marker (C2PA + audio watermark).
4. **Document the GPAI-provider upstream.** For every underlying model (Whisper, Realtime, Gemini Live), record its provider and the version and retain the provider's public documentation. If the provider is on a GPAI Code of Practice signatory list, note that too.
5. **Plumb the human-oversight breakpoint.** For any high-risk decision, the agent cannot complete the action autonomously; a human must have the ability to intervene.
6. **Register where required.** High-risk AI systems must be registered in the EU AI System Database (Article 71).

## Interaction with GDPR

The AI Act's data obligations layer on top of GDPR, they do not replace it. If a voice agent processes voice data in the EU, GDPR applies regardless of the AI Act (see Chapter 4). In particular:

- **Lawful basis** is a GDPR question (Article 6).
- **Special-category biometric** treatment is GDPR Article 9.
- **DPIA** is GDPR Article 35. The AI Act adds a *Fundamental Rights Impact Assessment* obligation under Article 27 for certain deployers of high-risk systems — an FRIA, distinct from a DPIA, feeding the same governance machinery.

## Summary

- Article 5(1)(f): no emotion inference about employees or students.
- Article 5(1)(g): no inference of race, politics, union membership, religion, sex-life, or sexual orientation from biometric data (including voice).
- Article 50: disclose that the caller is talking to AI; watermark synthetic audio in machine-readable form.
- Annex III classifies many voice use cases as high-risk: biometric ID, employment screening, credit / essential services, law enforcement, border control. High-risk → Article 9–17 full stack.
- GPAI provider obligations layer upstream; deployer obligations (you) focus on Article 50 + putting-on-the-market responsibilities.
- Penalties: up to €35 M or 7% of turnover for prohibited practices; up to €15 M or 3% for other breaches.

Next chapter: GDPR specifically, plus UK PECR / Ofcom for unsolicited calls.
