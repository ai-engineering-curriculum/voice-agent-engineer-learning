# 01. The OpenAI Realtime API: Session Lifecycle, Transports, and the Voice Loop

## Why this chapter

Modules 103 and 104 wired the two halves of the classical pipeline — STT and TTS — as independent streaming services bridged by an LLM. This chapter turns the first page on the alternative: a single model that accepts audio in, reasons, calls tools, and emits audio out, over one persistent socket. OpenAI's Realtime API (the family around `gpt-4o-realtime` and `gpt-4o-mini-realtime`, and the ChatGPT Advanced Voice product it powers) is the industry's most-deployed speech-native stack and the most useful one to learn first — not because it is the right answer for every deployment, but because its session-lifecycle grammar (begin → append audio → commit → model responds → interrupt → end) is a template that recurs in Gemini Live and in every realtime-relay stack you'll touch for the rest of the module.

This chapter teaches you the contract: the transports (WebSocket vs. WebRTC), the session configuration knobs that decide how the server detects turn boundaries, the event vocabulary you send and receive, the function-calling path that lets the model hit your tools inside the voice loop, the interrupt semantics, and the prompt-caching shape that keeps latency and cost sane across long sessions. It does not teach you the model's personality or voice selection in depth — those shift with every API release and belong in a vendor doc, not a curriculum.

## The mental model

The Realtime API is a **stateful conversation socket**. A single connection carries:

- Audio from the user (streamed in as PCM or μ-law chunks).
- Text or audio from the model (streamed out as events — partial transcripts, audio deltas, tool calls, completion markers).
- Session-level configuration (voice, instructions, VAD mode, tool schemas, temperature, modality).

That socket is the session. Everything you do — append input, commit a turn, cancel a response, ingest a tool result — is an event you send on it, or an event you receive. You never POST a fresh request; you push events into a long-lived connection and read events back.

This is the first thing to internalise, because every bug in your first Realtime integration is a mis-modelling of state. The client is not request/response; it is a two-sided event loop bound to a `session` object.

## The two transports

OpenAI exposes the Realtime API over two transports. Pick on the ingress shape, not the preference of your framework.

### WebSocket (server-to-server)

- **URL:** `wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-<date>`. Verify the current model tag at [platform.openai.com/docs/models](https://platform.openai.com/docs/models) at integration time; the `-preview` and dated suffixes rotate.
- **Auth:** `Authorization: Bearer <OPENAI_API_KEY>` plus the required `OpenAI-Beta: realtime=v1` header.
- **Who speaks the socket:** your backend. You proxy audio from whatever ingress you have (Twilio Media Streams, LiveKit SFU server-side, a mobile client via your own WS, etc.) and forward events.
- **Audio framing:** base64-encoded PCM16 or G.711 μ-law in the `input_audio_buffer.append` event payload. You chunk at ~20–100 ms of audio per event (smaller is lower latency, larger is less event overhead).
- **When to pick it:** telephony (Twilio, SignalWire, Vonage), any case where a browser or mobile client cannot hold your API key, batch / server-driven testing, or any case where you want a single service accounting for the whole socket.

### WebRTC (client-to-server)

- **URL:** an SDP offer POSTed to `https://api.openai.com/v1/realtime` with an ephemeral token minted server-side (see below), yielding a direct peer connection between the client and OpenAI's media server.
- **Auth:** an **ephemeral client secret** issued by `POST /v1/realtime/sessions` on your backend. Never put your long-lived key in the browser.
- **Who speaks the socket:** the user's browser or mobile app, plus a data channel for control events.
- **Audio framing:** Opus frames over SRTP, the native WebRTC stack. You get jitter buffering, FEC, and congestion control for free; you do not touch PCM in the client.
- **When to pick it:** in-browser web apps, native mobile apps where the handset holds the microphone, demos, anything latency-sensitive where you want RTP's media handling rather than rolling your own.

**Rule of thumb.** Telephony ingress (PSTN → Twilio Media Streams) is a WebSocket deployment. A browser or mobile voice app is a WebRTC deployment. A LiveKit Agents or Pipecat pipeline usually runs the Realtime leg as WebSocket because the SFU already owns the client-side RTP; see [Module 107](../mod-107-telephony-webrtc-transport/README.md) for the full transport decision tree.

## The session lifecycle

The grammar is the same on both transports — only the enclosing pipe changes. Walk it once; you will walk it the same way in every other realtime stack you touch.

### 1. Begin the session

**WebSocket.** Open the socket. Immediately send a `session.update` with the configuration you want. The server will echo a `session.updated` event with the resolved config. Until you do this, the model has defaults (`alloy` voice, `verse`-style instructions, server VAD, no tools).

**WebRTC.** First, your backend calls `POST /v1/realtime/sessions` with the desired session config and receives an ephemeral `client_secret`. The browser uses that secret to POST its SDP offer; the server returns an SDP answer. Once ICE completes you have an audio track in each direction and a data channel for events. You still send `session.update` on the data channel for mid-session changes.

Session config fields the chapter cares about:

```jsonc
{
  "type": "session.update",
  "session": {
    "modalities": ["text", "audio"],         // emit both; "text" only is useful for testing
    "voice": "verse",                        // alloy, ash, ballad, coral, echo, sage, shimmer, verse — verify current list
    "input_audio_format": "pcm16",           // or g711_ulaw, g711_alaw
    "output_audio_format": "pcm16",          // or g711_ulaw for telephony egress
    "input_audio_transcription": {           // enable to get user-side transcripts
      "model": "whisper-1"                   // Whisper runs server-side on the input audio
    },
    "turn_detection": {                      // see §Turn detection below
      "type": "server_vad",
      "threshold": 0.5,
      "prefix_padding_ms": 300,
      "silence_duration_ms": 500,
      "create_response": true
    },
    "tools": [ /* function schemas */ ],
    "tool_choice": "auto",
    "temperature": 0.8,
    "max_response_output_tokens": "inf",
    "instructions": "You are a..."
  }
}
```

Values move with API versions; always cross-check against the current [Realtime API reference](https://platform.openai.com/docs/api-reference/realtime).

### 2. Append user audio

```jsonc
{ "type": "input_audio_buffer.append", "audio": "<base64 PCM16 chunk>" }
```

Send one of these per audio chunk as it arrives from the microphone / telephony leg. Chunk size is a comfort choice: 20 ms is the WebRTC convention, 100 ms keeps the event rate low. The server appends into an input buffer bound to the current session turn.

### 3. Commit or let VAD commit

A **turn** ends when the server flips from listening to responding. That transition can happen two ways:

- **Server VAD (default).** You keep streaming `input_audio_buffer.append` events; the server-side voice-activity detector watches the energy envelope, declares end-of-speech after `silence_duration_ms` of quiet past a `threshold`, automatically commits the buffer, and (if `create_response: true`) begins generating a response. You do nothing. This is the voice-agent-default mode.
- **Manual commit.** You set `turn_detection: null` in `session.update`. Now you own the turn boundary: send `input_audio_buffer.commit` when you decide the user has finished (e.g. you have your own VAD upstream, or a push-to-talk UX), then `response.create` to ask the model to respond. This is the mode you want for push-to-talk UX, for telephony integrations where an upstream VAD already decided, and for deterministic test harnesses.

Everything downstream cares about which mode you're in — especially Module 106's barge-in design. Pick explicitly; the default is not always right.

### 4. The model responds

Once the turn has been committed and `response.create` has fired (either implicitly via server VAD or explicitly via your event), the server streams a sequence of events for the response:

- `response.created` — the response has an id; subsequent deltas tag against it.
- `response.output_item.added` — one item per output element (a message, a function call).
- `response.audio.delta` — base64 PCM16 (or μ-law) chunks of the model's spoken audio. Pipe these directly into your audio sink; they arrive as fast as the model can synthesise, usually faster than realtime.
- `response.audio_transcript.delta` — the model's own speech transcribed as it speaks. Use for logging, captions, and alignment.
- `response.function_call_arguments.delta` and `.done` — the model is calling a tool; see §Function calling below.
- `response.output_item.done` — one item completed.
- `response.done` — the full response is finished; includes `usage` for billing.

You consume these in a loop. The audio deltas are what the user hears; the other events are for logging, UI, and tool execution.

### 5. Interrupt

If the user starts speaking while the model is still emitting audio, you need to stop the model fast. Two paths:

- **Server VAD will do it for you.** When `turn_detection.type = server_vad` and `interrupt_response: true` (default), the server detects the user's incoming speech and automatically cancels the in-flight `response`. You still have to stop playing any buffered audio you've already received.
- **Manual.** Send `response.cancel`. The server stops generating; it then emits a `response.done` with `status: "cancelled"`.

In either case, your client is responsible for:

1. Stopping audio playback immediately and flushing the player buffer. Any audio frames you've already received but not yet played must be discarded; the user interrupted the sentence the agent was about to finish.
2. Truncating the conversation history to what the user actually heard. The Realtime API exposes `conversation.item.truncate` for exactly this — tell the server "the model's last turn effectively ended at audio-ms X," so future context reflects what the user heard, not what the model generated after X.

Barge-in is a Module 106 topic; the plumbing for it lives here. Design the interrupt handler before you write the happy path, because adding it later means retrofitting playback-buffer accounting into a pipeline that doesn't have it.

### 6. Function calling inside the voice loop

The model can call tools mid-response — "check order status," "look up patient record," "cancel the appointment." The shape:

1. You declared `tools` in `session.update` with JSON-Schema signatures (identical in shape to Chat Completions function calling).
2. Mid-response, the server emits `response.function_call_arguments.delta` events as the model streams its arguments JSON, followed by `response.function_call_arguments.done` with the complete arguments string and a `call_id`.
3. You execute the tool on your backend.
4. You append the result back to the conversation with:
   ```jsonc
   { "type": "conversation.item.create",
     "item": { "type": "function_call_output", "call_id": "<...>", "output": "<json>" } }
   ```
5. You send `response.create` to ask the model to continue with the tool result in hand.

Two latency practicalities:

- **Cover the gap.** Tool calls take hundreds of milliseconds to seconds. Users notice silence. Pattern: as soon as the function call begins, emit a filler ("let me check that for you") via a short `response.create` with `modalities: ["audio"]` and a one-sentence instruction — then run the tool and send the real response. Some teams precompute a short audio stub per tool to avoid even the filler synthesis latency.
- **Keep tool outputs small.** The tool's output goes into the context window and is tokenised. A 5 KB JSON blob costs tokens and TTFB. Return the minimum viable payload for the next turn.

### 7. End

Close the socket. The session and its conversation state are discarded. There is no persistent conversation ID; if you want to resume, you reconstruct the conversation on a fresh session by replaying the salient `conversation.item.create` events.

## Turn detection in depth

The `turn_detection` object is the single most consequential knob in session config. Three settings matter.

| Setting | Behaviour | When to pick |
|---|---|---|
| `null` | No server VAD. You commit turns manually via `input_audio_buffer.commit` + `response.create`. | Push-to-talk UX; upstream VAD you trust; deterministic tests. |
| `{type: "server_vad", ...}` | Server-side energy-based VAD decides when the user is done talking. Fast, works OK on clean audio, tunable via `threshold` and `silence_duration_ms`. | Default for conversational voice agents where the user's audio is reasonably clean. |
| `{type: "semantic_vad", ...}` | Server-side model-based VAD that uses linguistic cues (not just silence) to decide whether the user is really finished. Reduces interrupting the user mid-thought ("I was going to say — [pause] — three things"). Verify availability and current parameter surface at integration time. | Conversations where the user takes natural pauses mid-utterance; usability wins over raw latency. |

The three tuning knobs under `server_vad` worth naming:

- `threshold` (0.0–1.0): the energy threshold for the VAD classifier. Lower is more sensitive (catches quiet speakers; more false triggers on background noise).
- `prefix_padding_ms`: how much audio to include before the first detected speech frame, so the first word is not clipped.
- `silence_duration_ms`: how long of a quiet tail before the server declares end-of-turn. Lower is snappier; higher avoids cutting off users mid-thought.

Module 106 is the place to actually tune these; name them here so the vocabulary is familiar when you get there.

## Prompt caching in the voice loop

Realtime sessions accumulate context fast: a few minutes of conversation can easily be thousands of tokens of transcript and tool exchanges on top of the system prompt. OpenAI's prompt-cache mechanism (automatic, token-prefix-based, applied to the model family) rewards long, **stable** prefixes — a system prompt, a tool catalogue, a long-form instruction block — with lower latency and lower cost for repeat invocations that reuse the same prefix.

What this means for session design:

- Put the stable part of the context — persona, tool schemas, policy language — at the **front** of the `instructions` string and the `tools` array. Append per-user and per-session variables **after** that block. The first few thousand tokens of a session are what the cache keys on; keep them identical turn to turn.
- Do not interleave per-call state into the system prompt. "The current time is …" and "the caller's name is …" belong in the first user turn or a dedicated setup `conversation.item.create`, not in `instructions`.
- When the session is long enough that the running conversation itself matters, note that each generation step prefixes the entire running conversation; the cache amortises across repeat invocations within and across sessions sharing the same system block.

Verify the specific cache-discount percentage and TTL on the current [pricing page](https://platform.openai.com/docs/pricing); they move. The structural guidance — stable prefix, variable suffix — is the invariant.

## Function calling, audio, and tool-heavy turns

The Realtime API is strong at chatty turns and competent at tool calls — but tool-heavy flows (multi-step function calling, long tool arguments, or tools that return large payloads) are the shape where speech-native stacks start to lose to the classical pipeline. Reasons:

- Each tool round-trip is model tokens + tool latency + user-perceived silence. A pipeline with a separate reasoning model can plan the tool sequence without speaking, then narrate a summary.
- Tool-call arguments stream as audio-adjacent events, which is cheap but not free; large argument JSON eats into TTFB for the follow-up audio response.
- The model can't easily "think out loud" without the user hearing it. Chain-of-thought is a text-mode behaviour.

Chapter 4 is the decision drill on pipeline vs. speech-native vs. hybrid. For now, hold the heuristic: **chatty, conversational turns → Realtime. Tool-heavy, multi-step, long-reasoning turns → pipeline (or hybrid).**

## A minimal walkthrough (WebSocket, pseudo-Python)

```python
import asyncio, base64, json, os, websockets

URL = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview"
HEADERS = {
    "Authorization": f"Bearer {os.environ['OPENAI_API_KEY']}",
    "OpenAI-Beta": "realtime=v1",
}

async def run(mic_audio_chunks, speaker):
    async with websockets.connect(URL, additional_headers=HEADERS) as ws:
        # 1. Configure the session.
        await ws.send(json.dumps({
            "type": "session.update",
            "session": {
                "modalities": ["text", "audio"],
                "voice": "verse",
                "input_audio_format": "pcm16",
                "output_audio_format": "pcm16",
                "input_audio_transcription": {"model": "whisper-1"},
                "turn_detection": {
                    "type": "server_vad",
                    "silence_duration_ms": 500,
                    "create_response": True,
                },
                "instructions": "You are a concise voice assistant...",
            },
        }))

        # 2. Pump mic in, read events out, concurrently.
        async def pump_mic():
            async for pcm16_chunk in mic_audio_chunks:
                await ws.send(json.dumps({
                    "type": "input_audio_buffer.append",
                    "audio": base64.b64encode(pcm16_chunk).decode(),
                }))

        async def read_events():
            async for raw in ws:
                event = json.loads(raw)
                match event["type"]:
                    case "response.audio.delta":
                        speaker.write(base64.b64decode(event["delta"]))
                    case "response.audio_transcript.delta":
                        log("assistant:", event["delta"])
                    case "conversation.item.input_audio_transcription.completed":
                        log("user:", event["transcript"])
                    case "response.function_call_arguments.done":
                        await handle_tool(ws, event)
                    case "error":
                        log("error:", event["error"])

        await asyncio.gather(pump_mic(), read_events())
```

This is scaffolding, not production code — a real implementation needs reconnect, playback buffering, interrupt handling, audio clock drift correction, and the tool dispatch layer. Build those on top.

## Common anti-patterns

- **Treating the session as request/response.** You cannot "await the response"; you must concurrently push audio and read events. Any linear `send → recv` loop will deadlock the mic.
- **Shipping your API key to the browser.** The WebRTC path exists specifically so you don't have to. Mint ephemeral session tokens server-side; the long-lived key never leaves your backend.
- **Letting the audio clock drift.** The model emits PCM faster than realtime. If your player plays faster than it should, you fall behind; if it plays slower, you fall ahead. Rate-control playback against the sample-rate wall clock, not against the arrival rate of deltas.
- **Forgetting `conversation.item.truncate` on interrupt.** Without it, the model's context believes it spoke more than the user heard. Future turns reference content the user never actually heard, and the agent sounds confused.
- **Putting per-session variables in the system instructions.** Breaks prompt caching. Put them in a user-role conversation item, not the stable prefix.
- **Treating `semantic_vad` as free.** It is a more capable turn-detector but it has its own latency and failure modes; measure it on your audio before shipping.
- **No filler during tool calls.** Users experience the silence as the agent "hanging." Always cover the gap.

## Summary

- The Realtime API is a stateful event socket, not request/response. WebSocket for server-to-server (telephony, backend); WebRTC for client-to-server (browser, mobile).
- The session lifecycle is: `begin → session.update → append audio → commit (manual or VAD) → read response events → interrupt (optional) → end`. Learn it as a grammar; the next three chapters will show you the same grammar in other stacks.
- `turn_detection` is the highest-consequence knob: `null` (push-to-talk, manual), `server_vad` (energy-based, default), `semantic_vad` (linguistic turn-end, more natural). Pick explicitly.
- Function calling works inside the voice loop with the same JSON-Schema shape as Chat Completions, plus two extras: cover the latency gap with a filler, and keep tool outputs small.
- Interrupts need both server-side cancel (`response.cancel` or VAD-driven) and client-side playback flush plus `conversation.item.truncate`. Design them before the happy path.
- Prompt caching rewards a long, stable system prefix. Put variable state after the prefix, not inside it.

Next chapter: the same lifecycle grammar, but in Google's Gemini Live API — bidirectional WebSocket, native audio in and out, continuous video and screen-share in the same turn, and tool calling mid-turn.
