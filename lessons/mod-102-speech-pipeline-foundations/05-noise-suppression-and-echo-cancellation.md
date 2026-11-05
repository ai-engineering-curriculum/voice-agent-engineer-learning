# 05. Noise Suppression and Echo Cancellation

## Why this chapter

Three audio-cleanup problems will show up in every production voice stack:

1. **Acoustic echo** — your agent's TTS comes out the user's speaker, gets picked up by the user's microphone, and lands back in your ASR. The agent transcribes itself and gets confused.
2. **Background noise** — fans, HVAC, traffic, keyboard clicks, a baby in the next room. Reduces ASR accuracy; makes VAD triggers unreliable.
3. **Crosstalk / background voices** — someone else in the room speaks; the ASR picks it up and treats it as the user.

Three families of solutions exist for these: **acoustic echo cancellation (AEC)** for problem 1, **noise suppression (NS)** for problem 2, and **voice-focus / target-speaker extraction** for problem 3. This chapter covers the three implementations you'll actually meet — WebRTC AEC3, RNNoise, Krisp — what each does and doesn't fix, and where to place them in the pipeline.

The placement question matters as much as the choice of library. Run AEC in the wrong place and it has no reference signal to cancel against. Run noise suppression too aggressively on the server when the client already did it and you get artefacts. Run voice-focus on crosstalk when the real problem was an open mic.

## Acoustic echo: the physics

```
   ┌──────────────┐          ┌──────────────┐
   │ user's mic   │ ──────── │ your pipeline│
   │              │          │              │
   │ user's spkr  │ ←─────── │  agent TTS   │
   └──────┬───────┘          └──────────────┘
          │  acoustic coupling
          ▼
   user's mic also picks up
   the speaker signal
```

The echoed signal is the TTS output, played out the speaker, round-tripped through the room's acoustics (reverb, delay, non-linear speaker colouration), and recaptured by the mic. In a browser or a native mobile client, this loop is **~50–300 ms long** depending on OS audio stack and hardware.

Why can't you just high-pass filter it out? Because the echoed audio *is* speech-band energy, indistinguishable by frequency alone from the user's own voice. The standard trick is **adaptive linear echo cancellation**: subtract an estimate of the reference (far-end / TTS) signal from the microphone signal, where the estimate is produced by an adaptive filter that models the room's impulse response.

AEC needs **two inputs**:

- **The microphone signal** — what the user's mic captured.
- **The reference signal** — what was played out the speaker.

If you cannot provide the reference signal, you cannot cancel echo. This is the single most important deployment rule in this chapter.

## WebRTC AEC3 — the production default

WebRTC's AEC3 is the third-generation acoustic echo canceller in the WebRTC audio-processing-module (APM), developed by Google. It is open-source (BSD), battle-tested in Chrome and Google Meet, and bundled with every WebRTC implementation.

Properties you'll rely on:

- **Linear + non-linear residual suppression.** Handles both the straightforward echo and the harder non-linear coupling caused by speaker distortion or moving the mic.
- **Delay-adaptive.** Figures out the mic–speaker delay dynamically; doesn't need a user-tuned constant.
- **Double-talk aware.** When the user speaks *while* the TTS is playing, AEC3 suppresses the TTS portion and lets the user's voice through — not perfectly, but the state of the art for open-source.

Where it runs in a WebRTC-native stack:

- **Client-side, in the browser or native WebRTC SDK.** Chrome, Firefox, iOS WebKit, Android WebView all run WebRTC APM (including AEC3) on the mic capture path *by default* when you call `getUserMedia({audio: true})`.
- You can toggle it explicitly: `getUserMedia({audio: {echoCancellation: true}})` (default) or `false` (off, if you know what you're doing).
- Server-side AEC (inside your media server / SFU) is usually not needed because the client already did it.

Where WebRTC AEC3 does *not* run, and you have to do something else:

- **PSTN legs.** The user is on a landline or cellphone; echo cancellation is the *carrier's* responsibility. You usually have no reference signal to cancel against and no place to inject one. Most telephony providers run echo cancellation on their edge; if the user still complains of echo, escalate to the carrier, not into your pipeline.
- **Server-side synthetic agents in SIP/RTP flows.** Your media server is both receiving user audio (from carrier) and sending TTS to it. In principle you could AEC the user-inbound stream against the TTS-outbound stream — but with typical PSTN round-trip delays (~100–200 ms plus jitter) and non-linearities in the carrier path, AEC is usually a net loss. Trust the carrier.
- **Native mobile apps using a custom audio capture pipeline.** If you used `AVAudioEngine` or `AudioRecord` instead of a WebRTC SDK, you have to invoke the OS's echo cancellation explicitly (`AVAudioSession` voice-processing mode on iOS, `AudioEffect` AEC on Android).

> *Call-of-the-wild rule.* Before shipping any custom audio capture on mobile, verify the platform's built-in AEC is in the path. "We'll fix echo later" ends with the user hearing themselves over a 300 ms delay on every call.

## RNNoise — noise suppression, open-source

[RNNoise](https://jmvalin.ca/demo/rnnoise/) is a small recurrent-neural-network noise suppressor by Jean-Marc Valin (also the author of Opus). It operates on 20 ms frames of 48 kHz speech, outputs per-frame band gains, and runs in a few tens of microseconds per frame on a single CPU core.

Properties:

- **Open-source** (BSD). Compiles to a ~100 KB library.
- **Realtime by construction.** No batch processing; one frame in, one frame out.
- **Reasonable quality** against steady-state noise (HVAC, fan, hum). Less impressive against non-stationary noise (keyboard, dog barking, siren).
- **Does not cancel echo.** Treat RNNoise and AEC as orthogonal. A typical pipeline runs AEC *before* NS.

Where it runs:

- **Client-side**, if you have a WebAssembly build and bandwidth to add it to your JS bundle.
- **Server-side**, as a pre-ASR cleanup pass. Common in production deployments where client-side suppression can't be guaranteed (e.g., SIP ingress — the user's phone is not going to run RNNoise).
- **Inside a media server**, as a per-track processor (both LiveKit and Pipecat pipelines have patterns for this).

Where RNNoise falls short:

- **Background voices.** NS treats non-stationary voice as signal, not noise. If someone else is talking in the room, RNNoise lets them through.
- **Heavy over-suppression.** When misconfigured or driven hard, NS wipes out quiet consonants along with the noise. Diagnose with the log-mel spectrogram (Chapter 2) — if voiced regions look thin and smeared, back off.

## Krisp SDK — the commercial option

Krisp (<https://krisp.ai/>) is a commercial noise suppression and voice-isolation SDK used by Zoom, Discord, and various contact-centre platforms. It's not open source; you license the SDK.

What you get beyond RNNoise:

- **Better quality on non-stationary noise** (keyboard, music, sirens, baby). This is Krisp's core differentiation — it's trained on a much larger corpus of noise types than RNNoise.
- **Voice isolation / target-speaker extraction.** Suppresses other human voices in the room, not just non-voice noise. This is the one problem RNNoise does not address.
- **Background voice cancellation** marketed as a distinct feature; useful in open-office and shared-household scenarios.

When to pay for Krisp instead of using RNNoise:

- **Your user base calls from loud or shared environments** (call-centre agents, retail, warehouse).
- **Background-voice crosstalk is actually biting your ASR** — i.e., you've measured WER degradation that correlates with background speech.
- **You can place the SDK where you need it.** Krisp provides client SDKs for desktop and mobile, as well as server-side integrations. Some contact-centre vendors (e.g., Twilio's Noise Cancellation feature powered by Krisp) run it as a provider-edge option, so you may be able to enable it without integrating the SDK yourself. <!-- needs-research: verify current Twilio Krisp integration branding and surface at author time -->

Krisp is not a replacement for AEC; run AEC first.

## Placement decision tree

Three decisions, in order:

### 1. Who provides the reference signal for AEC?

- **WebRTC client with `echoCancellation: true`.** Done. Client runs AEC3 itself; your server never sees echo.
- **Native mobile client with voice-processing mode on.** Done.
- **PSTN leg.** The carrier handles echo. Don't add AEC server-side; if you need to, escalate to the carrier.
- **Custom pipeline where you control both mic capture and speaker playback.** You can invoke an AEC library server-side — the Speex DSP library, a port of WebRTC APM, or a commercial component — using the TTS you sent as the reference. Harder than it sounds in practice because of delay compensation.

### 2. Where does noise suppression live?

- **Client-side if possible.** Lowest-cost place to filter — the server never sees the noise, so you save bandwidth and don't ASR against it.
- **Server-side when the client can't or won't.** PSTN ingress, legacy mobile clients, browsers with suppression disabled. Run NS on the inbound audio before ASR.
- **Provider-edge** (Twilio Krisp, LiveKit Krisp plugin, etc.) when the provider already has the integration. This is usually the cheapest place to ship a quality bump.

### 3. Do you have a background-voice problem?

- **No measurable WER delta from background speech.** Don't buy Krisp yet.
- **Yes, measurable degradation.** Add voice isolation. Prefer provider-edge or client-side; server-side as last resort.

## Cross-cutting failure modes

Three bugs this chapter will spare you:

### Failure mode 1 — AEC with no reference

You enabled AEC server-side but forgot to pipe the TTS output to it as the reference. AEC runs, finds nothing to cancel against, passes the microphone through untouched — but costs 10–30 ms of latency for no gain. Trace the reference-signal wiring; verify the AEC actually attenuates a known test tone played through the TTS path.

### Failure mode 2 — Double suppression

Client runs RNNoise; server runs RNNoise again. Voice sounds "underwater" or watery. Measure the log-mel spectrogram — if voiced regions have characteristic smeared harmonics, suppression is stacking. Disable the server pass or the client pass (keep the one that gets the better quality on your corpus; usually client-side when available).

### Failure mode 3 — Echo cancellation suppresses the user

AEC's double-talk detection is imperfect. Under aggressive settings, during user barge-in the AEC continues to suppress what it thinks is TTS echo, which includes the user's voice. The agent stops hearing the user. This is the single most common "barge-in doesn't work" root cause when AEC is live. Mitigations: back off the AEC aggressiveness ("ECHO_SUPPRESSION_LEVEL" in WebRTC APM); ensure the TTS playback stops the moment VAD detects user speech so the reference signal goes silent.

## A minimum pipeline diagram

A production browser-to-agent voice loop with cleanup in the right places:

```
browser                         server                             ASR
───────                         ──────                             ───
getUserMedia  ← AEC3 (client)   WebRTC inbound stream              transcript
     │        ← NS   (client)         │
     │                                ▼
     └── Opus @ 48 kHz ────►   decode Opus →  RNNoise or Krisp →  ASR @ 16 kHz
                                   (only if client NS wasn't guaranteed)

LLM + TTS                        server                             browser
──────────                       ──────                             ───────
TTS @ 24 kHz ──► mixer / encoder ──► Opus @ 48 kHz ──► WebRTC outbound ──► speaker
```

For a PSTN variant, the first segment becomes:

```
PSTN
────
user mic → carrier AEC → Twilio/Vonage → µ-law @ 8 kHz → your server
                                                            │
                                                            ▼
                                                   decode µ-law → upsample → RNNoise? → ASR
```

The "RNNoise?" is intentional: many carriers already clean the signal well enough that an added NS pass does more harm (over-suppression, extra CPU, extra latency) than good. Test before shipping.

## Summary

- Echo and noise are separate problems; cancel echo with AEC (needs a reference signal), suppress noise with NS.
- WebRTC AEC3 is the production default for client-side echo cancellation; RNNoise and Krisp are the production defaults for noise suppression.
- Place cleanup as close to the capture point as possible — client-side > provider-edge > server-side.
- Double suppression, missing reference signals, and over-aggressive AEC-during-barge-in are the three recurring failure modes.

Next chapter: neural audio codecs — EnCodec, SoundStream, Mimi — and why 12.5 Hz tokenisation matters for the next generation of speech-to-speech models.
