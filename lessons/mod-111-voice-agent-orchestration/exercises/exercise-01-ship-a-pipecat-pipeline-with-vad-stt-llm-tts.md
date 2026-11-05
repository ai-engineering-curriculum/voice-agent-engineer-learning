# exercise-01: Ship a Pipecat Pipeline with VAD + STT + LLM + TTS

**Estimated effort:** 3 hours · **Output:** a working Pipecat pipeline (local + WebRTC transport) and a short measurements note

## Objective

Stand up a cascaded voice agent on [Pipecat](https://docs.pipecat.ai/) end to end: VAD, STT, context aggregator, LLM, streaming TTS, and transport. Prove it works on a local microphone/speaker loop, then repeat on a browser-based WebRTC transport (Daily or LiveKit — your call). Measure the per-leg latency with the same spans you set up in mod-109. This is the baseline pipeline you will compare against in exercises 02 and 03.

## Prerequisites

- [Chapter 1 — Pipecat: Processor Pipelines, Transports, and First-Class Integrations](../01-pipecat-pipeline-and-processors.md).
- Working knowledge of mod-102 (speech pipeline) and mod-109 Chapter 2 (per-leg latency attribution).
- API keys for your chosen STT (Deepgram recommended; AssemblyAI / Azure / local Whisper also fine), LLM (OpenAI / Anthropic / Gemini), and TTS (Cartesia or ElevenLabs recommended for streaming). Use low-tier models to keep cost bounded.
- Python 3.11+ and a working audio device, or a Daily / LiveKit developer account.

## Steps

1. **Install Pipecat and the service plugins you need.** Follow the current install guide in the [Pipecat docs](https://docs.pipecat.ai/). Install only the service extras you intend to use (`pip install "pipecat-ai[deepgram,openai,cartesia,silero,daily]"` style).

2. **Build the pipeline against the Local transport.** A minimal pipeline:
   - `LocalAudioTransport` (microphone in, speaker out) with `SileroVADAnalyzer` enabled.
   - `DeepgramSTTService` (or your chosen STT).
   - `OpenAILLMContext` + user-side `OpenAILLMContextAggregator`.
   - `OpenAILLMService` (or your chosen LLM).
   - `CartesiaTTSService` (or your chosen streaming TTS).
   - Assistant-side `OpenAILLMContextAggregator`.
   - Transport output.

   Verify end-to-end conversation on your laptop. The agent should greet, respond, and correctly handle "stop talking" barge-in.

3. **Swap in a WebRTC transport.** Replace `LocalAudioTransport` with `DailyTransport` (if you have a Daily account) or `LiveKitTransport` (if you have a LiveKit setup). You should change only the transport configuration; the rest of the pipeline is unchanged. This is the point of the transport abstraction — prove it.

4. **Instrument per-leg latency.** Using OpenTelemetry (or Pipecat's built-in observability if available at your date), add spans around each processor. At minimum capture: `transport_input → stt_first_partial`, `stt_final → llm_first_token`, `llm_first_token → tts_first_audio`, `tts_first_audio → transport_output_first_frame`. Use the [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) for the LLM span attributes.

5. **Run a measurement pass.** On 20 scripted user turns, record p50 and p95 latency per leg for both the Local and the WebRTC transports. Tabulate.

6. **Verify interrupts.** Write a short test script (or do it manually) where the user starts speaking while the bot is mid-sentence. Confirm the TTS stops within ~500 ms (mod-106 Chapter 6) and the next assistant turn does not continue the interrupted sentence. Note any drift from the state machine in Chapter 5.

7. **Write a short measurements note.** One page: the final pipeline listing, per-leg latency numbers for both transports, observed interrupt behaviour, and anything that surprised you. Date everything.

## Starter guidance

- **Use streaming TTS.** The whole point of the pipeline is that TTS starts on the first LLM token. If you pick a non-streaming TTS, you've silently defeated the pipeline model. Cartesia, ElevenLabs, and Deepgram Aura all support streaming at the time of writing — confirm the current list.
- **Keep the system prompt short and voice-first.** "Respond in one or two sentences unless the user asks for detail. Do not use markdown, lists, or special formatting." The pipeline can be perfect and still sound terrible if the LLM emits markdown.
- **Prefer Deepgram Nova or similar low-latency STT for the first pass.** You're learning the framework; don't fight the ASR today.
- **If you pick a local Whisper for STT**, it will have higher latency than a streaming hosted STT and will skew your measurements. Note this in the write-up.
- **Daily is the lowest-friction WebRTC transport if you don't already have a LiveKit footprint.** It's by the same maintainer as Pipecat and has the best out-of-box integration.

## Acceptance criteria

- [ ] A Pipecat pipeline that handles a full conversational loop on a Local transport, with VAD, streaming STT, context aggregation, LLM, streaming TTS, and voice-first output (no markdown).
- [ ] The same pipeline running on a WebRTC transport (Daily or LiveKit) with no changes outside the transport configuration.
- [ ] Per-leg latency spans instrumented and emitted (OpenTelemetry or Pipecat observability).
- [ ] A measurement table with p50 and p95 per leg for 20 scripted turns on both transports.
- [ ] Verified interrupt behaviour (TTS stops within ~500 ms of user barge-in; next turn does not continue the interrupted sentence).
- [ ] A one-page measurement note, dated.

## Stretch goals

- **Realtime variant.** Swap the STT + LLM + TTS trio for a single `OpenAIRealtimeBetaLLMService` (or Gemini Live equivalent). Re-run the measurements. Compare end-to-end p95.
- **Twilio transport.** Add a Twilio transport and dial the agent from a phone. The conversation should still work; expect worse audio quality (G.711).
- **End-of-utterance tuning.** Replace plain VAD with a Pipecat end-of-utterance processor (SmartTurn-style; check current availability). Measure whether the turn-start latency changes meaningfully.
- **Add a logging processor.** Insert a custom processor that logs every frame type and timestamp. This becomes your in-pipeline debug tool — keep it off by default but easy to toggle.

## Keep it

This pipeline is the baseline for the rest of the module. Exercise-02 implements the same shape on LiveKit Agents; exercise-03 implements it on Vocode; exercise-04 compares all three against managed platforms. Keep the measurement note alongside each subsequent exercise so you have a single evolving comparison.
