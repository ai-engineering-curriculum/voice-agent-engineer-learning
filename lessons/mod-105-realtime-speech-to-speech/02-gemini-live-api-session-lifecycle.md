# 02. The Gemini Live API: Bidirectional Streaming, Native Audio, and Continuous Video

## Why this chapter

Google's Gemini Live API is the second realtime speech-native stack you need to know by name, and it is deliberately different from OpenAI's — enough so that one of the two always wins on any given deployment. Where the Realtime API's defining shape is a session socket with audio + tool calls, Gemini Live's defining shape is a **bidirectional streaming conversation that natively ingests audio, continuous video, and screen-share in the same turn**, and emits either native audio or text + synthesised audio. The video dimension pulls it into territory the Realtime API does not cover — the live kitchen-camera assistant, the field-service worker walking you through a panel on their phone, the tutor watching a student's screen.

This chapter is the same contract walk as Chapter 1 but in Gemini's vocabulary: the model variants, the WebSocket transport, the setup / realtime-input / tool-call event grammar, continuous video and screen-share, grounding with Google Search, and the places the two APIs' lifecycles diverge in ways you have to design around.

## The mental model

Gemini Live is also a **stateful bidirectional socket**. A single WebSocket connection carries:

- **Client → server:** a one-time `setup` message, then an open-ended stream of `realtime_input` messages carrying audio chunks, video frames, text, and screen-share frames; plus `tool_response` messages when the model calls a function.
- **Server → client:** a `setup_complete` message, then an open-ended stream of `server_content` events (interim transcripts, streamed audio or text, interrupt signals, turn completion) and `tool_call` events when the model wants to call a tool.

The socket is the session. Unlike the Realtime API, there is no "send events to configure session mid-flight" grammar — configuration happens once in `setup` and then you're committed; a mid-flight parameter change means a fresh socket. Hold that difference; it shapes how you build the client.

See the official reference at [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live).

## Model and endpoint surface

Gemini Live is served by model variants under the Live API. As of this writing, the relevant names you will see in Google's docs are the Gemini 2.0 and 2.5 Flash Live families, sometimes labelled with "native audio" vs. "half-cascade" variants (see Chapter 5 for the half-duplex/full-duplex distinction). Verify current model IDs and availability at [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live) at integration time — the IDs rotate faster than any curriculum can track. <!-- needs-research: confirm current Gemini Live model IDs and the Live/Flash Live variant naming at author time -->

- **Endpoint (WebSocket):** `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=<API_KEY>` for the Gemini Developer API, or a Vertex AI equivalent for enterprise GCP projects.
- **Auth:** API key in the URL query (Developer API); OAuth / service-account auth (Vertex AI).
- **SDKs:** the `google-genai` Python SDK is the current reference; there are JavaScript and Go SDKs. The wire protocol is documented so you can roll your own client where the SDK is inconvenient.

## The session lifecycle

Walk this alongside Chapter 1's. The grammar is similar enough that you can port mental models; the places it diverges are where surprises land.

### 1. Begin — the `setup` message

Open the WebSocket. Send one `setup` message with the full session configuration:

```jsonc
{
  "setup": {
    "model": "models/gemini-2.5-flash-preview-native-audio-dialog",
    "generation_config": {
      "response_modalities": ["AUDIO"],       // or ["TEXT"]; audio and text are mutually exclusive per response
      "speech_config": {
        "voice_config": {
          "prebuilt_voice_config": { "voice_name": "Charon" }
        }
      },
      "temperature": 0.7
    },
    "system_instruction": {
      "parts": [{ "text": "You are a concise voice assistant..." }]
    },
    "tools": [
      { "function_declarations": [ /* JSON-Schema signatures */ ] },
      { "google_search": {} }                 // enable Google Search grounding, see §Grounding
    ],
    "realtime_input_config": {
      "automatic_activity_detection": {       // Gemini's server VAD, see §Turn detection
        "start_of_speech_sensitivity": "START_SENSITIVITY_LOW",
        "end_of_speech_sensitivity": "END_SENSITIVITY_LOW",
        "silence_duration_ms": 1000
      }
    }
  }
}
```

The server responds with `setup_complete`. Until then you cannot send media. This is the **one-shot config** moment; mid-session changes to `response_modalities`, `voice_config`, or tools are not supported. If the user asks to switch languages or voice, you either (a) change what you can within the socket (prompt-level steering, short of config changes) or (b) close the socket and open a fresh one.

Verify current field names and model IDs at integration time; the Live API is still evolving and field renames do happen between versions.

### 2. Append user input — the `realtime_input` message

The ingress for every kind of user input is `realtime_input`. You can mix modalities freely:

```jsonc
{ "realtime_input": {
    "media_chunks": [
      { "mime_type": "audio/pcm;rate=16000", "data": "<base64 PCM16 chunk>" },
      { "mime_type": "image/jpeg",           "data": "<base64 JPEG frame>" }
    ] } }
```

Rules:

- **Audio.** Linear PCM at **16 kHz, 16-bit mono**, base64-encoded. Note that this differs from the Realtime API's typical 24 kHz output — you may need resamplers on both sides. Chunk at ~100 ms or so; smaller chunks are fine.
- **Video.** JPEG or PNG frames at a cadence you choose. Google's own examples run at **1–2 frames per second** by default; higher cadences (up to a few fps) are supported but cost tokens and TTFB. The model does not need 30 fps — it needs enough samples to see what changed in the scene.
- **Screen-share.** Same wire format as video — JPEGs of screen captures. The model does not distinguish "video frame" from "screen frame" at the wire; the distinction is semantic for your UX.
- **Text.** Also allowed; useful for mixed modal input ("can you look at this screenshot and read the user's typed description").

You can send media on either or both of the audio and video lanes in a single turn. This is the big thing Gemini Live does that the Realtime API does not: a single conversational turn can include speech + sustained video + screen frames + typed text.

### 3. Turn detection

Gemini Live supports automatic (server-side) and manual turn detection, with slightly different vocabulary from OpenAI's.

- **Automatic (default).** `realtime_input_config.automatic_activity_detection` is enabled by default. The server watches the audio envelope and declares end-of-speech based on energy + silence duration. Tunable via `start_of_speech_sensitivity`, `end_of_speech_sensitivity`, and `silence_duration_ms`.
- **Manual.** Set `realtime_input_config.automatic_activity_detection: { "disabled": true }`. You then mark turn boundaries explicitly with `activity_start` and `activity_end` entries in the `realtime_input` payload. Equivalent to the Realtime API's push-to-talk / `input_audio_buffer.commit` mode.

The manual mode is the right pick when you have an upstream VAD you trust (Silero, Deepgram's embedded VAD), when the ingress is a push-to-talk UX, or when deterministic test harness behaviour matters.

### 4. The model responds — `server_content` events

Once the server has decided the turn is complete (or you asserted it manually), it streams `server_content` events back:

- `input_transcription` — the model's transcript of the user's audio input, streamed as the user spoke. Useful for logging and UI. (Opt in via `setup.input_audio_transcription` on current models; verify the exact field at integration time.)
- `output_transcription` — the model's transcription of its own audio response, streamed as it speaks. Use for captions.
- `model_turn.parts[].inline_data` — the model's audio response, **PCM at 24 kHz, 16-bit mono** (verify sample rate at integration time). Pipe to the player.
- `model_turn.parts[].text` — text parts if `response_modalities` is `["TEXT"]` or if the model produced a text-only turn.
- `interrupted` — a flag the server sets when it auto-cancels a response because the user started speaking. See §Interrupts.
- `turn_complete` — this generation step is done; the model is listening again.

You consume these in a loop, same as Chapter 1.

### 5. Tool calling mid-turn — `tool_call` events

Gemini Live supports function calling inside the voice loop. The shape:

1. You declared `function_declarations` in `setup.tools`.
2. Mid-turn, the server emits a `tool_call` event with one or more `function_calls`, each with a `name`, an `args` object, and an `id`.
3. You execute the tool on your backend.
4. You post the result back with a `tool_response` message:
   ```jsonc
   { "tool_response": {
       "function_responses": [
         { "id": "<call_id>", "name": "<tool>", "response": { /* json */ } }
       ] } }
   ```
5. The model resumes its response with the tool's output in context.

The novel capability here is **mid-turn tool calling with continuing audio**: the model can call a tool, briefly pause its spoken output, receive the result, and continue speaking without the client having to re-prompt. Combined with continuous video input, this makes it the strongest stack for "the user is showing the camera a scene and asking the model to act on it" shapes.

See Chapter 4 for when that mid-turn capability matters enough to pick Gemini over Realtime.

### 6. Grounding with Google Search

Add a `google_search: {}` tool entry in `setup.tools`. The model will (at its discretion) issue Google Search queries, retrieve snippets, and cite them in its output. The citations arrive as `grounding_metadata` on the relevant `server_content` turn.

Use cases: live information (weather, prices, breaking news), fact-checking, any voice-agent workflow where the model would otherwise hallucinate because its training is too old.

Trade-offs:

- Additional per-turn latency (a search round-trip inside the generation).
- Attribution requirements — you should display or voice the citations in some form when the model used them.
- Cost model differs from pure generation; verify current pricing.

Grounding is a **safety primitive as well as a feature**: for domains where hallucination is dangerous, it is the mitigation the model itself offers. Note it here; it recurs in Module 108 as a safety mitigation.

### 7. Interrupts

A user starts speaking while the model is emitting audio. Gemini Live's server VAD detects it, cancels the in-flight response, and sends a `server_content.interrupted` flag. You still have to stop playback and drop any un-played audio buffered on your side — exactly the same client-side obligation as in Chapter 1.

The specific affordance to know: Gemini Live's `interrupted` flag arrives as part of the response stream; there is no explicit `response.cancel` you send. The barge-in path is entirely VAD-triggered on the server side. When you need deterministic manual interrupts (e.g., "stop now" button in your UI), you do it by (a) closing the socket and reopening, or (b) running in manual-activity-detection mode and marking `activity_start` to pre-empt the server.

### 8. End

Close the WebSocket. Session state is gone. Session resumption and context compression (for keeping long sessions within the model's context window) are documented capabilities worth investigating for long-running deployments. <!-- needs-research: verify the exact session-resumption API surface for the current Live API version -->

## The bits that are Gemini-specific

Walk these one by one — each is where a port from Chapter 1 goes wrong.

### Continuous video + screen-share in a single turn

The Realtime API accepts audio in and emits audio out. Gemini Live accepts audio, video, text, and screen frames in, and emits audio or text out. The protocol doesn't distinguish "video" from "screen share"; they are both streams of image frames under `media_chunks`.

Implications:

- **Frame cadence is a budget.** 1–2 fps is the usual starting point. Higher fps costs latency and tokens; model performance generally does not improve linearly past ~2 fps for conversational use.
- **The model's "attention" to the video lane is probabilistic.** You cannot guarantee it will look at every frame. Design prompts that explicitly tell it "the user is showing you X"; design UX that confirms what the model saw.
- **Screen-share needs OCR-grade framing.** Downscaling a 4K screen to a thumbnail destroys the text. Capture at a resolution where the text in your app is legible to a human in a thumbnail; this is usually around the 1280-wide range.

### Response modalities are mutually exclusive per-session

`response_modalities` is either `["AUDIO"]` or `["TEXT"]`, not both. If you want the model to speak and emit text captions, you get the captions from `output_transcription`, not by asking for both modalities. This is a frequent gotcha when porting from the Realtime API, which multiplexes freely.

### No mid-session config updates

Once you've committed to a voice in `setup`, you don't change it without closing the socket. Any product UX that offers "switch voice mid-conversation" needs to be designed around reconnect — probably with a short, graceful handover.

### The audio sample rates differ

Input is 16 kHz PCM; output is 24 kHz PCM. Players and resamplers that worked on the Realtime API's 24 kHz symmetric shape may need adjustment.

### Session duration ceilings

Gemini Live sessions have documented maximum durations — minutes to tens of minutes depending on the variant. Long-running phone calls will need session resumption (documented) or multi-session stitching. <!-- needs-research: verify the specific session-duration limits for the current Live API variants at author time -->

## A minimal walkthrough (Python, pseudo-code)

Using the `google-genai` SDK; the SDK shape mirrors the wire protocol closely:

```python
import asyncio, base64, os
from google import genai
from google.genai import types

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
MODEL = "gemini-2.5-flash-preview-native-audio-dialog"  # verify current ID

CONFIG = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    speech_config=types.SpeechConfig(
        voice_config=types.VoiceConfig(
            prebuilt_voice_config=types.PrebuiltVoiceConfig(voice_name="Charon"),
        ),
    ),
    system_instruction=types.Content(
        parts=[types.Part(text="You are a concise voice assistant...")],
    ),
    tools=[types.Tool(google_search=types.GoogleSearch())],
)

async def run(mic_pcm_chunks, speaker):
    async with client.aio.live.connect(model=MODEL, config=CONFIG) as session:

        async def pump_mic():
            async for pcm16 in mic_pcm_chunks:   # 16 kHz PCM, mono
                await session.send_realtime_input(
                    media=types.Blob(mime_type="audio/pcm;rate=16000", data=pcm16),
                )

        async def read_events():
            async for response in session.receive():
                if response.data:                 # 24 kHz PCM audio
                    speaker.write(response.data)
                if response.text:
                    log("assistant:", response.text)
                if response.server_content and response.server_content.interrupted:
                    speaker.flush()               # user started speaking

        await asyncio.gather(pump_mic(), read_events())
```

This is scaffolding, not production code. Production needs reconnect, playback buffer accounting, interrupt flush, video pump, tool-call dispatch, and input-transcription logging.

## Picking Gemini Live over the Realtime API

Hold these as the distinguishing axes. Chapter 4 is the full decision drill.

- **You need continuous video or screen-share in the voice loop.** Gemini Live is the only speech-native stack that does this natively today. Not a tie — a decider.
- **You want Google Search grounding baked into the model.** Gemini Live is the native path. Realtime can do search via function calls; it is not the same shape.
- **You want 2.5 Flash's price point for long sessions.** Gemini Live's Flash variants are often cheaper per audio-second than `gpt-4o-realtime`; verify at integration time.
- **You are on GCP for data residency.** Vertex AI is a cleaner story than routing to a partner.

Pick OpenAI's Realtime API instead when:

- **You need telephony today with a mature partner ecosystem.** Twilio, LiveKit, and the voice-agent vendor stack has first-class Realtime adapters; Gemini Live is catching up.
- **You need mid-session config changes** (voice swap, language swap with distinct voice) without reconnecting.
- **Your team already runs on OpenAI's platform** and the ops overhead of adding a second vendor is not justified.

## Common anti-patterns

- **Porting a Realtime client without changing the sample rates.** 16 kHz in, 24 kHz out. If the player still thinks everything is 24 kHz, users hear chipmunks.
- **Sending video at 30 fps.** The model does not need it; your bandwidth bill and TTFB will suffer. 1–2 fps is the usual starting cadence.
- **Treating `interrupted` as optional.** When it fires, you must flush your playback buffer immediately. Users do not care whether it was client-side or server-side VAD.
- **Changing voice "live" without reconnect.** It does not work; the session is committed in `setup`.
- **Shipping Google Search grounding without surfacing citations.** The model's output includes attribution metadata; the UX has to carry it.
- **Running one long socket for an hour.** You will hit the duration ceiling. Design a session-resumption path up front.

## Summary

- Gemini Live is a stateful bidirectional WebSocket, `setup` → `realtime_input` events in, `server_content` events out, `tool_call` + `tool_response` for function calling.
- The single capability it owns that the Realtime API does not is **continuous video + screen-share + audio + text in one turn**, plus Google Search grounding as a first-class tool.
- Audio in at 16 kHz PCM; audio out at 24 kHz PCM; output modalities are mutually exclusive per session; config changes mean reconnect.
- Turn detection: automatic (server VAD, default) or manual (`automatic_activity_detection: { disabled: true }` + `activity_start`/`activity_end`). Same decision shape as Chapter 1.
- Interrupts arrive as `server_content.interrupted` flags; the client-side playback flush is still your job.
- Pick it when the deployment is video-in-the-loop, Search-grounded, GCP-native, or Flash-priced. Pick OpenAI's Realtime API when the deployment is telephony-mature, mid-session-dynamic, or already on that platform.

Next chapter: the third of the three major frontier labs — Anthropic Claude for voice — and why its current path is **not** a first-party realtime WebSocket but a partner realtime-relay story. Then the architecture lesson is to design your pipeline so the LLM leg is swappable.
