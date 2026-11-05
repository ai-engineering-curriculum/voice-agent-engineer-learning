# 06. Anti-Spoofing, Deepfake Detection, and Audio Provenance

## Why this chapter

Chapters 1–5 made the compliance envelope explicit. This chapter is about the two attack surfaces that sit *inside* that envelope and are specific to voice:

- **Inbound** — someone on the other side of the call is not who they claim to be. Replay, voice cloning, synthetic-voice impersonation of an executive, playback of pre-recorded card-authorisation audio to defeat a voice-biometric check.
- **Outbound** — the recipient of your call needs to be able to tell that this voice is in fact AI-generated, and ideally that it came from your organisation specifically. Article 50 of the EU AI Act (Chapter 3) demands this; a growing set of US state bills does too.

The toolchain splits along the same line:

- **Inbound defence** = speaker verification + spoof countermeasures + deepfake detection + call-origination checks.
- **Outbound provenance** = watermarking the TTS output + binding a C2PA Content Credential manifest to it.

Both deploy at the media layer, below the LLM; neither is cured by a prompt. This chapter gives the engineer-shaped working depth on what to run and where to chain it in.

## Inbound — the attack taxonomy

The ASVspoof community's canonical taxonomy ([asvspoof.org](https://www.asvspoof.org/)) splits spoofing attacks against voice-biometric / caller-identity systems into three families:

- **Logical Access (LA)** — synthesised or voice-converted speech injected at the signal layer: TTS, voice conversion (VC), and now neural-codec-based voice clones (OpenAI Voice Engine, ElevenLabs Instant Voice Cloning, Microsoft VALL-E, F5-TTS and similar).
- **Physical Access (PA)** — replay attacks where recorded audio of the legitimate user is played back into the microphone. Classic room-impulse-response signatures.
- **DeepFake (DF)** — the ASVspoof 2021 / 2024 formulation targeting high-quality deepfake audio that is not necessarily aimed at a biometric system (e.g., media-published synthetic audio).

ASVspoof 5 (announced 2024, Interspeech Challenge) includes increasingly realistic end-to-end TTS and uses the 2024 adversarial threat model. The datasets to benchmark against:

- **ASVspoof 2019 LA / PA** — original baseline.
- **ASVspoof 2021** — adds the DF track and codec / channel variability.
- **ASVspoof 5 / 2024** — adversarial, including unseen TTS systems.

See the ASVspoof overview paper (Wang et al., 2024, [arXiv:2408.08739](https://arxiv.org/abs/2408.08739)) for the current methodology. <!-- needs-research: confirm specific challenge dataset year in use at the time of training a countermeasure -->

## Speaker verification baseline

A speaker-verification model trained on [VoxCeleb](https://www.robots.ox.ac.uk/~vgg/data/voxceleb/) (Nagrani et al., the standard open-source training corpus) emits a per-utterance embedding, typically 192- or 256-dimensional. Verification = cosine similarity against the enrolled embedding. Open implementations to start from:

- **SpeechBrain** — ECAPA-TDNN pretrained model, Apache-2 licensed. [speechbrain.github.io](https://speechbrain.github.io/). The canonical open baseline; usable as a starting point.
- **WeSpeaker** — [github.com/wenet-e2e/wespeaker](https://github.com/wenet-e2e/wespeaker). Research-grade speaker verification toolkit.
- **NVIDIA NeMo SV** — the TitaNet family. Enterprise-licence-friendly.

Equal Error Rate (EER) on VoxCeleb1-test for a modern ECAPA-TDNN is in the low-single-digit percent range. That is *not* strong enough to be a sole authentication factor; voice-auth products pair verification with knowledge-based factors or OTP for exactly this reason.

## Spoof countermeasures

A speaker-verification model trained only on bonafide speech is trivially defeated by a TTS or VC clone of the enrolled voice — the system scores the fake as *more* similar than a genuine recording because TTS is less noisy. The countermeasure (CM) model is a *separate* classifier trained to distinguish bonafide speech from spoofed speech.

Baseline CMs in the ASVspoof canon:

- **AASIST** (Jung et al., 2022, ICASSP) — graph-attention on raw waveforms. Open-source baseline.
- **RawNet2 / RawNet3** — convolutional end-to-end on raw audio.
- **ResNet-SE** family — spectrogram-domain CNN.

Deploy pattern:

```
inbound audio
     ↓
  CM model → bonafide_score
     ↓
  SV model → speaker_score
     ↓
  fusion → accept / challenge / reject
```

Fusion is usually a linear combination of the two scores with a learned weight. Don't collapse to a single model: a spoof detector that was never trained on genuine speaker verification will mis-score silence, codec artefacts, and legitimate accented speakers.

## Deepfake detection products

For the deeper deepfake-detection problem (audio that is not necessarily targeted at a biometric system — e.g., a customer sending in a "voicemail from the CEO" asking for a wire transfer), commercial products exist:

- **Pindrop Pulse / Pindrop Protect** — call-authentication + deepfake-detection platform, integrated into contact-centre stacks. [pindrop.com](https://www.pindrop.com/).
- **Resemble Detect** — Resemble AI's standalone deepfake-detection API. [resemble.ai/deepfake-detection](https://www.resemble.ai/deepfake-detection/).
- **ID R&D IDVoice / IDLive Face** — voice-biometric + anti-spoof. [idrnd.ai](https://www.idrnd.ai/).
- **Microsoft Azure Video Indexer deepfake detection** — audio support included in the broader toolkit.

Treat these as detection layers, not oracles. Current research consistently shows detection accuracy degrades against *unseen* TTS systems; the arms race is live. See the [Deepfake-Eval 2024](https://arxiv.org/abs/2403.12775) paper and the [In-the-Wild](https://arxiv.org/abs/2203.16263) benchmark for the kind of domain shift that matters. <!-- needs-research: cite the most recent cross-dataset benchmark at the time of deployment -->

## Outbound — audio watermarking

A watermark is a signal hidden *in* the audio waveform that a detector can recover with high probability, even after codec compression, noise, or partial re-synthesis. The design axes:

- **Imperceptibility** — the watermark must not be audible to a human listener. Typical target: PESQ score degradation < 0.1.
- **Robustness** — survival through telephony codecs (G.711, Opus), noise injection, time-stretching, re-recording.
- **Localization** — ability to say *which segment* of the audio carries the watermark. Matters when a legitimate TTS clip is spliced into a longer attack recording.
- **Security** — an attacker without the key cannot forge or remove the watermark.

Current open implementations:

- **Meta AudioSeal** ([github.com/facebookresearch/audioseal](https://github.com/facebookresearch/audioseal), MIT license) — Proactive Detection of Voice Cloning with Localized Watermarking, San Roman et al., ICML 2024 ([arXiv:2401.17264](https://arxiv.org/abs/2401.17264)). Sample-level localization, ~100ms resolution. Reported robust to common audio edits and codec passage.
- **SilentCipher** ([github.com/sony/silentcipher](https://github.com/sony/silentcipher), from Sony AI) — perceptual-domain watermark with a longer payload, intended for general audio provenance.
- **WavMark** (open research) — earlier open watermark, superseded in robustness by AudioSeal.

Deployment pattern for the TTS pipeline:

```
LLM → TTS → PCM audio → AudioSeal encoder (inject watermark) → encoder output → mix, codec, send
```

At the recipient side (or at a monitoring point), any audio can be run through the AudioSeal decoder to recover a per-chunk "is this watermarked?" score and the embedded payload.

The watermark carries a payload — typically 16–32 bits — into which you can encode a provider identifier plus a sequence number. That payload is the *machine-readable* detection surface Article 50 contemplates. A regulator or downstream service can derive "this audio was generated by ExampleCo on date X with model Y" from the watermark alone, without needing to touch a manifest.

## C2PA Content Credentials

Watermarking alone proves the audio *is* AI-generated by some signer. For the full provenance story — which model, which prompt lineage, which user — you need a cryptographically-signed manifest bound to the audio. That is the **Coalition for Content Provenance and Authenticity (C2PA)** standard.

The C2PA technical specification ([c2pa.org/specifications](https://c2pa.org/specifications/)) version 2.x defines:

- **Assertions** — individual claims about an asset (creator, creation time, AI model used, prior lineage).
- **Claim signer** — the party making a cryptographically-signed set of assertions.
- **Manifest** — a signed collection of assertions + the hash of the asset.
- **Binding** — the mechanism that attaches the manifest to the asset. For audio, common bindings are container-embedded (WAV `c2pa` chunk; MP3 `GEOB` ID3 frame; MP4 `uuid` box) or sidecar (separate `.c2pa` file).

The AI-specific assertions matter most for voice:

- `c2pa.actions` with `c2pa.created` — the audio was created by this actor.
- `c2pa.training-mining` — if the model was trained on content with specific permissions.
- `c2pa.ai_models` — the specific model and version used.
- `c2pa.ingredients` — any inputs that fed into the generation (reference audio, prompt text if disclosed).

Open-source tooling:

- **c2patool** ([github.com/contentauth/c2patool](https://github.com/contentauth/c2patool), Apache-2) — reference CLI; generates and verifies manifests.
- **Content Authenticity Initiative SDK** ([contentauthenticity.org](https://contentauthenticity.org/)) — client libraries.
- **OpenAI DALL·E and Sora** already ship C2PA metadata for images and video; similar for Google, Adobe, Microsoft. Expect audio support from major TTS vendors to grow in 2025–2026.

### Chaining C2PA to watermark

The two are complementary, not competitive:

- **Watermark** is embedded in the audio signal; survives codec and partial re-encoding; carries a small identifier.
- **C2PA manifest** is a container-level signed object; carries full provenance; is removed by naive re-encoding.

Pattern: embed an AudioSeal watermark carrying a *reference* (e.g., a hash prefix) that resolves to the C2PA manifest. If the recipient has the full container, they get full provenance. If they only have the stripped audio signal, they still get a cryptographic hint that lets them look up the manifest server-side.

This pattern was proposed in the AudioSeal paper (§ "Attribution"); the chain is live in several vendor deployments.

## Inbound verification path — chaining it all

For a voice agent's inbound call path:

```
caller → SIP ingress → ASR / STT
                  ↓
              anti-spoof CM (AASIST) → bonafide_score
                  ↓
              speaker verification (ECAPA-TDNN) → speaker_score (if enrolled)
                  ↓
              deepfake-detection (optional commercial layer) → deepfake_score
                  ↓
              carrier STIR/SHAKEN attestation from SIP → attestation_class
                  ↓
              fusion → risk_score → policy: accept / challenge / reject
```

At the policy layer:

- `risk_score > T_hard` → reject the call; log; return a generic "we cannot verify your identity" message.
- `T_soft < risk_score ≤ T_hard` → step up: knowledge-based auth, OTP to phone on file, "please call back from the number on file."
- `risk_score ≤ T_soft` → proceed.

Set `T_soft` and `T_hard` from your false-acceptance / false-rejection curves on a *representative* sample of your traffic, not on the research dataset.

## Outbound provenance path — chaining it all

For the agent's outbound audio:

```
LLM → TTS → PCM@48kHz
              ↓
       AudioSeal encode (payload = agent_id || sequence_number || timestamp_hash)
              ↓
       MP3 / Opus encode
              ↓
       C2PA manifest (signed) attached to container (if container preserved)
              ↓
       send on SIP/WebRTC
              ↓
       log: provenance_payload, manifest_hash, call_sid
```

Preserve the provenance log for the same retention window as the recording. If a recipient later disputes whether a given audio clip came from your system, you reproduce the payload + manifest from the log.

## Costs and gotchas

- **Codec passage** — AudioSeal survives Opus 24+ kbps and G.711 reasonably well; higher loss on 4 kbps codec extremes. Benchmark on the actual codecs your stack negotiates (see mod-107, Chapter 2).
- **TTS pre-processing** — any post-TTS filter (noise gate, loudness normalisation, echo cancellation at the far end) can degrade the watermark. Insert the watermark *after* all deterministic TTS processing but *before* lossy transport.
- **Legitimate echo-cancel re-recording** — a legitimate recipient recording your agent's output then replaying it does not preserve C2PA (container is lost) but should still preserve AudioSeal (signal-domain).
- **False positives on legitimate effects** — AudioSeal detectors return 0/1 per-chunk scores; expect a small false-positive rate on music or voice-overlapping effects. Tune thresholds.
- **Private keys** — the C2PA claim signer uses a certificate; manage the key as you would any production signing key. Rotation and HSM-backing are tablestakes.
- **Model licenses** — AudioSeal (MIT) and SilentCipher are permissively licensed; verify commercial terms for any proprietary detector (Resemble Detect, Pindrop Pulse).

## Summary

- Inbound: ASVspoof-shaped CM + VoxCeleb-trained SV + commercial deepfake detector + STIR/SHAKEN attestation → fused risk score → tiered policy.
- Outbound: AudioSeal watermark on the signal + C2PA manifest on the container, with a payload that cross-links the two.
- Regulatory mapping: inbound defence satisfies the authentication side of GDPR Article 32 and HIPAA 164.312(d). Outbound provenance satisfies EU AI Act Article 50(2) and matching state AI-disclosure bills.
- Private-key management, codec-aware tuning, and false-positive thresholds are the operational long tail; without them, the watermark is marketing.

Next chapter: the adversarial threat model — OWASP LLM Top 10, OWASP Agentic Top 10, MITRE ATLAS — mapped to the voice channel.
