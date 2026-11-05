# 03. The Interrupt Path: Barge-In, Cancel-in-Flight TTS, and State Preservation

## Why this chapter

A voice agent that cannot be interrupted is not a voice agent; it is an IVR. The ability for the user to speak over the bot, have the bot stop within a couple of hundred milliseconds, and continue the conversation without losing state is the single feature that separates "feels like a conversation" from "feels like a push-to-talk form." Everything you built in Chapters 1 and 2 feeds into this chapter: the VAD on the user mic has to fire while the bot is speaking, the EOU logic has to understand that an interrupt is not a turn boundary in the normal sense, and the generation stack (TTS + LLM) has to reverse safely in flight.

This is also the chapter where the latency budget bites. A barge-in that reacts in 600 ms feels broken; one that reacts in 150 ms feels human. We target **<200 ms from user speech onset to bot audio stopping**, and we design the path around that target.

This chapter is about **user-interrupts-bot** primarily (the dominant case), with a shorter treatment of **bot-interrupts-user** (backchannels, "go ahead" cues, and the non-speech-interrupt rules). Chapter 4 is the overlap and backchannel classifier that decides, for a given user utterance mid-bot-turn, whether it is a barge-in or just an "mm-hm."

## The anatomy of a barge-in

When the user starts speaking while the bot is speaking, five things must happen, in order, within ~200 ms of speech onset:

1. **Detect** — the mic-side VAD fires on the user's audio.
2. **Classify** — is this a backchannel ("mm-hm", "right") or a true interrupt ("wait, no")? Chapter 4. For the rest of this chapter, assume the classifier says "interrupt."
3. **Cancel** — stop generating new content. This has two layers: stop LLM generation, stop TTS synthesis.
4. **Flush** — drop audio frames you have buffered but not yet played. The user hears "...and the next step is—" cut cleanly; they do not hear five seconds of buffered speech finish playing.
5. **Truncate** — update the conversation state to reflect what the user actually heard, not what the bot generated. Otherwise the next turn references content the user never heard.

Each of these layers has a specific latency contribution and a specific failure mode. Let's walk them.

## Where the 200 ms goes

Suppose the user says "wait, no —" starting at `t=0 ms`. Breakdown of the typical latency budget:

| Stage | Typical | Notes |
|---|---|---|
| Audio capture + packetisation at the client | 20–40 ms | Opus frame size × packetiser. |
| Network transit to server (regional) | 20–80 ms | WebRTC, telephony hairpin, or vendor socket. |
| VAD inference + decision | 30–50 ms | Silero at 30 ms frame rate + hangover-free "speech started" detection. |
| Classifier (barge-in vs. backchannel) | 20–100 ms | Zero if you commit to interrupt-on-speech; up to 100 ms if you run Chapter 4's classifier. |
| Cancel signal to TTS / LLM | 10–30 ms | Protocol round-trip on the generation socket. |
| Playback flush | 5–20 ms | One audio-player buffer drain. |

Sum of the fast path: ~120–200 ms if everything behaves. Sum of the slow path: 300–400 ms, which is where the "it feels sluggish" complaints come from. Treat the 200 ms target as the budget you design against; measure p95, not p50 — the failure case is the one that defines the UX.

## Stage 1: detect

The mic-side VAD runs continuously, including while the bot is speaking. Three details you have to get right:

**Echo-cancelled audio only.** The mic captures the room, which includes the bot's own voice coming out of the speaker. Without acoustic echo cancellation (AEC), the mic-side VAD sees the bot's speech as "user speech" — the bot interrupts itself. AEC is a hard dependency; see Module 102 Chapter 5. On a headphone deployment AEC is unnecessary (no feedback path). On a speakerphone / telephony deployment AEC is mandatory, and must be running before the VAD, not after.

**No hangover for the start edge.** Chapter 1's VAD uses `silence_duration_ms` on the trailing edge to decide end of turn. For barge-in, you want **zero hangover on the leading edge** — the moment VAD crosses the speech threshold, you want to act. The hangover only applies to the "stop" decision, not the "start" decision. In Silero this is the natural behaviour; in some implementations with symmetric hangover, verify.

**Signal-to-noise discipline.** During bot-speaks, the room acoustics are different. AEC leaves residuals; the mic is picking up speaker reflections; the VAD threshold tuned for user-speaks-alone may misfire. Either tune a second threshold for the bot-speaking window, or (simpler) require 60–120 ms of continuous above-threshold speech before declaring barge-in, so a single AEC residual spike doesn't fire.

## Stage 2: classify

The classifier call ("is this barge-in or backchannel?") is Chapter 4's content; the integration point is here. Two engineering decisions:

**Does the classifier block the cancel, or does cancel happen in parallel?** The safe default is parallel: start cancelling TTS the moment VAD declares speech-started, then classify the user audio; if the classifier says "backchannel," restart the bot's utterance from where it left off (hard — see Stage 5). The alternative — block cancel on classifier output — adds the classifier's latency to the 200 ms budget and costs you the fast-reaction feel.

**What is your minimum confidence before cancelling?** On a noisy mic with AEC residuals the classifier will be less certain. A reasonable policy: cancel if the classifier confidence is above 0.5 OR the user audio has been active for more than 400 ms (clearly not a transient). The duration floor catches true barge-ins the classifier is uncertain about.

A simpler policy that ships well: **always cancel on VAD speech-start; let the backchannel classifier only prevent the LLM from treating the backchannel as a turn.** You get snappy barge-in universally, and the cost of mis-cancelling a backchannel is low — the bot just resumes from a natural pause point. The cost of mis-waiting for a classifier when the user is actually barging in is high — a sluggish-feeling bot.

## Stage 3: cancel in-flight TTS

The TTS is producing audio faster than the user is hearing it; several seconds of synthesised PCM may be in flight between the TTS server and your player. Cancelling means:

### If you run a pipeline with a streaming TTS vendor

- **Close the TTS stream or send the provider's cancel event.** ElevenLabs' websocket, Rime, Deepgram TTS, OpenAI TTS streaming — each has its own close / stop semantics; cross-check the SDK. On most, closing the websocket is effective; some provide a `stop` message to flush gracefully.
- **Stop paying for synthesis you'll never play.** For character-based billing TTS, every token not spoken is money. Design the cancel to arrive at the TTS provider as fast as possible; a 200 ms earlier cancel on a chatty bot across millions of calls is a real cost line.
- **Release the voice slot.** Some TTS providers rate-limit concurrent sessions per voice; failing to close them cleanly starves future requests.

### If you run speech-native (OpenAI Realtime, Gemini Live)

- **OpenAI Realtime.** With `turn_detection.type: "server_vad"` and `interrupt_response: true`, the server will automatically cancel the in-flight `response` when it detects incoming user speech. Equivalently (manual mode, or if you distrust the server's detection), send `response.cancel`. In both cases, you still have to do Stages 4 and 5 on the client — the server cancels generation, not the audio in your player buffer.
- **Gemini Live.** Analogous: the Live session supports an interrupt event; verify the current field name and semantics at integration time. The server stops generating when it hears incoming user audio.

<!-- needs-research: verify the current Gemini Live interrupt field name and semantics at author time -->

### Cancel the LLM, not just the TTS

In a pipeline deployment, the LLM is still streaming tokens into the TTS; if you close the TTS but not the LLM generation, the LLM continues to be billed and produces tokens no one will speak. Cancel the LLM generation first (close the HTTP stream, abort the request), **then** close the TTS. Order matters for cost.

```python
# Pseudocode: cancel order
async def on_barge_in_detected():
    await llm_stream.cancel()      # stop generating new tokens
    await tts_socket.close()       # stop synthesising new audio
    audio_player.flush()           # drop buffered audio (Stage 4)
    await conversation.truncate_at(audio_player.last_played_ms)  # Stage 5
```

## Stage 4: flush the playback buffer

This is where naive implementations lose their users. The TTS stream delivers audio faster than realtime; your player has buffered 3–5 seconds of audio. If the player drains the buffer naturally, the "cancel" is invisible — the bot keeps talking for several more seconds after the user tried to interrupt.

**The player must be interruptible.** On stop:

- Clear the output queue immediately.
- Fade out the last 20–50 ms of audio (a hard cut produces a click; a short fade sounds clean).
- Record the `audio_played_ms` — the exact position in the response where playback actually stopped. This is what Stage 5 needs.

A reasonable player API shape:

```python
class InterruptiblePlayer:
    def __init__(self):
        self.output_queue: asyncio.Queue[bytes] = asyncio.Queue()
        self.played_ms: int = 0  # cumulative audio-ms actually sent to the sink

    async def feed(self, pcm_bytes: bytes):
        await self.output_queue.put(pcm_bytes)

    def stop(self) -> int:
        """Drop pending audio, fade out, return ms of audio played before stop."""
        drained = self._drain_queue()
        self._fade_out(ms=30)
        return self.played_ms
```

The `played_ms` return is the handoff to Stage 5. Without it, the conversation state will believe more was spoken than the user heard.

## Stage 5: truncate conversation state

The LLM's context still contains the full response it was generating. On the next turn, the LLM will reason as if the user heard the entire response. The user heard only the first N seconds. The bot's next reply references content that, from the user's perspective, never existed — "as I mentioned, step three is to…" when step three was in the part the user interrupted.

The fix is to truncate the model's view of its own last turn to what the user actually heard.

### OpenAI Realtime — `conversation.item.truncate`

```jsonc
{
  "type": "conversation.item.truncate",
  "item_id": "<the response item id>",
  "content_index": 0,
  "audio_end_ms": 1480     // ms of audio the player confirmed it played
}
```

The server rewrites the server-side conversation state to reflect that the model's utterance effectively ended at `audio_end_ms`. See the [Realtime API reference](https://platform.openai.com/docs/api-reference/realtime).

### Pipeline (STT + LLM + TTS)

You are the server-side conversation state. On cancel:

1. Compute the character / token position in the generated response corresponding to `played_ms` using the TTS's word-timing output (streaming vendors expose per-word timestamps; if yours does not, approximate by assuming a constant speaking rate — ~15 characters per second as a rough prior).
2. Rewrite the last assistant turn in the running conversation history to the truncated prefix.
3. Optionally append a short suffix like `[interrupted]` or `[interrupted here]` so the LLM knows the next turn starts from an interrupted state. Prompt the model to acknowledge the interruption naturally.

```python
def truncate_last_turn(history, assistant_text, played_ms, word_timings):
    cutoff_idx = next(
        (i for i, t in enumerate(word_timings) if t.end_ms > played_ms),
        len(word_timings),
    )
    truncated = " ".join(w.text for w in word_timings[:cutoff_idx])
    history[-1] = {
        "role": "assistant",
        "content": truncated + " [interrupted]",
    }
    return history
```

### Why you cannot skip this

Without truncation, two specific defects appear reliably:

- **"As I was saying."** The agent refers to content the user didn't hear.
- **Looping.** The agent re-opens a topic the user already redirected away from, because its context says it answered a question the user then interrupted before hearing.

Both defects are in the top-5 complaints on any voice-agent product that shipped barge-in without truncation.

## Preserving partial state so the response can pick up

The flip side of truncation: sometimes the user interrupts not because they want a new topic, but to add a parameter. "Book a flight to San Francisco on—" the user interrupts with "wait, on Tuesday, not Monday."

Design question: can the bot pick up its previous turn, now parameterised by the user's correction, instead of starting over?

Two approaches worth knowing:

1. **Let the LLM do it.** Post-truncate, append the user's new turn, and let the LLM generate a response that incorporates both the truncated prior response and the correction. Modern LLMs do this well without explicit scaffolding. Cost: one extra inference; latency: the time of that inference.

2. **Explicit state carry.** If the response was a tool call in progress, do not re-issue the tool call — reuse the partial arguments the LLM had emitted, patch the one field the user corrected, and resume. This is engineering-heavy and only pays off on tool-heavy pipelines.

For most voice agents, Approach 1 is correct. Approach 2 is a Module 111 orchestration topic; name it here and move on.

## Bot-interrupts-user: a shorter treatment

The symmetric case — the bot wants to intervene while the user is speaking — is rarer but matters in specific product shapes.

### Backchannels

A short, non-floor-taking acknowledgement the bot emits while the user is still speaking: "mm-hm," "right," "sure." These are not interruptions; they are rapport signals. They are also optional; many production voice agents omit them entirely.

If you want them:

- Emit on prosodic cues from the user (end of a clause, a brief pause) — VAP can supply these; a silence-detection heuristic over user audio works adequately.
- Keep them short (<300 ms) and quiet (−6 dB vs. the main voice).
- Do not pause TTS to emit them; use a separate audio lane if your player supports it.

### "Go ahead" / yield cues

When the user starts, pauses, and the bot can't tell if they are done, a short "go ahead" or silent acknowledgement nudges them to continue. Very occasionally appropriate; often grates. Measure before shipping.

### Non-speech interrupts

The bot interrupts not with speech but with a non-speech signal — a tone, a UI update. Appropriate for safety-critical cases ("you're approaching your appointment time") where the bot must notify the user without taking the floor. Rare in general agents; worth naming so you know the pattern exists.

### The hard rule: bot does not take the floor from a user mid-utterance

Unless the product shape is an explicit interview / proctoring UX, the bot should not interrupt the user's utterance to take the turn. It can backchannel, it can emit non-speech signals, but it should not start speaking over. Users experience bot-takes-the-floor as rudeness; the agent's job is to listen.

## Deployment patterns

### Pipeline deployment

- User audio: AEC → VAD → STT.
- Bot audio: LLM stream → TTS stream → interruptible player.
- Barge-in signal: VAD on user audio with no leading hangover.
- Cancel: close LLM stream; close TTS stream; flush player; truncate conversation history client-side.

### Speech-native deployment (OpenAI Realtime)

- User audio: client mic → server socket.
- Bot audio: server → `response.audio.delta` → interruptible player.
- Barge-in signal: server-side VAD + `interrupt_response: true` (automatic), or manual `response.cancel`.
- Cancel: server stops generation on its own; client flushes player; client issues `conversation.item.truncate` with `audio_end_ms`.

### Telephony deployment

- Add: a jitter buffer on both directions. Audio arrival is bursty; the player must account for that when computing `played_ms`.
- Add: DTMF as an alternate interrupt channel. Some users will press a key; treat as an interrupt event with the key as the user turn's content.
- Caveat: AEC and noise-suppression quality on PSTN is worse; the barge-in false-trigger rate goes up. Tune.

## Observability: what to log on every barge-in

- Timestamp of `response.created` and timestamp of the server's / your `response.cancel` or TTS close.
- `audio_played_ms` at the moment of cancel.
- `audio_generated_ms` at the moment of cancel (how much was synthesised but not played — your waste).
- Classifier output (barge-in vs. backchannel, confidence).
- User's intent after interrupt (from the next user turn's transcript).

These five fields give you the dashboard for the Chapter 5 metric suite: p95 interrupt reaction time, false-barge-in rate, missed-barge-in rate, and the characteristic defect that follows unlogged interrupts — "the bot keeps saying things I interrupted."

## Common anti-patterns

- **No AEC.** The bot's own voice re-enters the mic; the VAD fires on its own speech; the bot cancels itself mid-sentence. The symptom is the bot cutting itself off; the fix is AEC upstream of VAD.
- **Leading-edge hangover on VAD.** Hangover protects the trailing edge of a user utterance. If you also hang over the leading edge, your barge-in detection is intrinsically 300 ms late. Zero leading-edge hangover; let the classifier (Chapter 4) handle spurious short triggers.
- **Blocking cancel on classifier.** Running the backchannel classifier before cancel adds its latency to the barge-in budget. Cancel in parallel; let the classifier only decide whether to restart.
- **Non-interruptible player.** The TTS buffer drains naturally; the bot keeps talking for seconds after the "cancel." Fix the player.
- **No conversation truncate.** The next turn references content the user never heard. Visible in log playback; easy to miss in dev; obvious in prod.
- **Treating all user speech during bot-speaks as barge-in.** You cut users off for saying "mm-hm." Chapter 4.
- **Cancelling TTS but not LLM.** The LLM keeps generating tokens that go nowhere; you pay for generation you discard.
- **No per-barge-in log row.** When product reports "the bot sounds laggy," you have no data to diagnose. Log every interrupt.

## Summary

- Barge-in is the five-stage path: detect (VAD on user mic), classify (Chapter 4), cancel (LLM then TTS or vendor response), flush (player drops buffered audio), truncate (conversation state reflects what the user actually heard).
- The target is < 200 ms from user speech onset to bot audio stopping. Measure p95, not p50.
- Zero hangover on the leading edge of user VAD during bot-speaks. Trailing hangover is for end-of-turn, not start-of-interrupt.
- AEC is mandatory on any deployment with a speaker-and-mic in the same acoustic space. Without it, the bot interrupts itself.
- `conversation.item.truncate` on OpenAI Realtime; equivalent server-side rewrite in pipeline deployments. Without truncation the bot references content the user never heard.
- Bot-interrupts-user is a narrower design space: backchannels (optional, prosody-cued), "go ahead" cues (occasionally useful), non-speech signals (safety-critical). The bot does not take the floor from a user mid-utterance.
- Log the five fields: `response.created`, cancel timestamp, `audio_played_ms`, `audio_generated_ms`, classifier output, user's post-interrupt intent. This is the input to Chapter 5's metric suite.

Next chapter: the backchannel-vs-turn-grab classifier. How to tell "mm-hm" (hold the floor) from "wait, no" (take the floor) in under 200 ms, and how to train the classifier on the data you actually have.
