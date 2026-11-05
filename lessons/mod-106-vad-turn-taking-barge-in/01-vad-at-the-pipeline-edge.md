# 01. Voice Activity Detection at the Pipeline Edge

## Why this chapter

Every voice agent needs an answer to one question, re-asked tens of times a second: **is the person on the other end speaking right now?** That one bit — "speech / not speech" — gates STT billing, drives the first pass of turn-end detection, triggers barge-in, and in the speech-native stacks (OpenAI Realtime, Gemini Live) decides whether the server commits a turn and starts generating a response. A VAD that is 50 ms too slow or 5 % too eager is a felt defect: the agent clips users, talks over them, or stares silently at background noise until the user gives up.

This chapter is the working depth on VAD for a voice-agent engineer at L35. You will leave able to (a) pick between Silero, WebRTC VAD, and server-side VAD for a given deployment, (b) tune the three knobs that actually matter — threshold, hangover / silence duration, and minimum speech length — against a representative audio corpus, and (c) name the failure modes each VAD exposes so you can diagnose them from telemetry.

VAD alone is **not** end-of-utterance detection. Silence is a weak signal of "the user is done" in natural speech, where people pause mid-thought all the time. Chapter 2 is where EOU proper lives. This chapter is the frame stage before that: the thing that watches the audio at 10 ms resolution and emits a `speech / not-speech` decision per frame.

## Where VAD sits in the pipeline

In a classical pipeline:

```
mic ─► capture ─► [VAD] ─► STT ─► LLM ─► TTS ─► speaker
                     │
                     └──► barge-in detector (while bot is speaking)
```

Two uses, one detector:

1. **Gating STT ingress.** When the user is silent, you do not want to be streaming silence frames at a per-minute-of-audio billed ASR. VAD opens the gate when speech starts and closes it (after a hangover) when speech ends.
2. **Detecting barge-in.** While the bot is speaking, VAD on the mic input tells you the user has started talking, and the barge-in path (Chapter 3) kicks in.

In a speech-native pipeline (Realtime / Gemini Live), the VAD lives inside the vendor's server by default but the same knob surface is exposed:

```
mic ─► capture ─► vendor socket
                     │
                     ├──► server VAD (tunable: threshold, silence_duration_ms, prefix_padding_ms)
                     └──► barge-in cancel in-flight response
```

You still deploy VAD at the edge for two reasons even when the server has one: (1) to decide whether to send audio at all (don't pay to stream pure noise on a telephony leg), and (2) to run your own barge-in logic when the vendor's cancel-on-interrupt is too slow or too cautious. Chapter 3 covers the handoff.

## The three VAD stacks to know by name

### Silero VAD — the open-weights baseline

- **What it is.** A small (<2 M params) convolutional + recurrent neural VAD, distributed as ONNX / PyTorch / JIT weights under MIT.
- **Repo.** [github.com/snakers4/silero-vad](https://github.com/snakers4/silero-vad).
- **Operating point.** Frame-level speech-probability output at 16 kHz with a 30 ms window (the default; 8 kHz support is available for telephony audio). You stream PCM in; it returns a float in [0, 1] per window.
- **Why it is the default.** Compared with energy-based VADs it tolerates non-speech noise (fans, keyboard, background TV) far better, and it generalises across accents and languages better than hand-crafted spectral baselines. It runs in ~1 ms per frame on a modern CPU — no GPU required — which is what makes it viable at the pipeline edge.
- **Where it fails.** Music, singing, and highly stylised audio can produce false positives. Very short utterances (< 200 ms — "yes", "no", "mm") get clipped if you tune the minimum-speech knob too conservatively.

A minimal integration sketch (PyTorch):

```python
import torch

model, utils = torch.hub.load(
    repo_or_dir="snakers4/silero-vad", model="silero_vad", trust_repo=True
)
(get_speech_timestamps, _, read_audio, *_ ) = utils

wav = read_audio("sample.wav", sampling_rate=16000)
speech = get_speech_timestamps(
    wav, model,
    sampling_rate=16000,
    threshold=0.5,            # probability threshold for "speech"
    min_speech_duration_ms=250,
    min_silence_duration_ms=100,
    speech_pad_ms=30,          # pre/post-roll so we don't clip
)
# speech == [{"start": sample_idx, "end": sample_idx}, ...]
```

For streaming use (one audio chunk at a time) you drive the model's iterator form; see the repo's `VADIterator` example. Target frame rate: a decision every 30 ms of audio, chunk size 512 samples at 16 kHz is the canonical setting.

### WebRTC VAD — the historical compare-point

- **What it is.** The energy + spectral GMM VAD shipped inside WebRTC's `audio_processing` module. ≈300 lines of C; no training data; four aggressiveness modes (0–3). Introduced with WebRTC in the early 2010s.
- **Python wrapper.** [py-webrtcvad](https://github.com/wiseman/py-webrtcvad).
- **Operating point.** 10, 20, or 30 ms frames at 8 / 16 / 32 / 48 kHz. Returns a hard `True / False` per frame; no probability.
- **Why it is still cited.** It is free, fast (microseconds per frame), ships inside the WebRTC stack you likely already carry for transport, and was the de facto default before neural VADs matured. Any production voice agent before ~2022 almost certainly ran it.
- **Why we no longer default to it.** On realistic telephony audio (background TV, keyboard clatter, HVAC, coffee shop) WebRTC VAD false-positives far more than Silero. The "aggressiveness" knob is coarse (four modes), and the harder modes false-negative on quiet speakers. Per-frame probabilities are not available, so you cannot threshold.

Use WebRTC VAD when: you cannot run a neural model (ultra-low-power edge), you already depend on WebRTC and can call into it cheaply, or you are reproducing a 2018-era baseline. Otherwise, Silero.

### Server-side VAD — OpenAI Realtime and Gemini Live

Both speech-native vendors run a server-side VAD and expose a small knob surface in session config.

**OpenAI Realtime** ([platform.openai.com/docs/api-reference/realtime](https://platform.openai.com/docs/api-reference/realtime)) — `turn_detection` object inside `session.update`:

```jsonc
"turn_detection": {
  "type": "server_vad",
  "threshold": 0.5,            // 0..1, speech-probability threshold
  "prefix_padding_ms": 300,    // include this much audio before the first speech frame
  "silence_duration_ms": 500,  // commit the turn after this much trailing silence
  "create_response": true,
  "interrupt_response": true
}
```

Alternatives on the same field: `"type": "none"` (you commit turns manually) and `"type": "semantic_vad"` (vendor-side linguistic turn-end model — Chapter 2 revisits this). Verify the current parameter surface at integration time; the schema evolves.

**Gemini Live** ([ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live)) — automatic VAD on the server by default; a `realtime_input_config.automatic_activity_detection` block exposes the equivalent knobs (`disabled`, `start_of_speech_sensitivity`, `end_of_speech_sensitivity`, `prefix_padding_ms`, `silence_duration_ms`). Field names and defaults shift across releases; cross-check at integration.

<!-- needs-research: verify the current Gemini Live automatic_activity_detection field names and defaults against the live API docs at author time -->

**What you give up by using server VAD.** You do not see the per-frame probability. You cannot stratify your false-trigger rate by background-noise class. Your only lever is the exposed knob surface — threshold, prefix padding, silence duration. For most deployments that is enough; for telephony with aggressive background noise or for high-floor-latency call-centre audio, running your own Silero-side VAD upstream of the vendor socket and sending audio only when you think it is speech is the right escape hatch.

## The three knobs that matter

Every production VAD has three tuning levers (names vary; semantics do not).

### Threshold

The speech-probability cutoff above which a frame is "speech." Silero's default is `0.5`. The failure modes it bounds:

- **Too low (e.g. 0.3).** VAD opens on background noise. STT ingest wastes compute / money. Barge-in triggers spuriously. In quiet rooms, fine; in telephony audio with hold-music leaks, disastrous.
- **Too high (e.g. 0.7).** VAD misses the leading phoneme of quiet-starting utterances ("mm-hm," "yes," softly-spoken names). First word clipped; STT returns `"? Hm. This is John."` instead of `"Yes, this is John."`

Tune per deployment: a consumer web app with clean USB-mic audio can live at 0.5; a call-centre deployment needs you to benchmark both false-trigger rate and missed-word rate against a corpus of your actual audio (Exercise 1). Do not ship the default without the number.

### Hangover / silence duration

How long of trailing silence before the VAD declares end-of-speech. Also called `silence_duration_ms` (OpenAI) / `min_silence_duration_ms` (Silero). Sets the boundary between "the user paused for a breath" and "the user is done."

Typical ranges:

- **200–350 ms** — snappy / interrupt-friendly. Appropriate for push-to-talk UX or consumer assistants that tolerate occasional user interruption.
- **400–700 ms** — the voice-agent default. The usability sweet spot for natural conversation. OpenAI Realtime's `silence_duration_ms: 500` is in this band.
- **800 ms–1.5 s** — slow / tolerant. Appropriate for elder-care UX, users operating in a second language, deployments where mid-utterance thought pauses are common.

Too short and the bot steps on the user mid-sentence (the classic IVR failure mode). Too long and the bot feels dead. Chapter 2 extends this: for truly natural conversation you want silence duration plus a semantic EOU signal, not silence alone.

### Minimum speech duration / prefix padding

Two related knobs that together decide what counts as a real utterance and how much audio you keep before it.

- **`min_speech_duration_ms`** (Silero) — speech blips shorter than this are discarded as spurious. Default 250 ms. Lowering it catches short confirmations ("yes", "no") at the cost of more false positives on coughs and keyboard clicks.
- **`prefix_padding_ms`** / `speech_pad_ms` — include this much audio before the first speech frame in the downstream buffer, so the first phoneme is not clipped. 200–300 ms is typical. Too small and the ASR gets `"...es, I agree"` instead of `"Yes, I agree"`.

The common bug: tuning threshold and silence-duration carefully and then leaving prefix-padding at a tiny default. The transcript begins mid-syllable and the LLM reasons on garbled text.

## Failure modes, named

VAD failures are the first-cause of at least a quarter of "the agent sounds broken" bug reports. Learn to spot them in a packet trace.

### False trigger (opens on non-speech)

Symptoms: bot interrupts the user to acknowledge silence; STT bill goes up; barge-in cancels in-flight TTS on noise bursts. Causes: threshold too low, background noise class the training data under-covered (music, TV talk, traffic), or an input gain structure where noise exceeds the energy floor the VAD was tuned against.

Mitigations: raise threshold; apply noise suppression upstream (Module 102 Chapter 5); classify ambient and refuse to start a turn below an SNR floor.

### Missed trigger (fails to open on speech)

Symptoms: user speaks, STT gets silence, agent sits silent or asks "sorry, can you repeat?" Causes: threshold too high; input gain too low; speaker is quiet or distant from the mic.

Mitigations: lower threshold; automatic gain control upstream; prompt the user to speak louder (rare, low-value) or route quiet speakers through a different acoustic profile.

### Clipped onset (opens too late)

Symptoms: STT drops the first phoneme or first word of an utterance. Causes: prefix padding too small; VAD latency itself (neural VADs have a few frames of inherent delay; energy VADs are zero-delay but louder false-triggers).

Mitigations: increase `speech_pad_ms` / `prefix_padding_ms` to 300 ms or more; keep a 500 ms rolling input buffer so you can backfill when VAD finally fires.

### Clipped offset / premature endpoint

Symptoms: agent talks over users who pause mid-thought ("I was thinking — [pause] — three things"). Causes: silence duration too short.

Mitigations: raise silence duration; **add a semantic EOU signal** (Chapter 2) so a mid-utterance pause doesn't fire the endpoint when the user is clearly not done ("so I was thinking — ").

### Hangover drift

Symptoms: on quiet-floor deployments the hangover feels too long; on noisy deployments the hangover feels too short. Causes: a fixed silence-duration is not the right abstraction — you really want a signal-energy-relative hangover.

Mitigations: estimate the noise floor on a rolling window (first 500 ms of the call, or during the greeting); make the hangover a function of (noise floor, SNR). OpenAI's Realtime server VAD is an energy + probability hybrid that approximates this internally; your Silero wrapper needs to add it.

### Non-speech vocalisations

Symptoms: user laughs, coughs, or sighs; VAD opens, STT returns an empty transcript or a random hallucination, LLM responds to nothing. Causes: Silero treats laughter and loud coughs as speech, because they fill the frequency bands the model learned.

Mitigations: run a non-speech classifier downstream of the VAD (even a tiny MFCC + logistic regression clipping against laughter / cough does well), or require that the ASR return at least one high-confidence word within 500 ms of the VAD fire to count the turn.

## Deployment patterns

### Pattern A — client-side VAD + STT streaming

VAD on the client (browser mic or mobile mic). Audio only flows upstream when VAD says speech. Server never sees silence frames.

- **Pros.** Minimum bandwidth. Minimum STT bill. Easiest to tune against the user's local acoustic environment.
- **Cons.** Client bugs (incorrect sample rate, JS thread stalls, Opus packet loss) show up as apparent VAD failures and are hard to debug. Different clients ship different VAD versions; your field distribution of VAD behaviour is wider than you'd like.
- **When to pick.** Consumer web and mobile voice agents. The bandwidth / cost win dominates the version-drift cost.

### Pattern B — server-side VAD on a continuous mic stream

Client streams raw audio continuously. Server runs the VAD. All version control, all threshold tuning, all telemetry, one place.

- **Pros.** One VAD version in production at a time. Tuning iterates server-side. Telemetry is uniform.
- **Cons.** You pay bandwidth to transport noise. On a telephony leg with 10 k concurrent calls the Opus-encoded silence frames add up.
- **When to pick.** Contact centres; any deployment where the audio is already going to the server unconditionally (telephony via SIP / WebRTC relay).

### Pattern C — vendor's server VAD inside Realtime / Live

You configure `turn_detection` and let the vendor decide. No client-side or server-side VAD on your part.

- **Pros.** Zero code. Server-side state (buffer committing, response creation) follows automatically.
- **Cons.** You cannot stratify false-trigger rates. Your only tuning lever is the exposed knob surface. If the vendor's VAD is too aggressive for your audio, your only escape is Pattern A / B with `turn_detection: "none"` and manual commits.
- **When to pick.** Prototypes and mid-scale consumer deployments on the speech-native stacks where the default is good enough.

### Pattern D — two-stage: client VAD gates ingest, server VAD commits turns

A client-side Silero decides whether to stream at all; the server-side vendor VAD (Realtime, Gemini Live) decides when the turn ends.

- **Pros.** Bandwidth / cost control from Pattern A + vendor state machine from Pattern C.
- **Cons.** Two VADs can disagree. The client thinks the user is done; the server is still collecting buffer; the response trigger is late.
- **When to pick.** Telephony voice agents on a speech-native stack where you must cost-control the socket but cannot abandon vendor server-side turn detection.

Name which pattern you ship. Each pattern has a different telemetry surface and a different test harness.

## A reference configuration to deviate from

For a greenfield voice agent on English consumer audio, start here and tune against your corpus:

- **Stack.** Silero VAD client-side (Pattern A) + manual `input_audio_buffer.commit` to OpenAI Realtime (`turn_detection: null`).
- **Threshold.** `0.5` (Silero default).
- **Silence duration.** `500 ms`.
- **Minimum speech.** `250 ms`.
- **Prefix padding.** `300 ms`.
- **Chunk size.** 30 ms frames (Silero's design) aggregated to 100 ms payloads over the socket to keep event rate sane.

Call it the baseline. Measure against the metric suite in Chapter 5. Then decide what to change.

## Common anti-patterns

- **Shipping the default threshold without measuring.** `0.5` is a sensible prior, not a tuned value. The tune is cheap (Exercise 1); the untuned bill is paid in every user interaction.
- **Using WebRTC VAD on 2025-era telephony audio.** Save the engineering cost of fighting its false-trigger rate — go to Silero or vendor VAD.
- **Treating VAD as end-of-utterance.** It isn't. Natural speakers pause mid-utterance; silence-duration alone will cut them off. Chapter 2.
- **No hangover relative to noise floor.** A fixed `silence_duration_ms` is too long in a quiet room and too short in a noisy one. Estimate noise floor on a rolling window.
- **Forgetting to pad the onset.** Threshold and silence duration are the glamorous knobs. `speech_pad_ms` quietly protects the first phoneme.
- **Running both a client-side VAD and a server-side VAD without naming which one is authoritative.** Pattern D needs explicit coordination; otherwise the two VADs will fight.
- **No non-speech-vocalisation classifier.** Laughter, coughs, throat-clears, and sighs fool neural VADs more than noise does.

## Summary

- VAD gates STT ingest and triggers barge-in. In pipeline deployments it lives at the edge; in speech-native deployments the vendor runs one server-side and exposes a knob surface.
- Silero VAD is the open-weights default (MIT, <2 M params, ~1 ms / frame CPU). WebRTC VAD is the historical compare-point; prefer Silero in new work. OpenAI Realtime and Gemini Live expose `turn_detection` fields that the engineer must tune explicitly.
- The three knobs that matter: **threshold** (0.5 default, trade false-trigger vs. missed-trigger), **silence duration / hangover** (500 ms voice-agent default, trade snappiness vs. premature endpoint), **minimum-speech + prefix padding** (250 / 300 ms defaults, guard against lost short utterances and clipped onsets).
- The failure modes: false trigger, missed trigger, clipped onset, premature endpoint, hangover drift, non-speech vocalisations. Each maps to a specific knob or a specific upstream fix.
- Pick an explicit deployment pattern (client-only, server-only, vendor-only, two-stage). Each has a distinct telemetry surface.
- VAD is not EOU. Silence is only a weak signal for "the user is done." Chapter 2 adds the semantic and model-based signals.

Next chapter: end-of-utterance detection beyond VAD — the LiveKit open-weights turn-detector, semantic EOU with a small LLM classifier on the running transcript, and the Voice Activity Projection family from Ekstedt and Skantze.
