# 05. Half-Duplex vs. Full-Duplex: Moshi, Advanced Voice, and What Duplex Actually Means

## Why this chapter

Chapter 4 named "speech-native" as a single architecture option against the pipeline. It is not. Inside the speech-native category there are **three different duplex regimes**, each with different interaction behaviour, latency floor, consent implications, and engineering surface. The three reference points worth knowing:

- **Kyutai Moshi + Mimi** — true full-duplex: the model streams audio alongside the user, can hear itself and the user at the same time, and can interject mid-utterance. Sub-200 ms response, streaming overlap.
- **OpenAI Advanced Voice (GPT-4o Realtime)** — effectively **half-duplex**: the model listens, then speaks; server-side VAD decides the turn boundary; interrupts cancel the in-flight turn. Low latency, but the "it's my turn / your turn" state is single-threaded.
- **Google Gemini Live** — bidirectional streaming closer to full-duplex in intent (the stream is open in both directions), with server VAD handling turn-taking and the client able to emit audio continuously; closer to "overlapping half-duplex" than to Moshi's co-articulated full-duplex.

Understanding where each lives on the spectrum changes how you design the user-facing interaction model (Module 106's barge-in territory), how you budget latency (Module 109), and how you capture consent for recording what is effectively two simultaneous audio channels (Module 108).

This chapter is the duplex taxonomy, the architectural reasons each stack sits where it does, and the downstream engineering implications.

## The duplex spectrum

Borrow the vocabulary from radio engineering. Three regimes:

### Simplex

One speaker at a time; the channel is uni-directional. The non-speaking end cannot interject at all. Think: a public-address loudspeaker. Not useful for conversational voice agents but included for completeness.

### Half-duplex

The channel is shared. One speaker at a time, with a turn-taking protocol — a VAD on both sides decides who has the floor. The other side can start speaking (which implicitly asks for the floor) and the current speaker stops; this is barge-in. Think: walkie-talkies, 1980s speakerphones, most conversational voice agents today.

**OpenAI Advanced Voice sits here.** The Realtime API's server VAD watches the user's audio; while the model is emitting audio, incoming user speech triggers a cancel and the floor swaps. The model does not continue speaking into the user's audio. This is a design choice, not a limitation: it maps onto a familiar phone-call mental model and is cheap to reason about.

### Full-duplex

Both sides can speak simultaneously. The channel is literally bidirectional in time. In human conversation this is overlap: backchannels ("mhm," "yeah," "right"), laughter, mid-sentence clarification, simultaneous starts, polite interruption. The agent can say "right" while the user is speaking, without needing to take the floor.

**Moshi sits here.** The Moshi model emits audio continuously, in lock-step with the user's incoming audio, over its own stream. It generates its own turn in time, not after.

The spectrum is a design choice, not just a technical one. A half-duplex agent feels like a very fast phone operator. A full-duplex agent feels like a human conversational partner — which is either exactly what you want or distinctly unsettling, depending on the product.

## Kyutai Moshi + Mimi: the full-duplex reference

Moshi is the Kyutai Labs open-weights speech-language model that popularised full-duplex. Paper: [Défossez et al., 2024](https://arxiv.org/abs/2410.00037). Code: [github.com/kyutai-labs/moshi](https://github.com/kyutai-labs/moshi). Deploy as self-hosted open-weights; there is no first-party managed API.

### The architecture, at the level you need to pick it

Moshi is built on **two independent token streams** running synchronously:

1. **The user stream.** Audio from the user is tokenised by Mimi (the paired neural audio codec — see Chapter 6 of Module 102 and Chapter 7 of this module) into a sequence of discrete tokens at ~12.5 Hz (one token per 80 ms frame).
2. **The model stream.** The model generates its own audio tokens at the same rate, in parallel. It also generates a hidden "inner monologue" text stream that is used as a planning-and-coherence scaffold inside the model but is not the output surface.

Because both streams tick at ~12.5 Hz on a shared clock, the model can **hear what the user is saying as it is saying it**, and can decide at each frame whether to emit silence, continue its own utterance, backchannel, or interject. There is no turn-taking VAD in the client — turn-taking is an emergent property of the model's output stream. The model has learned when human speakers take turns, backchannel, overlap, and cut each other off.

Latency: Moshi advertises **sub-200 ms response latency** from user speech to the first bytes of model audio. Verify on your hardware — the floor depends on inference latency per 80-ms frame, which depends on the deployment GPU.

Mimi is the paired neural codec: compression down to ~1.1 kbit/s while remaining intelligible, with the specific property that its discrete tokens are the model's input and output vocabulary. The pairing is tight; you do not use Moshi with a different codec.

### What it means for product design

- **Backchannels are free.** The model naturally emits "mhm" during the user's utterance if that is what the context calls for. In user studies this is one of the biggest "feels like talking to a person" wins.
- **Overlapping speech is a first-class interaction.** The user can interrupt mid-word; the model can interject mid-word. There is no turn-taking error.
- **Prosody carries across turns.** Because the model is listening while it speaks, it can adapt tempo and emphasis to the user's prosody in real time.
- **The product UX decision reverses.** Where half-duplex agents are designed to "hand off cleanly," a full-duplex agent is designed to "co-occupy the channel politely." These are different UX disciplines.

### The trade-offs

- **Self-hosted only.** No managed API. You own the inference stack — GPU, deployment region, capacity planning, observability. The Moshi codebase runs on one or a few GPUs per concurrent session; scaling to thousands of concurrent sessions is a real infrastructure problem.
- **English-first.** As of this writing, Moshi is strongest in English; Kyutai has released multilingual variants, but coverage is far below managed models. <!-- needs-research: confirm current Moshi multilingual variants at author time -->
- **Voice selection is baked.** The model has a voice (or a small set); it does not clone.
- **Tool use and reasoning quality** are substantially below a frontier text LLM. Moshi is a research-grade conversational model, not an agent-grade one.
- **Compliance posture is yours.** Because you self-host, you also own HIPAA / GDPR / PCI posture. There is no shared-responsibility model.

### When Moshi is the right pick

- Research or product demos where the full-duplex interaction is the feature.
- Deployments where sub-200 ms is a hard requirement and self-hosting is acceptable.
- Open-weights compliance stories where "the model runs entirely on our infra" is non-negotiable.
- Native-audio features (laughter, whispering, hesitations) that managed vendors do not expose.

Do not pick Moshi for tool-heavy agentic workflows or for languages outside its coverage. Pick it where its full-duplex capability is the point of the product.

## OpenAI Advanced Voice: the managed half-duplex reference

Advanced Voice is the ChatGPT product built on `gpt-4o-realtime`; it is the most-deployed speech-native stack in consumer use. Its duplex regime is **half-duplex with server-side VAD and fast interrupt**.

The stance, concretely:

- **Model emits while user is quiet.** Server VAD watches the input stream; while the model is speaking, incoming user audio is still accepted and transcribed, but the model's response continues until VAD declares the user has started speaking.
- **Interrupt cancels the in-flight response.** On detected user speech, the server fires `response.cancel` under the hood; the model stops generating; the floor swaps.
- **No overlap generation.** The model does not emit "mhm" while the user is still speaking. It can't backchannel in the way Moshi does.
- **Latency floor is roughly 300 ms perceived.** Fast enough that most users do not notice the half-duplex boundary; slow enough that some will — especially on bad audio where VAD mis-fires.

Why half-duplex? Three defensible reasons, each worth naming:

1. **Phone-call mental model.** Users already expect "one of us talks at a time" from decades of phone calls. Half-duplex matches the model.
2. **Safety mitigations.** The GPT-4o System Card describes safeguards the Realtime API applies before emitting audio — refusal routing, voice-cloning prevention, safety-classifier gating. Half-duplex gives the safety stack a well-defined response window; full-duplex would require those checks on every frame.
3. **Compute and latency simplicity.** A half-duplex model generates one audio stream at a time. A full-duplex model generates two in parallel (self + attend to user), which roughly doubles the inference cost per concurrent session.

Chapter 6 reads the GPT-4o System Card on these mitigations.

## Google Gemini Live: between the two

Gemini Live is **bidirectional streaming by design** (the socket is open in both directions; the client can push audio at any time), with **server-side VAD** mediating turn-taking. It does not, today, co-articulate backchannels the way Moshi does — but the socket is not forced to go quiet when the model is speaking, and the model's `interrupted` signal uses continuous user-side audio to pre-empt itself.

Pragmatically, treat Gemini Live as **overlapping half-duplex**: slightly more concurrent than Realtime (the client sends continuously; the server interrupts the model when it detects user speech), without Moshi's per-frame co-articulation.

The practical implications for a design doc:

- **Turn-taking still single-threaded.** You still get a "whose turn" state machine; the model does not emit while the user is speaking.
- **Interrupts are snappy.** Server VAD + `interrupted` is low-latency; closer to Moshi in feel than to a pipeline.
- **Backchannels are not supported.** No "mhm" while the user speaks.

Where you need to mark Gemini Live on the duplex spectrum, mark it between OpenAI Advanced Voice and Moshi — closer to Advanced Voice in behaviour, closer to Moshi in socket shape.

## Downstream implications (preview of later modules)

The duplex choice does not just change the latency number. It changes four downstream engineering surfaces.

### Turn-taking (Module 106)

- **Half-duplex.** Your VAD / turn-detection model is the central piece. You tune `silence_duration_ms`, `threshold`, and (where available) semantic-VAD parameters to balance "do not cut the user off mid-thought" against "the agent feels dead." Two canonical regimes: aggressive VAD (snappier, more false triggers), conservative VAD (slower, fewer interruptions).
- **Full-duplex.** The model handles turn-taking as part of its own emission. You do not tune a VAD. You tune the *model* (prompt, system instructions, fine-tune) to behave appropriately — more or less interjective, more or less willing to overlap. This is a different discipline.

### Latency budget (Module 109)

- **Half-duplex.** The budget is linear: STT (or input encode) + LLM inference + TTS (or output decode) + transport. You optimise each component.
- **Full-duplex.** The budget is per-frame: input encode + model step + output decode, running every 80 ms. The per-frame budget is tight (≈ 50–80 ms on current GPUs) but the end-to-end feel is dominated by the model's decision latency to begin speaking, not the sum of components.

### Consent and recording (Module 108)

A subtle one worth naming now:

- **Half-duplex** interactions can be represented as a single alternating audio track. Consent-to-record and legal interception rules apply to the one track.
- **Full-duplex** interactions are two simultaneous tracks. "Two-party consent" states (US: twelve states have two-party consent laws — California, Florida, Illinois, Maryland, Massachusetts, Michigan, Montana, Nevada, New Hampshire, Pennsylvania, Washington, Connecticut) apply differently when both parties were literally speaking at the same time. This is not a theoretical quibble; it changes the recording retention design. Module 108 returns to it.

### Barge-in cost (Module 106)

- **Half-duplex.** Barge-in is a well-defined event: user speaks, server cancels, floor swaps. Easy to reason about; easy to log.
- **Full-duplex.** There is no single barge-in event. The user ramping up their speech and the model tapering down are both continuous. You log turn boundaries probabilistically; you measure "time-to-silence-after-user-starts" as a distribution, not a value.

## Reading the three stacks side by side

| Dimension | Moshi (Kyutai, open-weights) | Advanced Voice (OpenAI Realtime) | Gemini Live (Google) |
|---|---|---|---|
| Duplex regime | Full-duplex (co-articulated, 12.5 Hz) | Half-duplex, server VAD-mediated | Bidirectional streaming; overlapping half-duplex behaviour |
| Response latency floor | Sub-200 ms | 300–500 ms | 300–700 ms |
| Backchannels (mhm, uh-huh) | Yes, native | No | No |
| Overlapping speech | Yes | No (interrupt cancels) | No (interrupt cancels) |
| Managed? | No (self-host) | Yes (API + ChatGPT) | Yes (API) |
| Multimodal in loop | Audio only | Audio only | Audio + video + screen + text |
| Tool / function calling | Weak | Yes | Yes |
| Voice selection | Fixed (model's voice) | Vendor's shipped voices | Vendor's shipped voices |
| Consent / recording shape | Two simultaneous tracks | One alternating track | Nearly one alternating track |
| Typical use | Research, demos, open-weights compliance | Consumer voice, telephony, API-driven | Vision-in-loop voice, GCP-native deployments |

Fill the columns for your deployment; the right pick usually becomes legible.

## Common anti-patterns

- **"Full-duplex is just lower-latency half-duplex."** It is not. Full-duplex is a different interaction model. Users respond to it differently.
- **Benchmarking Moshi against Advanced Voice on "latency to first response."** The metric is defined differently in the two regimes. In Moshi, the model may begin backchannelling at 200 ms and never make a clean "its response started here" event. In Advanced Voice, there is a clean VAD-driven transition. Measuring one with the other's metric is a category error.
- **Treating full-duplex as always better.** For telephony IVR, for authoritative-tone deployments (medical, legal), for scripted-flow products, half-duplex is the right choice — the clean turn-taking is the feature.
- **Underestimating the consent-flow delta.** A product moving from half- to full-duplex may have to re-clear its recording consent with legal. Do not treat this as a vendor swap.
- **Running Moshi on a laptop GPU for a production deployment.** The per-session GPU cost is real; your scaling story has to budget for it.
- **Not naming the duplex regime in architecture docs.** Reviewers will assume the worst: if the doc does not say "half-duplex, server VAD," someone will assume you meant "full-duplex," get the wrong mental model, and build the barge-in wrong.

## Summary

- The speech-native option is not one architecture; it is a spectrum of duplex regimes. Moshi anchors full-duplex, Advanced Voice anchors managed half-duplex, Gemini Live sits between them with bidirectional streaming but half-duplex turn-taking in practice.
- Moshi's architecture — dual 12.5-Hz token streams on a shared clock, Mimi as the paired neural codec, inner-monologue scaffold — is the reference for what full-duplex speech-language models look like.
- Advanced Voice's half-duplex stance is a deliberate design choice, not a limitation: phone-call UX, safety-stack response windows, inference-cost simplicity.
- Downstream engineering surfaces that the duplex choice reshapes: turn-taking (Module 106), latency budget (Module 109), consent and recording shape (Module 108), and barge-in metrics.
- Name the duplex regime in your architecture doc. Reviewers will otherwise assume the wrong one.

Next chapter: the safety discussion the three vendors publish — GPT-4o System Card, Gemini safety, Anthropic's voice-partner stance — and how those map to the EU AI Act Article 5(1)(f) prohibition on workplace / education emotion inference.
