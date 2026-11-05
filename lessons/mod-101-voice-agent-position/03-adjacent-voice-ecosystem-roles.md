# 03. Adjacent Voice-Ecosystem Roles and Sharp Boundaries

## Why this chapter

Voice products are built by cross-functional groups where several disciplines overlap on the artefact but own very different things. The voice-agent engineer shares a build calendar with research scientists, conversation designers, and traditional telephony engineers. If you don't keep the boundaries sharp, two things go wrong at once: work gets duplicated, and the thing you are uniquely qualified to own gets dropped on the floor.

Use this chapter to recognise each adjacent role in the wild and to say out loud "that's theirs, not mine" (and the reverse).

## The map

```
                         ┌──────────────────────────────┐
                         │   voice-agent engineer       │
                         │       (L35, this role)       │
                         └──────────────────────────────┘
                                 │        │        │
         ┌───────────────────────┘        │        └─────────────────────────┐
         │                                │                                  │
  research side                    product / UX side                 operations side
         │                                │                                  │
  ASR research scientist           conversation designer             IVR / CTI developer
  TTS research scientist           voice-UX designer                 contact-centre ops eng
```

Three neighbourhoods. The role sits at the hinge.

## Research side

### ASR (speech-recognition) research scientist

**What they own.** Training objectives for acoustic + language models, dataset curation (LibriSpeech, Common Voice, in-house corpora), domain-adaptation fine-tunes, hard-negative mining, decoder improvements (RNN-T, CTC, attention), robustness to noise / accent / code-switching. They publish, they file internal model-release tickets, they write cards for new model versions.

**What you own.** Everything downstream of the model release: picking which model from which provider, wiring it into the pipeline, measuring **product-level** WER on your own calls, deciding whether a regression is launch-blocking.

**Sharp boundary.** If the question is "we need the ASR to handle paediatric cardiology terms better" — the methodology (fine-tune? LM biasing? custom vocabulary? in-context prompting?) is theirs. The *decision* of whether that work is worth starting, and the eval that proves the result, is a shared deliverable you drive.

### TTS research scientist

**What they own.** Acoustic models (Tacotron 2, FastSpeech, VITS, neural codec LMs like VALL-E, state-space models like Mamba-based audio, flow-matching TTS), vocoders (HiFi-GAN, BigVGAN), prosody modelling, voice cloning, emotion conditioning, cross-lingual transfer.

**What you own.** Voice selection, SSML / markup conventions, chunked streaming synthesis, end-of-utterance detection on the TTS side (so the agent knows when to accept barge-in), and all product-side tradeoffs between naturalness, latency, and cost.

**Sharp boundary.** "Sounds robotic" is a shared problem. "The vocoder is clipping at this phoneme boundary" is theirs. "The agent is speaking too fast because our prompt doesn't include pacing guidance" is yours.

## Product / UX side

### Conversation designer

**What they own.** The *dialogue* — personality, turn structure, recovery phrases, disclaimers, prompt templates, escalation paths, the specific wording of confirmations ("Just to confirm, you'd like to cancel this appointment, is that right?"). On regulated products they also own the compliant script that satisfies legal.

**What you own.** The *delivery mechanism* for that dialogue — how it gets rendered into audio at the right time, under the right latency, with the right voice, cleanly interruptible.

**Sharp boundary.** The engineer does not freelance the agent's personality; the designer does not freelance the barge-in threshold. Both collaborate on the contract between them: tagged prompt sections, markup conventions, voice-style tokens.

### Voice-UX designer

**What they own.** The multimodal UX — when to speak vs. when to show text on a paired screen, how to use earcons / non-speech audio, how to handle fallback to touch / DTMF / text chat, accessibility (hearing impairments, cognitive load, non-native speakers).

**What you own.** Making the UX decisions implementable — e.g. "we need a non-speech ‘thinking' audio cue if the LLM takes longer than 600 ms" is a design decision, and you own wiring it to a timer on the orchestration layer.

## Operations side

### Traditional IVR / CTI developer

**What they own.** Decades-old interactive voice response systems: DTMF menus, call routing, queue management, SIP / H.323, PBX / softswitch integration, CTI screen-pop for human agents. Many of them have deep carrier-grade reliability expertise (five-nines uptime, failover between data centres, number porting, STIR/SHAKEN caller-ID attestation).

**What you own.** The LLM-powered agent that an IVR flow now hands off to, and the hybrid behaviour in both directions ("if the user says ‘representative' or presses 0, hand off to the queue").

**Sharp boundary.** Carrier-grade telephony engineering is a different discipline, not a lesser one. You do not redesign the SIP trunking; you own the audio stream once it hits your pipeline. The handoff contract in both directions — call parameters, metadata, barge-out behaviour, DTMF passthrough — is what you write with them.

### Contact-centre operations engineer

**What they own.** The running contact centre — WFM (workforce management), ACD (automatic call distribution) tuning, SLAs, agent coaching, QA review, compliance recording (two-party consent states, PCI pause-and-resume).

**What you own.** The voice-agent product that either reduces that operations load (self-serve calls completed autonomously) or augments it (realtime assist / coaching). Your operations runbook (see Chapter 5) is what their oncall reads at 3 a.m.

## A single-row cheat sheet

| Role | Their centre of gravity | Your centre of gravity |
|---|---|---|
| ASR research scientist | Trains the ASR model | Chooses, measures, and integrates it |
| TTS research scientist | Trains the TTS model | Picks voices, streams synthesis, owns delivery latency |
| Conversation designer | The dialogue content | The dialogue delivery |
| Voice-UX designer | The multimodal interaction | The implementation of it |
| IVR / CTI developer | Carrier-grade telephony plumbing | The LLM agent on the far side of the handoff |
| Contact-centre ops engineer | Live call-centre operations | The voice-agent product in that call centre |

## Boundaries the track enforces

A voice-agent engineer in this curriculum is deliberately **not** the same as:

- An ML research scientist. You read papers, you understand architectures, you make informed model choices — you do not own the training objective or the dataset.
- A conversation designer. You care deeply about the dialogue, but you don't own its content.
- A telephony specialist. You bridge to telephony; you don't maintain the carrier side.

Where those boundaries are fuzzy in your organisation, be the person who writes them down.

## Summary

- Three adjacent neighbourhoods — research (ASR / TTS), product (conversation design, voice UX), operations (IVR / CTI, contact-centre ops) — surround the voice-agent engineer.
- Each has a crisp "they own X, you own Y" boundary that should be written into onboarding docs, not rediscovered by accident.
- The track enforces that this role is a full-stack voice-product engineer, not a research scientist, conversation designer, or telephony engineer.

Next chapter: the generalist stance — when you own full-stack voice, and when you escalate to a peer specialist.
