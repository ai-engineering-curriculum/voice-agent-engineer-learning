# 02. End-of-Utterance Detection Beyond VAD

## Why this chapter

VAD (Chapter 1) tells you when the audio is speech and when it is not. That is not the same as knowing when the **user is done talking**. Humans pause mid-utterance to breathe, to think, to search for a word; a silence-duration threshold alone treats every pause above 500 ms as a turn boundary, which is why early-generation voice agents cut people off mid-sentence. The better question — "has the user finished what they wanted to say, or are they still in the middle of it?" — is a classification problem over the running audio and transcript, and that is what this chapter is about.

Three families answer it today, and a voice-agent engineer should be able to deploy and defend any of them:

1. **Open-weights turn-detector models** — LiveKit's turn-detector is the open reference; a small transformer trained on conversational data to predict "end of turn" vs. "still going" from the transcript suffix.
2. **Semantic end-pointing via a small LLM classifier** — call a cheap LLM every ~200 ms on the current partial transcript with a two-choice prompt. Easiest to deploy; most expensive per decision; highest ceiling.
3. **Voice Activity Projection (VAP)** — self-supervised models from Ekstedt & Skantze that predict the next ~2 s of voice activity from the audio itself, giving you a continuous "likelihood the user will keep speaking" signal.

Each has a distinct operating point and a distinct failure mode. Pick on the latency budget and the deployment constraints; cite the research so a reviewer knows you did not just reach for the first option.

## The failure VAD cannot catch

Consider two utterances with the same silence profile:

- `"So I was thinking … [pause 700 ms] … three options come to mind."` — the user is not done; they're thinking mid-utterance.
- `"So I was thinking of taking the day off tomorrow."` — the user is done.

A silence-duration-500 ms VAD endpoints both the same way: at the end of the 700 ms pause in utterance one, and after the final word in utterance two. On the first utterance the agent interrupts mid-thought ("Sure, what are you thinking?"). The user experience is "the bot keeps cutting me off."

What distinguishes the two is linguistic. Utterance one ends on `thinking`, which is almost never the final word of a conversational turn. Utterance two ends on `tomorrow`, which plausibly is. End-of-utterance detection is the layer that adds this linguistic signal on top of the acoustic one.

Put the mental model in terms of a product: **VAD is the gate; EOU is the referee.** VAD opens the ingress and closes it; EOU decides when the turn is actually over and the response should fire.

## Family 1: open-weights turn-detector models (the LiveKit reference)

LiveKit ships an open-weights turn-detection model inside its Agents framework. See [docs.livekit.io/agents/build/turns/turn-detector/](https://docs.livekit.io/agents/build/turns/turn-detector/) for the current contract. The model takes the running transcript (ASR partials) and returns a probability that the current token position is a turn boundary — i.e., a lightweight next-token-style classifier over conversational-end-of-turn labels.

### The architecture, at engineer altitude

- **Input.** The running partial transcript, usually the last N tokens of the user's current turn.
- **Output.** `P(end-of-turn | transcript suffix)`.
- **Signal combination.** Downstream, the pipeline combines the VAD's "silence has lasted X ms" with the EOU model's probability into a single commit decision. A common shape: `commit_turn ← silence_ms ≥ base_silence_ms AND P(EOU) ≥ threshold`. Equivalently, modulate the silence duration itself by the EOU probability — require 200 ms of silence when `P(EOU) = 0.95`, 1000 ms when `P(EOU) = 0.3`.
- **Runtime.** Small enough to run on CPU alongside the agent process; sub-10 ms per inference. The model shipped as part of the Agents framework; license and weights are published on Hugging Face.

<!-- needs-research: confirm the current LiveKit turn-detector model repository URL, parameter count, and license at author time against docs.livekit.io -->

### When to pick it

- You already run LiveKit Agents and want a drop-in EOU with no additional vendor cost.
- Your transcript stream is reliable (good ASR partials). The model classifies the transcript; garbled transcripts lower its quality.
- You need deterministic latency. A local model is milliseconds; an LLM call is hundreds of milliseconds minimum.

### Where it fails

- Domain mismatch. A model trained on general conversational data will be weaker on highly technical domains (code dictation, medical shorthand) where a casual-sounding word is actually a continuation.
- Short utterances. On "yes" / "no" / "mm-hm" the suffix is too short to classify reliably; fall back to silence-duration + VAD alone for ultra-short turns.
- Multilingual. The current open models are English-strong; coverage of other languages varies.

## Family 2: semantic end-pointing with a small LLM classifier

The simplest EOU you can ship is a two-way classifier prompt against a small model, run every ~200 ms on the running partial transcript:

```jsonc
{
  "model": "gpt-4o-mini",         // or claude-haiku, gemini-flash — any sub-second-latency small model
  "temperature": 0,
  "max_tokens": 1,
  "system": "You are deciding whether a speaker has finished their turn. Output exactly one character: 'Y' if the transcript below is a plausible complete utterance a speaker would pause on, 'N' if the speaker is mid-utterance and likely to continue.",
  "user": "{running_partial_transcript}"
}
```

Two observations about this shape.

**It works.** A frontier-generation small LLM correctly classifies turn-end on natural conversational transcripts at > 90 % agreement with human labellers in most domains. The model has strong priors on what a complete sentence sounds like.

**It has an unavoidable cost floor.** Every poll is a billed API call. At 200 ms poll intervals on a two-minute call you burn 600 calls per session. Even at small-model pricing this is non-trivial on a contact-centre deployment with millions of minutes per month. Two mitigations that nearly always pay off:

1. **Gate the poll by VAD.** Only call the classifier when VAD says silence-has-started. One call per candidate endpoint, not one every 200 ms.
2. **Cache the suffix.** The input prompt repeats every call; prompt caching recovers most of the cost. Keep the system prompt and the first few thousand tokens of context stable.

### A hybrid that ships well

The pattern most teams converge on:

```
on_vad_silence_start:
    # VAD has detected silence. Wait the base hangover, then ask the classifier.
    wait base_silence_ms  # e.g. 200 ms
    p_eou = llm_eou_classifier(current_transcript)
    if p_eou >= 0.8:
        commit_turn()
    elif p_eou >= 0.4:
        wait extra_silence_ms  # e.g. 400 ms more
        if still_silent:
            commit_turn()
    else:
        wait long_silence_ms   # e.g. 1200 ms — user is clearly thinking
        if still_silent:
            commit_turn()
```

The LLM is doing the thing silence duration can't: adjusting the hangover based on whether the transcript looks finished. A short confident `Y` ships the turn in < 400 ms total; an uncertain middle case waits; a confident `N` on a visibly mid-thought transcript holds the turn open for a full second.

### When to pick it

- You already have a small-LLM infrastructure inside the voice loop (every pipeline does).
- Your domain is hard to collect training data for (medical, legal, long-tail enterprise). The LLM's priors generalise better than a trained classifier on 10 hours of your data.
- You want to iterate on behaviour via prompt changes, not model retraining.

### Where it fails

- Cost on high-volume deployments without caching. Measure.
- Latency floor. Even a fast small LLM costs 150–400 ms per call. If your end-to-end response budget (Module 109) is tight, you cannot afford an EOU call *and* a response generation call back-to-back.
- Streaming transcript races. The partial transcript at the moment of the poll is not the final transcript; words finalise after. Classify on the stabilised suffix (words that have not changed in the last 200 ms of ASR output), not on everything the ASR has emitted.

## Family 3: Voice Activity Projection (VAP)

Where the LiveKit model classifies the transcript and the LLM classifier classifies language, VAP classifies the **audio itself** in a self-supervised way. The question it answers is sharper than EOU: "given the audio of the last several seconds, predict the pattern of voice activity for both speakers over the next ~2 seconds."

- **Reference paper.** Ekstedt and Skantze, "Voice Activity Projection: Self-supervised Learning of Turn-taking Events," ICASSP 2022 — [arXiv:2205.09812](https://arxiv.org/abs/2205.09812).
- **Follow-ups.** The VAP line has continued ("Multilingual Voice Activity Projection," "Real-time Voice Activity Projection" and stream-first variants); see [Erik Ekstedt's repo](https://github.com/ErikEkstedt/VoiceActivityProjection) and the Skantze group's publications.
- **How it differs from VAD.** VAD classifies the current frame; VAP projects forward. Instead of saying "is the user speaking right now," it outputs a distribution over who will be speaking over the next N frames, including expected overlap and gap patterns.

### The output VAP gives you

VAP discretises the next window into bins of (speaker A active / speaker B active) and outputs probabilities over those bins. From the distribution you can derive:

- `P(user keeps talking)` — direct turn-hold signal.
- `P(user yields turn soon)` — direct turn-end signal.
- `P(overlap imminent)` — backchannel or barge-in likelihood (Chapter 4 uses this).
- Expected gap-to-next-speech — useful for pacing bot responses.

That is a richer signal than a single EOU probability, and it is grounded in prosody (not just transcript). A speaker lowering their pitch and lengthening their final vowel is a turn-end cue VAP can read directly from the audio — a transcript-only model cannot.

### When to pick it

- You have multimodal ground truth (audio + conversation-structure labels) and the appetite to train / fine-tune.
- Your users speak non-standard English, or your domain's turn-taking is prosody-rich (coaching, therapy, interviews) and transcript-only EOU is missing it.
- You need a signal for backchannel vs. interruption (Chapter 4), and want one upstream of the turn-grab classifier.

### Where it fails

- Not plug-and-play. VAP is a research line, not a packaged product. Deploying it means picking a checkpoint, measuring on your audio, and often training a small head on your data.
- Latency. The projection window is 2 s; inference is per-frame but the signal "the user is yielding the turn" is only as confident as the model's recent window. Fine for turn-hold detection, trickier for fast-reaction barge-in than a transcript classifier.

### The research fit

VAP is the modelling lineage you point a reviewer to when they ask "what is the right way to think about turn-taking as a learning problem?" Skantze's 2021 survey ([Skantze, "Turn-taking in Conversational Systems and Human-Robot Interaction: A Review," Computer Speech & Language 67](https://doi.org/10.1016/j.csl.2020.101178) — [preprint](https://www.csc.kth.se/~skantze/papers/Skantze_CSL_2020.pdf)) frames the gap between traditional silence-based end-pointing and prediction-based turn modelling, and the VAP work is the mainstream answer to the gap. Chapter 6 revisits the survey.

## Composing the signals: the EOU decision loop

Nothing stops you from running two or all three families. A production-grade decision loop combines them:

```
# Inputs updated every frame
vad_silence_ms          # from Silero or vendor VAD
p_eou_transcript        # from LiveKit turn-detector or small-LLM classifier
p_hold_audio            # from VAP, if deployed

def should_commit_turn():
    # Hard floor: at least 200 ms of silence.
    if vad_silence_ms < 200: return False
    # Transcript-side veto.
    if p_eou_transcript < 0.3: return False
    # Audio-side veto.
    if p_hold_audio is not None and p_hold_audio > 0.7: return False
    # Three progressive hangover bands, driven by EOU confidence.
    if p_eou_transcript >= 0.8 and vad_silence_ms >= 200: return True
    if p_eou_transcript >= 0.5 and vad_silence_ms >= 500: return True
    if vad_silence_ms >= 1200: return True
    return False
```

The structure, not the numbers, is the point. The numbers are tuned per deployment against the metric suite in Chapter 5.

## Vendor server-side EOU: `semantic_vad` on OpenAI Realtime

OpenAI's Realtime API exposes `turn_detection.type: "semantic_vad"` alongside `server_vad`. The semantic variant is the vendor's internal EOU — a small server-side model that considers transcript context in addition to silence. The schema and parameters evolve; cross-check the [Realtime reference](https://platform.openai.com/docs/api-reference/realtime).

What you get when you enable it:

- A turn-detector that handles most mid-utterance pauses correctly without you shipping a classifier.
- No code on your side; it's a session-config switch.
- Opaque behaviour — you don't see the probability, only the committed turn.

What you give up:

- Visibility. You can't stratify its errors.
- Tuning. The knob surface is thinner than the manual-commit path with your own EOU model.
- Vendor-lock on a specific behaviour you cannot carry across to Gemini Live or to your pipeline backup.

Appropriate for consumer voice agents on the speech-native stack where good-enough mid-pause handling is the whole requirement. Replace with Family 1 / 2 / 3 when you need observability or when the vendor's behaviour is wrong for your audio.

## Operational realities

### Partial transcript stability

ASR partials rewrite. The ten seconds of partial transcripts emitted during a single user utterance do not converge monotonically; word identities change, punctuation appears and disappears. If you call your EOU classifier on the raw partial stream, you will classify the same window multiple times with different inputs.

Pattern: wait for the ASR to emit a stabilised suffix — words that have not been revised for the last 200–300 ms — and classify on that. Deepgram's `is_final` partials, AssemblyAI's `final` messages, and Whisper's `no_speech_prob` are the primary signals. See Module 103 Chapter 2 for the streaming contract details.

### The transcript-arrival-vs-audio lag

Streaming ASR transcripts lag the audio by a provider-dependent amount — typically 150–400 ms for competitive providers, sometimes more. If your EOU classifier runs on transcript, your commit decision is at least that late. Silence duration runs on audio directly and is zero-lag. The silence floor in your decision loop is what makes you snappy; the EOU vote is what makes you correct.

### Cross-talk and overlap

What if the bot is speaking while you're trying to classify the user's turn boundary? That's the barge-in case, Chapter 3. The short answer: during bot-speaking, VAD on the user channel runs; EOU doesn't need to, because any user speech of sufficient length is an interrupt event, not a turn boundary.

### Observability

For every committed turn, log: `vad_silence_ms`, `p_eou_transcript`, `p_hold_audio` (if present), and the final decision. When users complain the bot cut them off, you reproduce from the log, not from guessing.

## Common anti-patterns

- **"Just make `silence_duration_ms` bigger."** You've swapped one failure (cut off mid-thought) for another (bot feels dead at end of utterance). The correct fix is variable hangover driven by EOU, not longer fixed hangover.
- **Classifying EOU on every ASR partial.** Expensive and noisy. Gate on VAD-silence-start; classify on a stabilised suffix.
- **Shipping an LLM-classifier EOU without prompt caching.** The input prefix repeats every call; the uncached cost is many times the cached cost.
- **Using VAP output as a direct commit signal.** VAP's output is a distribution over the next window, not a hard "turn over" decision. Compose it with VAD and transcript signals; don't replace them.
- **No per-turn log of the EOU inputs.** When the bot interrupts, you need the (silence_ms, p_eou, p_hold) triple that fired, not a hunch.
- **Tuning EOU without a ground-truth corpus.** The metric suite in Chapter 5 is how you measure false-turn-grab and missed-EOU. Author the labels before you tune the knobs.

## Summary

- VAD is a speech-vs-silence frame classifier; EOU answers "is the user done." A silence-duration-only pipeline interrupts natural speakers.
- Three families: (1) open-weights turn-detector models like LiveKit's — fast, local, transcript-based; (2) small-LLM classifiers on the running transcript — flexible, prompt-tunable, higher latency and cost per call; (3) Voice Activity Projection (Ekstedt & Skantze, ICASSP 2022) — audio-based, prosody-aware, richer signal, research-grade deployment cost.
- Compose signals rather than choosing one. A production decision loop combines silence duration with transcript-side and (optionally) audio-side EOU scores, with a variable hangover.
- OpenAI Realtime's `semantic_vad` is the vendor-side shortcut; use when visibility isn't required.
- Operational realities: partial transcripts rewrite (classify on a stabilised suffix), transcript stream lags audio (silence stays the snappy signal), log every commit decision (observability is how you tune).
- The research-fit: Skantze 2021 survey + VAP 2022 are the papers that frame this as a prediction problem, not a silence-threshold problem. Chapter 6.

Next chapter: the interrupt path — user interrupts bot (cancel in-flight TTS, truncate the LLM generation, preserve partial state), bot interrupts user (backchannels, "go ahead" cues, non-speech-interrupt rules), targeting sub-200 ms interrupt reaction.
