# exercise-03: Walk a Vocode Pipeline as the Minimal Compare Point

**Estimated effort:** 1 hour · **Output:** a running Vocode `StreamingConversation` plus a short "what did I give up / gain" note

## Objective

Build the smallest possible working voice agent on [Vocode](https://github.com/vocodedev/vocode-core) — four named components (input device, transcriber, agent, synthesizer, output device), one `StreamingConversation`, one process. Compare the developer experience and the resulting runtime to your Pipecat and LiveKit Agents builds. The goal is not to make Vocode "win"; the goal is to internalise what you gain by going minimal and what you give up versus the two heavyweights.

## Prerequisites

- [Chapter 3 — Vocode: The Minimal Compare Point](../03-vocode-minimal-compare-point.md).
- Completed exercise-01 and exercise-02 so you have Pipecat and LiveKit Agents builds to compare against.
- API keys for STT (Deepgram or similar), LLM (OpenAI), TTS (Cartesia / ElevenLabs).
- Python 3.11+.

## Steps

1. **Install Vocode core.** `pip install "vocode[all]"` (or just the extras you need). Check the [Vocode docs](https://docs.vocode.dev/) for the current package layout.

2. **Assemble a minimal local-microphone conversation.** Mic + speaker as input/output devices, Deepgram transcriber, `ChatGPTAgent` with a short voice-first system prompt, Cartesia or ElevenLabs synthesizer. Run it; have a two-minute conversation.

3. **Swap in a Twilio phone connector.** Reuse the same transcriber + agent + synthesizer; change only the input/output devices. Dial the agent from a phone. Confirm the conversation works over PSTN.

4. **Subclass `BaseAgent` with a non-LLM agent.** Replace `ChatGPTAgent` with a tiny custom agent that answers three or four scripted questions with hard-coded text. This forces you to use Vocode's extensibility point. Note how much code it takes vs. what the same thing would take on Pipecat / LiveKit.

5. **Inspect interrupt behaviour.** Interrupt the agent mid-sentence. Note how it feels compared to Pipecat / LiveKit. Where does it fall short? Where does it feel the same?

6. **Write a short compare note.** Half a page, direct. Cover:
   - DX (how long to a working conversation, in minutes, on each framework).
   - What you gave up versus Pipecat / LiveKit (integrations, transports, in-framework interrupt primitive, …).
   - What you gained (short mental loop, easy custom agent).
   - A one-sentence verdict on when you would reach for each.

## Starter guidance

- **This is a compare exercise, not a production build.** Keep the agent trivial; the point is the contrast.
- **Time yourself.** DX differences across these frameworks compress to "how many minutes to a working conversation". Record the number, honestly.
- **Don't instrument.** For this exercise, trust your subjective impression of the latency; you have real numbers from exercise-01 and exercise-02.
- **Pay attention to the `BaseAgent` subclassing step.** That's where Vocode shines — if your production agent is a bespoke engine, Vocode will out-DX the bigger frameworks.

## Acceptance criteria

- [ ] A working Vocode `StreamingConversation` on local mic/speaker.
- [ ] The same four components reused with a Twilio connector, driving a real phone call.
- [ ] A custom `BaseAgent` subclass that returns scripted responses, wired into the conversation.
- [ ] A subjective interrupt-behaviour note.
- [ ] A half-page compare note (DX, give-ups, gains, verdict), dated.

## Stretch goals

- **Mix a Vocode agent into Pipecat.** Pipecat has interoperability affordances for different service shapes; see whether you can wrap your Vocode custom agent as a Pipecat LLM-shaped processor. If you can, note what it took.
- **Try the Vocode hosted platform.** Create an account on Vocode's hosted offering (if available at your date) and ship the same agent there. Record the DX delta vs. running the library yourself.
- **Reach-for diagram.** Draw a decision tree: for a given incoming requirement (volume, control, in-house ML, operational team size), which of Vocode / Pipecat / LiveKit Agents / a managed platform do you reach for? Keep it short enough to fit on a slide.

## Keep it

The compare note is small but valuable — it is the paragraph you will cite in design reviews when someone asks "why not Vocode?". Keep it next to the exercise-01 and exercise-02 measurement notes.
