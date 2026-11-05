# 06. Reading the Turn-Taking Literature at Engineer Altitude

## Why this chapter

Chapters 1–5 are engineering: what to deploy, how to tune it, how to measure. This chapter is the literature chapter. The turn-taking choices you made (silence-duration hangover, EOU classifier, VAP, barge-in threshold, backchannel classifier) are all defensible — but only if you can point at the sources that motivated them. A reviewer asking "why Silero over WebRTC VAD, and why a VAP-style predictor over semantic end-pointing?" expects an answer that cites the work, not an answer that recites your choice.

We stay at engineer altitude. The point is not to reproduce the models or derive their loss functions — that is `nlp-engineer` and `model-evaluation-engineer` territory (see the boundary at the end of this chapter). The point is to read the research well enough to (a) defend the architectural decision to a peer reviewer, (b) spot when a paper's framing does not match your deployment, and (c) hand the deep model-internal questions to the right ladder-adjacent role.

Three works carry this chapter's reading list:

1. **Skantze, "Turn-taking in Conversational Systems and Human-Robot Interaction: A Review," Computer Speech & Language 67 (2021).** The survey that frames turn-taking as a prediction problem rather than a silence-threshold problem. Required for the vocabulary alone.
2. **Ekstedt and Skantze, "Voice Activity Projection: Self-supervised Learning of Turn-taking Events," ICASSP 2022** ([arXiv:2205.09812](https://arxiv.org/abs/2205.09812)). The model family you reach for when transcript-only EOU is not enough.
3. **LiveKit turn-detector methodology.** The open-source industrial reference. Not a paper per se; a model card + blog posts + code. What a shipped open-weights turn-detector actually looks like in 2025.

We also point to the adjacent reading — Duncan's 1972 signalling work, the ICSI MRDA / SwDA dialog-act traditions, more recent neural-end-pointing papers — so a motivated engineer can go deeper when the deployment warrants.

## 1. Skantze 2021: the survey

**Citation.** Skantze, Gabriel. "Turn-taking in Conversational Systems and Human-Robot Interaction: A Review." *Computer Speech & Language* 67 (2021), 101178. DOI [10.1016/j.csl.2020.101178](https://doi.org/10.1016/j.csl.2020.101178). Preprint typically available on Skantze's KTH page.

### What it is

A ~30-page survey of turn-taking from the perspectives of conversation analysis (Sacks, Schegloff, Jefferson), dialog systems engineering, and the then-recent neural approaches. The paper is structured around the gap between how humans manage turns (predictively, using prosodic and lexical cues to anticipate turn-end hundreds of milliseconds before silence) and how dialog systems manage turns (reactively, waiting for silence of threshold duration).

### What to pull from it

1. **The "silence is a weak signal" framing.** The survey makes the case — with references — that silence-threshold endpointing is intrinsically a compromise. Mid-utterance pauses are a normal feature of natural speech. Speakers project turn-ends via prosody (falling pitch, lengthened final syllable), gaze (in multimodal settings), and lexical structure. A silence-threshold detector sees none of this and must trade false-turn-grabs for missed-EOUs. This is Chapter 1 and 2's vocabulary; the survey is where it comes from.

2. **The "projection" view.** Turn-taking is a prediction problem: what will the other speaker do over the next several seconds? The paper names "turn-end prediction" and "turn-shift prediction" as distinct tasks, and surveys early neural approaches to each. The VAP paper (next section) is a direct descendant of this framing.

3. **The dialog-act layer.** Not every utterance is a turn-taking move. Dialog-act-labelling traditions (SwDA, MRDA, DAMSL) distinguish statements, questions, backchannels, and continuers. The survey's Section on backchannels is the shortest introduction to the backchannel inventory you'll need for Chapter 4.

4. **The multimodal section.** Gaze, head-nod, gesture, and body orientation are turn-taking signals in face-to-face conversation. For a voice-only agent most of this is out of scope — but when you ship a multimodal voice agent (Module 110), it is back in scope. The survey is where you remember the vocabulary is there.

### What to skip

The survey's dialog-system examples are from a pre-LLM era; many specific systems cited (POMDPs, hand-tuned rule systems) are not the engineering context today. Read the frame, skim the specific-system descriptions.

### How to defend your choices from it

- "Why variable-hangover EOU instead of fixed `silence_duration_ms`?" — Skantze §3 on the prosodic-projection view; silence duration is a compromise, and compromises with named failure modes should be made explicit and variable, not treated as a single tunable.
- "Why a backchannel classifier at all?" — Skantze §4 on backchannels; the phenomenon is pervasive and regular enough to classify.
- "Why consider a prosody-aware turn-end predictor?" — Skantze §3.2 on projection; the human performance ceiling for turn-end detection is well above what silence-threshold detectors reach.

## 2. Ekstedt & Skantze 2022: Voice Activity Projection

**Citation.** Ekstedt, Erik and Skantze, Gabriel. "Voice Activity Projection: Self-supervised Learning of Turn-taking Events." *ICASSP 2022*. arXiv [2205.09812](https://arxiv.org/abs/2205.09812).

Also: Ekstedt's PhD-era publications and the repository at [github.com/ErikEkstedt/VoiceActivityProjection](https://github.com/ErikEkstedt/VoiceActivityProjection).

### What it is

A self-supervised modelling approach for turn-taking. The model observes two channels of audio (speaker A, speaker B) and predicts the discretised pattern of voice activity over the next ~2 seconds. The labels are free — they come from the audio itself (future voice activity = future frames of the recording).

### The idea in one paragraph

Instead of training a model to emit "turn-end" labels (which requires hand-labelled turns, expensive), discretise the next 2 s of per-channel voice activity into a small bin alphabet (around 256 bins in the original) covering combinations of (A active, B active) over the next window. Train a model to predict the distribution over bins. At inference, derive turn-taking events from the distribution: `P(A keeps talking)`, `P(B takes turn)`, `P(overlap)`, `P(gap)` — all are projections of the per-bin probabilities.

### Why engineers care

- **The output is rich.** You get a turn-hold probability, a turn-shift probability, a backchannel-likelihood signal, and an overlap-prediction signal from one model. One architecture replaces the EOU model + the barge-in classifier + the backchannel classifier.
- **Prosody is baked in.** The model trained on audio sees pitch, energy, and speech-rate patterns the transcript-only models cannot. Falling pitch is a turn-end cue it learns automatically.
- **Self-supervised.** Any corpus of two-channel conversational audio is training data. You do not need hand-labelled turns.

### Where it bites

- **Not plug-and-play.** You deploy a research-grade PyTorch model, tune for your audio, and operate a model-serving path that most pipelines do not already have.
- **Latency.** The projection window is forward-looking; the signal "the user is about to yield" is confident around the moment they actually yield, not before. Good for turn-end; weaker for hair-trigger barge-in.
- **English-centric baseline.** The original paper's data is English conversational telephony. Multilingual VAP work has followed (see Ekstedt's follow-ups) but is less mature.

### How to defend a VAP deployment from it

- "Why trust the model's hold/yield signal?" — the VAP paper's §4 evaluation shows the model exceeds silence-threshold baselines on the Switchboard turn-shift task.
- "Why not just a bigger EOU classifier?" — the paper's framing: EOU classifies at a point in time; VAP predicts the next window. The forward-looking signal is what distinguishes turn-end from turn-hold under mid-utterance pauses.

### Follow-ups worth naming

- **Real-time / streaming VAP** — subsequent work by the same group adapts the model for streaming use (not just offline evaluation). <!-- needs-research: confirm the current title and citation of the streaming VAP work at author time -->
- **Multilingual VAP** — cross-language transfer of the projection objective. <!-- needs-research: confirm the current title and citation of the multilingual VAP work at author time -->
- **VAP in robot interaction** — Skantze's group has deployed VAP in human-robot interaction experiments; useful if your product shape is embodied rather than voice-only.

## 3. LiveKit's turn-detector: the open industrial reference

**Source.** [docs.livekit.io/agents/build/turns/turn-detector/](https://docs.livekit.io/agents/build/turns/turn-detector/), plus the model card on Hugging Face and the LiveKit engineering blog.

### What it is

A small open-weights transformer trained to predict whether a given transcript suffix is a turn boundary. Integrates into the LiveKit Agents framework's turn-detection pipeline, running alongside a VAD and combining with its silence-duration signal.

<!-- needs-research: confirm the current LiveKit turn-detector model parameter count, training data description, and license at author time against the LiveKit docs and the model card -->

### Why read the methodology

Not for the model architecture per se (it is a small transformer; the architecture is not the lesson). The lesson is in the *deployment* decisions LiveKit made and published:

- **It takes transcript, not audio.** Classification is on the stabilised partial transcript emitted by the upstream STT. The engineering constraint: the ASR partial-emission latency is the EOU model's input latency. Audio-only approaches (VAP) are not constrained by transcript latency; transcript-based approaches are.
- **It composes with VAD, not replaces it.** The LiveKit pipeline uses silence duration as the hard floor and the EOU probability as the modulator — exactly the variable-hangover pattern in Chapter 2. The EOU does not fire the turn by itself; silence has to also be present.
- **It is intentionally small.** CPU-runnable, deterministic latency, deployable next to the Agents worker. A larger model gated behind an API call would add latency and operational surface for marginal quality.
- **It is swappable.** The Agents framework defines a turn-detection interface; the open turn-detector is one implementation among others (silence-only, vendor semantic VAD, VAP, custom). Deployment pattern: integrate behind an interface so you can A/B swap implementations.

### Reading tasks

- Read the current docs page end-to-end. Note the integration contract (interface, inputs, outputs).
- Read the model card. Note the training data description, language coverage, and any published evaluation numbers.
- Read the Agents framework's turn-detection abstraction in the source code. Note where the EOU probability combines with VAD silence duration in the commit decision.

### How to defend a LiveKit-turn-detector choice

- "Why open-weights over vendor semantic VAD?" — observability. You see the probability, you can stratify by condition, you can swap the model.
- "Why this model over a small-LLM classifier?" — latency and determinism; local inference is milliseconds; LLM calls are hundreds of milliseconds plus variable network.
- "Why this model over VAP?" — operational simplicity. Transcript-based; CPU-runnable; no model-serving fleet needed.

## Adjacent reading worth naming

Not required for the engineering altitude but useful when a specific deployment pushes deeper.

### Conversation analysis — the foundational theory

- **Sacks, Schegloff, Jefferson, "A Simplest Systematics for the Organization of Turn-Taking for Conversation," Language 50 (1974).** [DOI 10.2307/412243](https://doi.org/10.2307/412243). The foundational paper on conversational turn-taking. Dense, linguistic, but cited by nearly every turn-taking paper since. Read for the vocabulary: Transition-Relevant Place (TRP), current-speaker-selects-next, etc.
- **Duncan, "Some signals and rules for taking speaking turns in conversations," J. Pers. Soc. Psychol. 23 (1972).** [DOI 10.1037/h0033031](https://doi.org/10.1037/h0033031). The signalling account: pitch drop, drawl, syntactic completion, gesture, loudness. A compact catalogue of the cues a turn-detector (human or machine) actually reads.

### Dialog acts and backchannels

- **Stolcke et al., "Dialogue Act Modeling for Automatic Tagging and Recognition of Conversational Speech," Computational Linguistics 26 (2000).** [aclanthology.org/J00-3003](https://aclanthology.org/J00-3003/). The Switchboard Dialog Act (SwDA) tag set; your source of truth for backchannel labels in the Switchboard data.
- **Jurafsky, Shriberg, Biasca, "Switchboard SWBD-DAMSL Shallow-Discourse-Function Annotation Coders Manual,"** [draft 13, 1997](http://web.stanford.edu/~jurafsky/ws97/manual.august1.html). The labelling guide; readable, and the source of the operational definitions for backchannels.
- **Shriberg et al., "The ICSI Meeting Recorder Dialog Act (MRDA) Corpus,"** 2004, [aclanthology.org/W04-2319](https://aclanthology.org/W04-2319/). Dialog acts for multi-party meeting audio.

### Neural end-pointing and turn-end prediction

- **Shriberg, Stolcke, et al., "Prosody-based automatic detection of annoyance and frustration in human-computer dialog," ICSLP 2002.** An early example of prosody-based classification in spoken dialog.
- **Masumura et al., "Online end-of-turn detection from speech based on stateful sequence labeling," Interspeech 2017** — a neural end-pointing baseline; worth reading for the before-VAP state of the art. <!-- needs-research: verify exact title / venue / authors of a canonical pre-VAP neural end-pointing paper at author time -->
- **Roddy, Skantze, Harte, "Multimodal continuous turn-taking prediction using multiscale RNNs," ICMI 2018** — [arXiv:1808.10622](https://arxiv.org/abs/1808.10622). A direct predecessor to VAP.

### Overlap and backchannel in multi-party

- **Shriberg, Ostendorf, Stolcke, "Backchannels revisited: Acoustic features, user perceptions, and personality evidence,"** Interspeech corpora work on backchannel prediction; useful for the acoustic-cue inventory.
- **Hara, Inoue, Takanashi, Kawahara, "Backchannel generation model for a third party listener agent," HAI 2019.** Robot-interaction backchannel generation; useful if your product generates backchannels rather than only receiving them.

### Vendor + open-source engineering references

- **OpenAI Realtime** — [platform.openai.com/docs/api-reference/realtime](https://platform.openai.com/docs/api-reference/realtime). The `turn_detection` field and semantics.
- **Gemini Live** — [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live). The `automatic_activity_detection` block.
- **Pipecat** — [docs.pipecat.ai](https://docs.pipecat.ai/). The pipeline framework's VAD and turn-management primitives.
- **Silero VAD** — [github.com/snakers4/silero-vad](https://github.com/snakers4/silero-vad). The current open-weights VAD default.
- **Pyannote** — [github.com/pyannote/pyannote-audio](https://github.com/pyannote/pyannote-audio). VAD, diarisation, overlap detection ([Bredin et al., 2020](https://arxiv.org/abs/1911.01255)).

## How to read research as a voice-agent engineer

Three habits that keep this reading tractable.

**1. Read the abstract and the evaluation section first.** The abstract tells you the claim; the evaluation tells you whether the claim stacks up on data that resembles your deployment. If the eval is on 8 kHz noisy telephony and you ship 16 kHz headset audio, the paper's numbers are only a vague guide.

**2. Treat every "we achieve X% on Y" as a target for your own replication.** Vendor and paper numbers are reported in their best conditions. The numbers are not wrong; they just don't predict your numbers. Replicate on a sliver of your own corpus before trusting the claim.

**3. Note the conditions the paper does not cover.** Low-SNR, multilingual, overlap-heavy, children's voices, elderly users, highly technical vocabularies — if the paper's eval does not include your condition, the methods have not been evaluated in your condition. That is the thing that bites a production deployment.

## The boundary to adjacent roles

This module is engineer-altitude. The following questions belong to adjacent roles on the ladder:

- **Model architecture internals, pretraining, loss design.** The `nlp-engineer` (L30) owns the "why a 256-bin discretisation of the next 2 s window" or "why this transformer encoder depth" questions. They also own deep ASR-architecture questions (Module 103 Chapter 7 names them).
- **Statistical methodology, calibration at depth, significance testing.** `model-evaluation-engineer` (L30) owns the "is this p95 change statistically meaningful" and "how do we compute a confidence interval for FTGR on a small corpus" questions.
- **Training-pipeline design for VAP or turn-detector fine-tunes.** `training-pipeline-engineer` (L35) owns the data pipeline, training infrastructure, and continuous-retrain cadence if you fine-tune a VAP or turn-detector on your own data.
- **Dataset licensing, corpus construction methodology, annotation schemes.** A dedicated data-annotation role, where your org has one, or `model-evaluation-engineer` by default.

A voice-agent engineer reads these papers well enough to:

- Pick between the three deployment families (open turn-detector, LLM-classifier EOU, VAP).
- Defend the pick in a design review.
- Recognise when the problem is outside the voice-agent lane and route to the right peer.

That is the altitude. Going deeper is correct when the deployment demands it; going deeper by default is scope creep.

## Reading plan for the module

Suggested order for a first pass, ~6–8 hours:

1. Skantze 2021 survey — read §§ 1, 3, 4; skim 2, 5. ~90 minutes.
2. Ekstedt & Skantze 2022 VAP — read in full; the model is small enough to internalise. ~90 minutes.
3. LiveKit turn-detector docs + model card + the Agents framework's turn-detection module source. ~60 minutes.
4. SwDA manual's backchannel section + Jurafsky et al. 1997 shallow-discourse. ~30 minutes.
5. Sacks/Schegloff/Jefferson 1974 excerpts (the TRP and turn-allocation sections). ~45 minutes.
6. Vendor docs — OpenAI Realtime `turn_detection`, Gemini Live `automatic_activity_detection`. ~30 minutes.
7. Silero VAD repo + Pyannote repo top-level reads. ~30 minutes.

Return to the papers whenever the metric suite (Chapter 5) points at a specific failure mode. The literature is the lookup table for "what is the right way to think about this failure?"

## Common anti-patterns

- **Reading only the vendor docs.** You get the knobs but not the vocabulary. A reviewer's "why" question needs the research-side framing.
- **Reading only the papers.** You get the vocabulary but not the pragmatics. The papers do not report the AEC-leak barge-in failure mode or the player-flush latency.
- **Treating one paper's benchmarks as the answer.** Numbers are conditional on the paper's dataset; your dataset is different.
- **Deferring deep model questions by not routing them.** "I don't know, that's `nlp-engineer`" is the correct answer; "I'll guess" is the wrong one.
- **Reading the 1974 Sacks/Schegloff/Jefferson paper in full as the first thing.** It is a hard read; start with Skantze's survey and let it point you back when needed.

## Summary

- Three load-bearing references: Skantze 2021 (the framing: turn-taking as prediction), Ekstedt & Skantze 2022 VAP (the model family for audio-based turn prediction), and the LiveKit turn-detector docs + model card (the shipped open-weights reference).
- The deeper reading is adjacent: Sacks/Schegloff/Jefferson 1974 and Duncan 1972 for the theory; SwDA / MRDA for dialog-act labels; follow-up VAP papers for streaming and multilingual extensions.
- Read the abstract and the evaluation section first; the claim matters only if the eval resembles your deployment. Replicate on your own corpus before trusting any number.
- The boundary to adjacent roles: model internals to `nlp-engineer`, statistical methodology to `model-evaluation-engineer`, training infrastructure to `training-pipeline-engineer`. Route; do not guess.
- Suggested first pass: ~6–8 hours over the three load-bearing sources plus the shorter adjacent reading. Return to the literature whenever Chapter 5's metrics flag a failure mode you don't have a framing for.

This closes the lecture track for Module 106. The exercises will walk you through deploying Silero and tuning thresholds, building a semantic-EOU classifier, authoring the barge-in path, designing the backchannel classifier, and authoring the metric suite — each one exercising the vocabulary you now have from the papers.
