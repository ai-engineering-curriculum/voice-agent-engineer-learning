# 03. Streaming Graph for Maximum Parallelism

## Why this chapter

If you sum the p95 budgets from Chapter 1 end-to-end without any overlap, you blow through 800 ms before you leave the ASR leg. The only reason production voice agents hit a conversational target at all is that **legs overlap**: ASR streams partials while the user is still speaking, the LLM starts generating before the full prompt is composed if you cache correctly, and — most importantly — **TTS synthesis starts on the first LLM token, not on the completed sentence**.

This chapter is the architecture half of the budget. Chapters 1 and 2 told you the numbers; this one tells you the graph.

## The sequential baseline, and why it fails

The naive pipeline is a chain of blocking stages:

```
asr.final ─► prompt.build ─► llm.generate (done) ─► tts.synthesize (done) ─► playout
```

Turn gap under this design ≈ ASR final + orchestration + LLM full generation + TTS full synthesis + playout.

A typical p95 for a 25-token agent utterance:

- ASR final: 200 ms
- LLM full generation (TTFT 300 ms + TPOT 30 ms × 25 tokens = 750 ms): ~1050 ms
- TTS full synthesis (TTFB 200 ms + the rest): ~600 ms
- Playout and transport: 100 ms

Sum: ~2.0 s. For an 800 ms target, this is unshippable. Three changes each cut hundreds of milliseconds.

## Parallelism move 1: start TTS on the first LLM token

The LLM streams tokens one at a time. A streaming TTS provider accepts text incrementally too. If you forward tokens to TTS as they arrive, the first audio byte can leave the server shortly after the LLM's first token — not after the LLM's last token.

The new graph:

```
asr.final ─► prompt.build ─► llm.generate (streaming) ──┐
                                                        ├─► tts.synthesize (streaming) ─► playout
                                            tokens ────►┘
```

The turn gap is now bounded by:

- ASR final + orchestration + **LLM TTFT** + TTS TTFB + playout.

Notice what dropped out: TPOT × N and TTS synthesis time of the rest of the utterance. Both still have to happen — they just happen *during* audio playback rather than before it. As long as TTS can produce audio faster than the user listens to it, the user never notices the cadence; you only pay for the first byte.

### Three implementation details that are easy to get wrong

1. **Phrase / sentence boundaries vs. per-token streaming.** Some TTS engines prefer at least a short phrase (a few words or a phrase-ending punctuation mark) before they synthesise, because prosody over a single word is poor. In practice, buffer until you see either a sentence boundary (`.`, `?`, `!`) or a comma or a hard word count (e.g., 6–10 tokens), whichever comes first. The right threshold is TTS-vendor-dependent; measure first-byte latency against buffer size and pick the inflection point.
2. **Backpressure.** If TTS synthesis falls behind the LLM's token rate, the token buffer grows. Prefer unbounded buffers for a *single turn* (the LLM finishes before the user could listen to that much audio anyway) but enforce a hard cap to detect pathological cases.
3. **Non-speakable tokens.** If the LLM is doing tool calls, structured output, or markdown-flavoured text, you can't just pipe bytes to TTS. The orchestrator has to parse the stream, extract the "speakable" tokens, and either silently handle the rest (tool call → execute → continue) or substitute a spoken description of what happened. This is where "streaming graph" becomes "streaming parser".

### Pseudocode sketch

```python
async def generate_and_speak(prompt):
    speakable = SpeakableTokenFilter()        # strips tool-call JSON, markdown, etc.
    phrase_buffer = PhraseBuffer(min_words=6, break_on=".?!,;")

    async for token in llm.stream(prompt):
        for chunk in speakable.feed(token):
            phrase_buffer.feed(chunk)
            while phrase_buffer.ready():
                await tts.send(phrase_buffer.pop())

    for chunk in speakable.finish():
        phrase_buffer.feed(chunk)
    await tts.send(phrase_buffer.drain())
    await tts.end_of_input()
```

The key invariant: `tts.send(...)` is called **during** the LLM stream, not after it.

## Parallelism move 2: cancellable, interruptible execution

The user barging in — starting to speak while the agent is still talking — is a constant. The pipeline has to:

1. Detect the start of user speech (VAD first-speech, covered in mod-106).
2. Cut TTS playback and stop pulling tokens from the LLM.
3. Cancel LLM generation (close the provider stream; most providers charge for tokens delivered, so cancellation matters for cost too).
4. Flush downstream buffers so the user doesn't hear a tail of audio that was in flight.

Patterns that work:

- **Structured concurrency.** Wrap the whole "LLM + TTS + playout" tree in a single scope (an asyncio `TaskGroup`, a Rust `tokio::select!`, a Go context) so cancelling the root cancels every leaf. The alternative — manually cancelling each stage — ships bugs where a cancelled TTS keeps pulling from an uncancelled LLM, draining your provider credit for nothing.
- **Cancellation tokens on the provider SDK.** Every modern LLM and TTS SDK supports request cancellation through context/abort signals. Use them. Closing the socket is a secondary mechanism, not a primary one.
- **Cut audio *locally*, then signal remotely.** The highest-priority action on barge-in is silencing the client's playback, which you control locally. Cancelling the provider streams is a few hundred milliseconds behind and is fine to do after local cutoff.

If the pipeline cannot cleanly cancel, every barge-in produces either (a) a ghost tail of audio, or (b) wasted provider spend, or both.

## Parallelism move 3: prompt caching for repeat-heavy prefixes

Voice agents have a lot of repeat structure:

- A system prompt that is identical on every turn.
- A persona / tool list / policy that changes rarely.
- A greeting ("Thanks for calling, this is Ava…") that is identical on the first turn of every call.
- A knowledge-base snippet that is reused across many turns inside one call.

Every major LLM provider now supports some form of **prompt caching**: if a prefix of the prompt is the same as a recent prefix, the provider skips the compute for that prefix and bills it at a reduced rate. The exact semantics vary by provider — cache TTL, cache granularity, and billing discount are all different — so consult the vendor docs before you commit to an architecture.

What matters for the budget:

- Prompt caching **reduces TTFT substantially** for cached prefixes, because the model does not have to re-process the cached tokens. On a very long system prompt, this can shift TTFT from "the top line of the turn gap" to "a minor contributor".
- Prompt caching is a **budget move, not a cost move**. The cost savings are nice; the latency shift is the structural win.
- The cache is only useful if **the same prefix is sent repeatedly**. If your prompt assembly reorders sections, interpolates per-turn data into the middle of the prompt, or regenerates system instructions, the cache will miss.

Design implication: keep all static content at the *start* of the prompt. Append per-turn content at the end. Resist the urge to "just put this one dynamic variable up top for readability".

## Parallelism move 4: warm pools and connection reuse

Opening a fresh TCP + TLS + WebSocket to an LLM or TTS provider on the first turn of a call adds 100–300 ms depending on the region. On every subsequent turn it should add zero.

Techniques:

- **Persistent sessions per call.** Open the LLM / TTS / ASR connections when the call connects, not when the first turn happens. Reuse them for every turn in the same call.
- **Pre-warmed pools of connections at the media server.** For low-latency routes to specific providers, keep a small idle pool of established connections and hand them to new calls on demand.
- **HTTP/2 or HTTP/3 multiplexing** where the provider supports it; one connection carries many concurrent requests.
- **Pre-flight the first turn.** Some teams send a tiny no-op request at call connect to prime the LLM's tokeniser, prompt cache, and connection — paying the TTFT variance *before* the user is waiting on it. This only helps if the first real turn hits a warm prompt cache from the prime, so test before shipping.

The cost side: idle connections consume provider quotas and sometimes idle fees. The budget side: for a product with a p95 turn-gap target, not having warm connections is architectural self-sabotage.

## Parallelism move 5: LLM / TTS speculation (advanced)

If you are willing to over-invest for a specific turn — say, the first turn of the call, which is the first impression — you can:

- **Pre-synthesise the greeting.** If your first turn's greeting is static, synthesise it once at startup, store the audio bytes, and play them on call connect. No LLM, no TTS, no latency.
- **Speculatively start generating on partial ASR.** As soon as ASR has emitted enough partial text that you are confident of the user's intent, speculatively kick off a draft LLM call. On confirmation (ASR final matches the partial), you use it; on mismatch, you cancel. This trades provider spend for latency and is only worth it on the most latency-sensitive legs.
- **Parallel LLM completions with rapid cancellation.** For a known small set of plausible responses (e.g., canonical confirmations), you can fan out to multiple models or prompts and take the first one that commits. Again, cost-vs-latency.

These are not defaults. They are tools for specific, measured bottlenecks.

## Putting it all together

The architectural shape is a **streaming graph** rather than a pipeline:

```
       ┌── asr (partials ───► orchestrator (speculative prompt build) ─────┐
       │                                                                   ▼
audio ─┤                                                        llm.stream (TTFT-first)
       │                                                            │       │
       └── asr (final) ────► prompt finalise ──► llm.inject ────────┘       │
                                                                            ▼
                                                             speakable-token filter
                                                                            │
                                                                            ▼
                                                              phrase buffer + boundary
                                                                            │
                                                                            ▼
                                                            tts.stream (TTFB on phrase 1)
                                                                            │
                                                                            ▼
                                                                  playout to client
```

Everything to the right of "asr.final" runs concurrently with "the rest of the LLM output". That is the whole trick.

## A defensive checklist before shipping

- [ ] Is TTS synthesis started on the **first LLM token** (or first phrase), not on the completed sentence?
- [ ] Is barge-in cancellation **structured** — a single abort signal reaches ASR, LLM, TTS, and playout?
- [ ] Is prompt caching **enabled** and is the static content kept at the **start** of the prompt?
- [ ] Are **LLM / TTS / ASR connections** opened at call-connect, not at first-turn?
- [ ] Have you measured the **TTS first-byte latency vs. phrase buffer size** curve for your TTS provider, and chosen the inflection point?
- [ ] Is the **speakable-token filter** in place so tool calls / markdown don't leak into the audio stream?

If you can honestly tick all six, the architecture is doing its job. The rest of the budget is region/edge (Chapter 4) and enforcement (Chapter 5).

## Summary

- Start TTS on the **first LLM token**, buffering to a short phrase boundary where prosody demands it. This is the single largest architectural move in the budget.
- Make the whole graph **cancellable** under one abort signal so barge-in doesn't leak audio or spend.
- Use **prompt caching** on static prefixes and keep the prompt layout cache-friendly.
- **Warm your connections** at call-connect; don't eat TLS and session-establishment on the first turn.
- Speculative moves (pre-synthesised greetings, speculative LLM on partials) are tools for *specific measured bottlenecks*, not defaults.

Next chapter: where to put all this compute — region, edge, and private peering.
