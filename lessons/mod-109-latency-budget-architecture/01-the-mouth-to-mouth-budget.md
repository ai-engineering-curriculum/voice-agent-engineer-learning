# 01. The Mouth-to-Mouth Budget

## Why this chapter

A voice agent either feels like a conversation or it doesn't. The single most reliable predictor of "feels like a conversation" is the **turn gap** — the time between the user ending an utterance and the agent's first audible syllable. Pick a loose budget and users talk over the agent, repeat themselves, or hang up. Pick an unrealistic budget and you over-engineer the pipeline, pay for private peering you don't need, and ship anyway because every leg blew through its allocation.

This chapter sets the target. The remaining chapters in the module tell you how to hit it, measure it, and defend it.

## G.114: the standard you inherit, not the one you aim at

ITU-T Recommendation G.114 is the long-standing reference for **one-way mouth-to-ear delay** on a telephony voice call. Its guidance is framed as "transmission time" — the delay a listener experiences between the speaker producing a sound and the listener hearing it — and the thresholds are:

- **0–150 ms** — acceptable for most user applications; the delay is unnoticeable to a listener.
- **150–400 ms** — acceptable provided administrators are aware of the transmission time impact on the transmission quality of user applications.
- **>400 ms** — unacceptable for general network planning purposes, except in exceptional cases.

G.114 was written for traditional PSTN carriers and later revised for packet voice (VoIP). Two things to remember when quoting it in a voice-agent design:

1. **G.114 is one-way delay, speaker-to-listener.** A voice agent experiences this *twice* — the user's audio has to reach the server, and the agent's audio has to reach the user — but those two legs are not the same thing as the **processing time** inserted between them.
2. **G.114 assumes the listener is a human on a phone call, not an AI pipeline.** A human on the far end can carry a conversation at 300 ms one-way delay because speech perception is forgiving within those ranges. An AI pipeline sitting between the two humans *adds* its own processing time on top of G.114's one-way path.

Treat G.114 as the **transport budget** you inherit from the telephony/WebRTC path. Your real architectural question is how much processing time you can insert inside that budget before the user feels the lag.

## The target the user actually judges you on

The metric your users judge you on is not one-way delay. It is:

> **Turn gap (p95)** — from the moment the user stops speaking (end of utterance) to the moment the first byte of the agent's audio reaches their ear.

Industry folklore on this threshold varies — orchestration platforms, realtime API vendors, and conversational-AI product teams all publish targets in roughly the **600 ms to 1.5 s range**, with "under about 800 ms" treated as the point where an agent starts to feel conversationally natural rather than chatbot-slow. There is no ITU recommendation that pins this number down the way G.114 pins down one-way delay. In this module we use **p95 turn gap < 800 ms** as the default target and tag it as a product decision, not a standard.

Some notes on choosing a target:

- **Use p95, not mean.** A mean turn gap of 600 ms with a p95 of 2 s is a worse product than a mean of 750 ms with a p95 of 850 ms. The tail is what users remember.
- **Measure the gap at the user's ear, not at your server.** The last 100 ms of jitter buffer and playout on a mobile network is part of the gap even though you did not cause it.
- **Decide separately whether you will fill the gap.** Some products use a filler utterance ("let me check that for you…") to mask a longer tail. That is a product choice, not a replacement for the measurement.

## Decomposing the mouth-to-mouth path

When a user speaks an utterance, that single turn flows through a long chain of components. Each one contributes to the turn gap:

```
┌─────────────────────────────────────────────────────────────────┐
│                       USER SPEAKING PHASE                       │
├──────┬───────────────────┬─────────────────────┬────────────────┤
│ Mic  │ Device encoding   │ Network ingress     │ Media server   │
│ capt │ (Opus / G.711 /   │ (UDP/SRTP over      │ jitter buffer  │
│      │  AAC)             │  WebRTC or WS)      │ + decode       │
└──────┴───────────────────┴─────────────────────┴────────────────┘
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      USER UTTERANCE ENDS                        │
├────────────────────┬────────────────────┬───────────────────────┤
│ VAD endpoint       │ ASR finalisation   │ Agent orchestration   │
│ detection          │ (partial → final)  │ (prompt build, tools) │
└────────────────────┴────────────────────┴───────────────────────┘
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT RESPONSE PHASE                         │
├───────────────┬───────────────┬────────────────┬────────────────┤
│ LLM TTFT      │ LLM TPOT      │ TTS TTFB       │ TTS streaming  │
│ (first tok)   │ (subsequent)  │ (first audio)  │ (subsequent)   │
└───────────────┴───────────────┴────────────────┴────────────────┘
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                       AUDIO BACK TO USER                        │
├─────────────────────┬────────────────────┬──────────────────────┤
│ Network egress      │ Client decode +    │ User's ear           │
│ (SRTP / WS)         │ playout buffer     │                      │
└─────────────────────┴────────────────────┴──────────────────────┘
```

Three observations:

1. **Not every leg runs sequentially.** The second half of this chapter — the LLM and TTS legs — can and should overlap, which is the subject of Chapter 3.
2. **Not every leg is under your control.** Mic capture, device encoding, and client playout are on the user's device. Network ingress and egress depend on the user's carrier and your region choice. You can budget for them but not eliminate them.
3. **The "user utterance ends" moment is itself a decision.** VAD endpointing introduces a trade-off: wait longer and you miss fewer trailing words, wait less and the agent starts talking before the user is actually done. That boundary is covered in mod-106; here we assume VAD latency is a fixed component of the budget.

## A reference budget (p95 target: 800 ms)

A realistic allocation for a mid-market voice-agent pipeline using cascaded STT → LLM → TTS over a WebRTC transport, hosted in the same region as the user:

| Leg | p95 budget | Notes |
|---|---|---|
| Mic capture + device encode | ~20 ms | One audio frame at 20 ms packetisation; depends on client. |
| Network ingress (user → media server) | 20–60 ms | Same-region WebRTC; cellular can add tens of ms. |
| Jitter buffer + decode | 40–80 ms | Media server policy; trades packet loss recovery vs. latency. |
| VAD endpoint detection | 150–300 ms | Silence-based endpointing dominates; semantic turn detection cuts this. |
| ASR final transcript | 100–300 ms | From end-of-utterance to final text; depends on provider. |
| Orchestration overhead | 10–40 ms | Prompt assembly, retrieval if any, tool routing. |
| LLM TTFT | 150–400 ms | Highly provider- and model-dependent. |
| TTS TTFB | 100–250 ms | From first LLM token to first TTS audio byte (if streaming). |
| Network egress (server → user) | 20–60 ms | Symmetric to ingress. |
| Client decode + playout | 40–100 ms | Jitter buffer again, on the client side. |

Notes on reading this table:

- **The numbers are p95, not best-case.** A benchmark that cites "~150 ms LLM TTFT" is typically a p50; the p95 is often 2–3× that.
- **The legs do not all add up linearly in a well-designed pipeline.** Chapter 3 shows how TTS synthesis overlaps LLM generation, which collapses the LLM TPOT phase almost entirely out of the turn-gap measurement.
- **If this table does not fit under 800 ms once summed naively, that is expected.** The whole point of the module is that naive sequential execution does not hit a conversational target; parallelism does.

Keep this table around. You will revise it against your own measurements in Chapter 2 and again as acceptance criteria for the exercises.

## What "unacceptable" looks like

Three failure modes you should be able to recognise from the budget alone:

- **Dead air.** Turn gap >1.5 s with no filler. Users assume the line dropped and either repeat themselves or hang up.
- **Barge-in collisions.** User starts to speak again before the agent has finished playback, because the user thought the agent was still thinking. Fixed by cutting the gap, not by louder TTS.
- **Rushed overlap.** Opposite failure: VAD endpoint is too eager, agent interrupts the user's trailing words. Users escalate to "stop interrupting me" complaints. Covered in detail in mod-106.

A latency budget is not just an engineering goal. It's the input to a conversation-design decision.

## Summary

- G.114's 150/400 ms thresholds are the **one-way telephony delay** budget you inherit. Your voice agent adds its own processing time inside that budget.
- The product metric you actually commit to is **p95 turn gap** — user-utterance-end to agent-first-audio — with a default target of **under ~800 ms** for a conversational feel. That number is a product decision, not a standard.
- Decompose the path into mic/encode/transport/VAD/ASR/orchestration/LLM/TTS/transport/playout legs and give each a p95 allocation; expect the legs to overlap in a real pipeline (Chapter 3).
- A budget without measurement is marketing. Chapter 2 covers how to measure each leg.

Next chapter: measuring and attributing every millisecond with WebRTC stats and OpenTelemetry.
