# 06. Reading the System Cards: Voice-Cloning Safeguards, Emotion Inference, and the EU AI Act

## Why this chapter

The three speech-native frontier stacks (OpenAI Realtime, Google Gemini Live, Anthropic-in-a-relay with a speech-native leg) each publish safety documentation describing what they will and will not do when audio enters the model. A voice-agent engineer at L35 is expected to **read these documents**, know which mitigations survive into the production API, and map them to the specific legal regimes the deployment will encounter — in particular, the EU AI Act's Article 5(1)(f) prohibition on workplace and educational emotion inference, which is the most consequential voice-specific rule in global AI regulation today.

This chapter is not legal advice. It is the engineering-side read of the safety envelope: what the vendor claims to guarantee (voice-cloning prevention, emotion-inference boundaries, refusal behaviour), what the regime requires (EU AI Act high-risk and prohibited practices, state-level voice-cloning laws, workplace / education-sector carve-outs), and where the gap between the two creates engineering work that must land in this module rather than being deferred to Module 108.

Module 108 is the full compliance-and-safety treatment. This chapter is the speech-native-specific sliver: **the safety discussion inside the model card, not the safety discussion outside it.**

## The GPT-4o System Card: what to read for

OpenAI published the [GPT-4o System Card](https://openai.com/index/gpt-4o-system-card/) in August 2024, with a dedicated discussion of the audio modality that Advanced Voice ships. Read the full document; the voice-specific sections to pull out for this module:

### 1. Voice-cloning safeguards

The System Card describes the model's refusal to impersonate specific voices. Concretely, the Realtime API and Advanced Voice ship a curated set of pre-defined voices (`alloy`, `ash`, `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse` as of this writing — verify the current roster). The model will not clone a voice at inference time, and OpenAI applies a **classifier that detects and refuses attempts to steer the output voice toward a specific speaker.**

The safeguard the engineer needs to know:

- **No upload path for a reference voice.** The API does not accept a voice sample as input. This is a product-surface safeguard: the capability is not exposed.
- **Classifier on the output.** A classifier compares the generated audio against a reference set of protected voices (OpenAI-defined — presumably celebrities, politicians, employees); detection triggers a refusal.
- **Watermarking.** OpenAI has stated it embeds a watermark in generated audio; technical details are not public. Do not rely on a specific watermark scheme in your own provenance flow (Module 104 Chapter 5); design for the vendor to change it.

**What this means for you.** If your product needs a cloned voice — a brand voice, a licensed celebrity voice, an employee's voice — Realtime / Advanced Voice is not the stack. Go to the pipeline with a TTS vendor that offers Instant or Professional Voice Clone (Module 104 Chapter 5). Do not "work around" the safeguard; the design is deliberate, the refusals are usually correct, and attempts to circumvent them put you on the wrong side of both OpenAI's Terms and voice-cloning law.

### 2. Emotion inference

The System Card distinguishes between the model's ability to **detect and respond to prosody** (useful; the model can hear "the user is excited" or "the user is upset" and respond appropriately) and its ability to **emit inferred emotion labels about the speaker as a product feature** (constrained).

Read the card for the specific framing. The engineering takeaway:

- **Prosody-aware response is a feature.** The model can empathise with a frustrated user. That is a conversational skill.
- **"Emotion recognition as an output" is a feature boundary.** A product that uses the API to claim "this user is 72% angry" as a scored signal is a different product shape. OpenAI's policies and the System Card discussion of the audio modality push back on that shape.

This boundary is the engineering mapping to the EU AI Act Article 5(1)(f) prohibition, which is the sharper constraint; see §Emotion inference under the EU AI Act below.

### 3. Refusals, disallowed content, and the audio-modality-specific amplifications

A general refusal that the text model would deliver ("I can't help with that") becomes a spoken refusal in the Realtime API. The System Card notes amplifications specific to audio:

- **Non-consensual sexual content in a specific voice.** A pipeline model could be prompted to speak explicit content in the voice of a named person; the Realtime API's classifier plus voice-restriction prevents this.
- **Deceptive impersonation in time-sensitive social-engineering** (grandparent scams, CEO-voice fraud). The voice-cloning safeguard is the primary mitigation.
- **Self-harm and crisis routing.** The model is trained to produce specific crisis-referral responses when the user's audio signals distress. Verify the current referral behaviour in your production region; the resources the model cites rotate.

Design implication: **the vendor's safety stack is not configurable by you.** You cannot turn it off; you cannot trivially override it. If your product has a specific crisis-referral workflow (handoff to a human, specific hotline numbers for the user's jurisdiction), you must detect the vendor's refusal and route to your flow before the vendor's spoken refusal ships.

### 4. Red-team findings worth internalising

The System Card discusses external red-team findings from OpenAI's pre-release evaluations. The ones most relevant to voice-agent engineering:

- **Model blends into user speech tone.** On long calls, the model's prosody drifts to match the user; red-teamers noted that this can create undue rapport. The mitigation is the system prompt constraining the model's tone.
- **Unexpected voice swaps.** During pre-release testing, the model occasionally produced audio in a voice not configured for the session; the production classifier reduces but does not eliminate this. If your product hard-depends on a specific voice persona, measure voice-identity consistency in your eval harness (Module 112).
- **Audio out-of-distribution behaviour.** On non-speech audio (music, background noise, deliberately adversarial audio), model behaviour is less predictable. Model-serving teams typically ship input-side VAD and silence detection to filter out non-speech frames before they reach the model.

## The Google Gemini safety discussion

Google publishes model cards and safety evaluations for Gemini variants, including the Live-API-eligible variants. The relevant material is scattered across [ai.google.dev/gemini-api/docs/safety-settings](https://ai.google.dev/gemini-api/docs/safety-settings), the model release notes, and Google's responsible-AI posts. The engineering read:

- **Safety categories are configurable per-request.** Unlike OpenAI's Realtime API (which bakes mitigations into the service), Gemini exposes a `safety_settings` array per request with categories (`HARASSMENT`, `HATE_SPEECH`, `SEXUALLY_EXPLICIT`, `DANGEROUS_CONTENT`) and threshold levels. In Live sessions, the settings apply to the session.
- **Voice-cloning posture.** Gemini Live ships a small voice roster (`Charon`, `Kore`, `Puck`, others — verify current) and does not accept a reference voice at inference. Equivalent capability-side safeguard to OpenAI.
- **Grounding as a safety primitive.** Google Search grounding (Chapter 2) is both a feature and a hallucination mitigation; in domains where incorrect information would be harmful, enabling grounding is the recommended path.
- **Video modality and privacy.** The continuous video / screen-share lane has privacy implications that are more subtle than audio. Faces in frame, sensitive text in frame, children in frame — these are consent and data-protection surfaces. The Live API does not surface per-frame privacy signals; your UX has to do it.

<!-- needs-research: confirm the current roster of Gemini Live voices and the exact field names for safety_settings on the Live API at author time -->

## The Anthropic stance

Because Anthropic does not ship a first-party realtime voice (Chapter 3), the safety surface for Claude-via-relay is composed:

- Claude's model-level safety behaviour (Anthropic's Acceptable Use Policy, model cards) governs what the LLM leg will say.
- The STT vendor's safety posture (biased transcription, mishearing names, regulated-domain refusals) governs what the LLM sees.
- The TTS vendor's safety posture (voice-cloning consent, provenance watermarks, IPR-safe voices) governs what the user hears.

The composite safety posture is the pipeline's; there is no single document that covers it end-to-end. The engineering job is to compose the three stacks' safety behaviours into one explicit posture — and to document in writing what each layer is responsible for. This is the kind of thing Module 108 reviews; mention it here so the composite nature of a pipeline's safety story is not a surprise.

## The EU AI Act and Article 5(1)(f)

The EU AI Act was adopted in 2024 and enters into force on a staged timeline through 2026 (verify the stage applicable to your product at integration time). The statute distinguishes four risk categories: unacceptable (prohibited), high-risk, limited-risk (transparency obligations), and minimal-risk. Voice agents can touch all four, but one prohibition is specific to the speech-native stack.

**Article 5(1)(f):** the Act prohibits the placing on the market, putting into service, or use of AI systems to infer emotions of a natural person **in the areas of workplace and education institutions**, except where the use is intended for medical or safety reasons. Read the operative text at [EUR-Lex 32024R1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) (Regulation (EU) 2024/1689). The exact Article 5 text evolved during the legislative process; cross-check the final published version.

### What the prohibition actually prohibits

- **Scope: workplace or education institutions.** A workplace surveillance tool that scores employee calls on inferred anger / stress is in scope. A school platform that scores student engagement by vocal affect is in scope.
- **Mechanism: inferring emotions.** An AI system that outputs "this person is angry / sad / fearful" from biometric signals (voice included). The prohibition is on the inference as a system output, not on the model's general prosody awareness.
- **Carve-out: medical or safety reasons.** Clinical use (say, voice-based depression screening under physician supervision) and genuine safety use (say, detecting a worker in distress in a dangerous environment) are carved out. The carve-out is narrow; "wellness" is not "medical."

### How this binds the voice-agent engineer

The stack choice interacts with the prohibition:

- **Speech-native with a vendor safeguard.** OpenAI's position (via the System Card) is that emotion inference as an output feature is not something Realtime exposes. The model hears prosody and responds; it does not emit "emotion = 0.72 anger." This posture aligns with the prohibition. You still have to design your product not to **repackage** prosody signals into an emotion score — the vendor's capability boundary does not stop you from adding your own classifier upstream.
- **Pipeline with a third-party affect model.** If your pipeline includes an affect / prosody classifier (Hume EVI, a self-trained model), the output of that classifier is the exact surface the Act prohibits in workplace / education contexts. You must gate the feature by deployment context: available in a consumer companion app; disabled or absent in a workplace / education deployment.
- **Voice biometric identification** is separately regulated under Article 5(1)(e) and (h) for mass surveillance and for law-enforcement identification. If your voice agent authenticates users by voiceprint, verify the Article 5 and high-risk-system paths at integration time; see Module 108.

**The engineering rule.** If your product is deployed in the EU to a workplace or educational institution, the product must not output inferred-emotion data. Design the data model so that emotion labels are either absent or guarded by deployment-tenant flags. Don't ship the feature "disabled by default, enabled by feature flag" if the product UX implies the feature is available; this is the kind of design detail an EU regulator will not accept at face value.

### Transparency obligations (Article 50)

Separate from the prohibition: Article 50 requires that AI systems interacting with natural persons be designed so that **users are informed they are interacting with an AI**, unless the AI nature is obvious. For voice agents this is live: your cold-open prompt ("This is Alex, an AI assistant for X") is the Article 50 disclosure. Design it in; log it per session.

## Downstream state-level voice-cloning law (US)

Not EU-specific, but adjacent and often forgotten at this module's depth:

- **Illinois BIPA** — biometric information includes voiceprints; consent requirements; private right of action.
- **Federal NO FAKES Act (NOEXTE trajectory; verify current status)** — designed to create a federal right of publicity over digital likeness including voice. <!-- needs-research: confirm the enacted/pending status of the NO FAKES Act at author time; federal bills move -->
- **State right-of-publicity laws** — California, New York, Tennessee ELVIS Act — all cover voice to varying degrees.

These apply to **cloned** voices. Speech-native stacks that ship curated voices (OpenAI, Google) are at lower exposure; pipelines with cloned voices (Module 104 Chapter 5) have the primary exposure.

## Composing the safety story for your product

A production-grade voice-agent safety story is a four-part composition:

1. **Vendor-side stack safeguards.** The System Card (OpenAI), the safety settings (Gemini), the vendor policies (Anthropic). What the vendor guarantees / refuses by default.
2. **Your prompting and routing.** The system prompt, the tool descriptions, the policy-aware classifier you run on transcripts. The refusals and routings specific to your product.
3. **The compliance envelope.** Which regulations apply (EU AI Act, HIPAA, PCI, state laws). The specific features you must disable in specific deployments.
4. **The operational layer.** Logging, red-teaming, observability (Module 112), incident response, and the ability to turn features off within hours when a regulator or an incident requires it.

The engineer's job in this module is parts 1 and 2. Parts 3 and 4 are Module 108 territory; name them so the hand-off is clear.

## Common anti-patterns

- **Reading the System Card as a list of features.** It is a description of safeguards, red-team findings, and known failure modes. Internalise it as design input, not marketing.
- **Assuming vendor safeguards remove your obligation.** They do not. If your product collects audio from EU workplace users, the obligation is yours regardless of which vendor's model generated the audio.
- **Treating emotion inference as "just a feature."** It is a prohibited practice in workplace and education contexts under the EU AI Act. Design around it; do not work around it.
- **Shipping a voice-biometric login without a full compliance review.** Voiceprints are biometric data; state and EU law treat them specifically. This is not "just another authentication."
- **No Article 50 disclosure.** "This is an AI assistant" at the top of the call is the baseline; absence is a transparency violation.
- **Logging audio without a retention policy.** Audio logs are tightly regulated; the default "log everything forever" of normal observability is wrong. Module 108 has the retention framework.

## Summary

- Each speech-native vendor publishes a safety discussion (System Card, model card, usage policies). Read them: the voice-cloning safeguards, the emotion-inference boundaries, the refusal behaviours.
- OpenAI's Realtime API ships curated voices with a voice-cloning classifier and a watermark; Gemini Live ships curated voices and configurable safety thresholds; Anthropic composes safety across STT + LLM + TTS.
- The EU AI Act Article 5(1)(f) prohibits inferred-emotion outputs in workplace and educational contexts (medical / safety carve-outs are narrow). Design your data model so the prohibition is a design invariant, not a feature-flag.
- Article 50 requires that users be told they are interacting with an AI. Your cold-open prompt is the disclosure; design it in.
- US state law (Illinois BIPA, state right-of-publicity, federal NO FAKES-track statutes) binds voice cloning but is less load-bearing on the speech-native vendors' curated-voice stacks.
- Compose the safety story: vendor stack, your prompting, the compliance envelope, the operational layer. This module owns the first two; Module 108 owns the rest.

Next chapter: the open-weights speech foundation model survey — Moshi + Mimi, Spirit LM, Qwen2-Audio, SeamlessM4T, AudioLM / SoundStream — and when open-weights becomes a defensible compliance or cost choice.
