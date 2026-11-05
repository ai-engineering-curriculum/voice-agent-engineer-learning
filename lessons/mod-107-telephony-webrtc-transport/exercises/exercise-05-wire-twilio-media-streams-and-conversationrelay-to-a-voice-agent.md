# exercise-05: Wire Twilio Media Streams and ConversationRelay to a Voice Agent

**Estimated effort:** 3 hours · **Output:** two working inbound-call paths (Media Streams + ConversationRelay), side-by-side latency measurements, and a one-page comparison

## Objective

Build **two** PSTN ingress paths for the same voice agent — one via [Twilio Media Streams](https://www.twilio.com/docs/voice/media-streams) and one via [Twilio ConversationRelay](https://www.twilio.com/docs/voice/conversationrelay) — call both from a real phone, and measure the practical differences you discussed in [Chapter 5](../05-pstn-providers.md). You will finish this exercise able to defend a Twilio-vs-provider-vs-trunk decision with numbers, not opinions.

## Prerequisites

- [Chapter 5 — PSTN Ingress / Egress Providers](../05-pstn-providers.md); [Chapter 2 — SDP](../02-sdp-offer-answer.md) for codec context.
- A Twilio account with a provisioned E.164 number (trial accounts work; you'll need to add ~$5 credit for toll-free verification bypass or use a local 10DLC number).
- A publicly reachable HTTPS endpoint. [ngrok](https://ngrok.com/) or [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) is fine.
- Python 3.11+ with `fastapi`, `uvicorn`, `websockets`, `twilio`, and an STT/TTS stack from mod-103/104 (Deepgram or Whisper for STT, ElevenLabs or Azure TTS for TTS, plus any LLM with a Chat Completions-like API).
- A regular phone (mobile or landline) to dial the Twilio number.

## Steps

### Part A — Baseline: Media Streams round-trip (75 minutes)

1. Create a FastAPI server with two endpoints:
   - `POST /voice` — returns TwiML that connects the call to a Media Streams WebSocket:
     ```xml
     <Response>
       <Connect>
         <Stream url="wss://<your-ngrok-url>/stream" />
       </Connect>
     </Response>
     ```
   - `WebSocket /stream` — accepts the bidirectional µ-law 8 kHz PCM stream.

2. In the WebSocket handler, implement the full agent loop:
   - Decode inbound µ-law → PCM16 @ 8 kHz → upsample to 16 kHz.
   - Stream PCM16 to a cloud STT via its WebSocket.
   - On each final transcript, call your LLM.
   - Stream LLM text to a TTS that emits PCM at 24 kHz.
   - Downsample TTS to 8 kHz, encode as µ-law, send back over the WebSocket.

3. Configure the Twilio number to POST to `/voice` on an incoming call.

4. Dial the number. Have a 60-second scripted conversation ("Hi, I'd like to check my balance" → "Your balance is $X" × 3 turns).

5. Instrument timestamps on each hop:
   - `t_stt_first_partial` — first partial transcript back from STT.
   - `t_stt_final` — final transcript on end-of-turn.
   - `t_llm_first_token` — first token back from the LLM.
   - `t_tts_first_byte` — first PCM frame back from TTS.
   - `t_first_ws_send` — first encoded µ-law frame written back to the Twilio WebSocket.

6. Record the **end-to-end turn-taking latency** (user-stops-speaking → first-audio-back) over 5 turns. Report p50 and p95.

### Part B — ConversationRelay path (60 minutes)

7. Create a parallel endpoint that returns ConversationRelay TwiML instead:
   ```xml
   <Response>
     <Connect>
       <ConversationRelay url="wss://<your-ngrok-url>/relay" welcomeGreeting="Hi, how can I help?" />
     </Connect>
   </Response>
   ```
   Follow the [ConversationRelay docs](https://www.twilio.com/docs/voice/conversationrelay) for the WebSocket event schema — you'll receive JSON events (`setup`, `prompt`, `interrupt`, `dtmf`, `error`) and send JSON directives (`text`, `interrupt`, `end`).

8. In the `/relay` WebSocket handler, on each `prompt` event (final user utterance text), call your LLM and send `text` directives back with the token stream. ConversationRelay handles STT and TTS.

9. Point a *second* Twilio number at the ConversationRelay endpoint.

10. Dial the ConversationRelay number. Same scripted conversation.

11. Record the same five timestamps (where applicable — some, like `t_stt_first_partial`, are hidden inside Twilio). Record end-to-end turn-taking latency over 5 turns.

### Part C — Comparison (45 minutes)

12. Produce a side-by-side table:

    | Metric | Media Streams | ConversationRelay | Winner |
    |---|---|---|---|
    | Codec on the wire | µ-law 8 kHz | (check — likely Opus internally) | |
    | STT control | You pick provider | Twilio picks | |
    | TTS control | You pick provider | Twilio picks | |
    | p50 turn-taking latency | X ms | Y ms | |
    | p95 turn-taking latency | X ms | Y ms | |
    | LoC to ship | ~N | ~M | |
    | Per-minute cost | Voice minutes + ElevenLabs + Deepgram + LLM | ConversationRelay minute rate + LLM | |
    | Vendor-lock-in surface | Low | High | |

13. Write a one-page recommendation for a hypothetical product with these constraints:
    - Launching in 3 weeks. Team of 2.
    - 10k call-minutes / month initial, 100k target by end of year.
    - US only for now, EU in 6 months.
    - Legal requires customer-managed TTS voice cloning (ElevenLabs Pro).

    Argue which path you'd pick. Changes at 100k minutes? EU launch? The legal constraint?

## Starter guidance

- The Media Streams audio is **mono µ-law 8 kHz** in base64-encoded 20 ms frames (160 bytes decoded) inside JSON. [Format docs](https://www.twilio.com/docs/voice/media-streams/websocket-messages).
- Upsampling 8 kHz → 16 kHz: use `torchaudio.functional.resample` or `soxr`. Don't `::2`-index; low-pass filtering is mandatory (see mod-102 Chapter 4).
- For STT with Deepgram, send PCM16 @ 16 kHz as a binary WebSocket frame; the API will stream partials back.
- For TTS with ElevenLabs streaming, call `/v1/text-to-speech/<voice_id>/stream` with `output_format=pcm_16000` (or use their realtime API if available at the time). Then resample to 8 kHz µ-law.
- ConversationRelay abstracts the above — `prompt` events are already-transcribed user text; `text` directives are LLM output tokens that Twilio synthesises.
- Do **not** skimp on timestamp logging; the whole point is to measure, not to guess.

## Acceptance criteria

- [ ] Media Streams path handling a full round-trip call; 5 turns logged with 5 timestamps each; p50/p95 reported.
- [ ] ConversationRelay path handling a full round-trip call; 5 turns logged; p50/p95 reported.
- [ ] Side-by-side table (12) filled completely, with sources for each cell.
- [ ] One-page recommendation for the hypothetical product with explicit trigger points for revisiting the decision.
- [ ] A short "which is harder to debug when something breaks?" reflection — Media Streams exposes more surface area, ConversationRelay hides it; which is worse when a customer complains?

## Stretch goals

- **Add STIR/SHAKEN attestation check.** From the Twilio call resource, read the inbound call's attestation (A/B/C). Log it per call. For outbound calls, verify Twilio is sending your attestation level correctly.
- **Enable call recording** on both paths. Add consent gating per Chapter 6. Note the differences in how each path exposes recording APIs.
- **Add a barge-in test.** During TTS playback, speak over the bot. Media Streams: your in-process VAD must cancel the outbound audio. ConversationRelay: Twilio handles this via the `interrupt` event. Measure barge-in reaction time (user-starts-speaking → bot-audio-stops) for each.
- **Add Voice Insights.** Open Twilio's Voice Insights dashboard for a completed call. Match the MOS, jitter, and loss figures to the Twilio-side of your call. Compare to the Chapter-7 client-side stats you'd measure on a WebRTC equivalent.
- **Compare against Vonage or Plivo.** Repeat Part A with Vonage Voice API WebSockets or Plivo's equivalent. Does the audio format differ? Does the latency?

## Keep it

The timestamp-logging harness is reusable: wire it into mod-109 (latency budget) and mod-112 (observability). The per-turn p50/p95 table is the numeric artifact you hand a product manager asking "is the agent fast enough?"
