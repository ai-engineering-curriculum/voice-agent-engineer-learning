# Resources for mod-102-speech-pipeline-foundations

Primary sources only. Cite these in your work; reach for secondary tutorials only when the primary source is unreadable.

## Standards and specifications

- **Opus audio codec** — [RFC 6716](https://datatracker.ietf.org/doc/html/rfc6716). Frame sizes, FEC, DTX, internal modes (SILK / CELT), bitrate ranges.
- **µ-law and A-law (G.711)** — [ITU-T Recommendation G.711](https://www.itu.int/rec/T-REC-G.711). Logarithmic 8-bit encoding of an 8 kHz speech signal.
- **G.722 wideband codec** — [ITU-T Recommendation G.722](https://www.itu.int/rec/T-REC-G.722). Sub-band ADPCM at 64 kbps with 7 kHz audio bandwidth.
- **SDP (Session Description Protocol)** — [RFC 8866](https://datatracker.ietf.org/doc/html/rfc8866). Negotiation format used by WebRTC and SIP.
- **RTP (Real-time Transport Protocol)** — [RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550). Payload types 0 (PCMU / µ-law) and 8 (PCMA / A-law) in [RFC 3551](https://datatracker.ietf.org/doc/html/rfc3551).
- **WebRTC 1.0** — [W3C WebRTC specification](https://www.w3.org/TR/webrtc/); [MDN WebRTC API overview](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API).
- **Nyquist–Shannon sampling theorem** — background: any undergraduate DSP textbook; [Wikipedia](https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem) is a serviceable starting point for the statement and its implications.

## Libraries and tooling

### Python

- **librosa** — [librosa.org](https://librosa.org/doc/latest/). Audio feature extraction, loading, resampling. Default resampler uses `soxr` when installed.
- **torchaudio** — [pytorch.org/audio](https://pytorch.org/audio/stable/). GPU-batchable transforms, codecs, datasets.
- **soundfile** — [python-soundfile.readthedocs.io](https://python-soundfile.readthedocs.io/). Thin bindings to libsndfile for WAV / FLAC / OGG I/O.
- **python-soxr** — [github.com/dofuuz/python-soxr](https://github.com/dofuuz/python-soxr). The SoX resampler; the quality reference for sample-rate conversion.
- **opuslib** — [github.com/orion-labs/opuslib](https://github.com/orion-labs/opuslib). Python bindings to libopus.
- **audioop** — [Python standard library](https://docs.python.org/3/library/audioop.html). µ-law / A-law conversion. (Note: `audioop` was removed from the Python standard library in version 3.13; see [PEP 594](https://peps.python.org/pep-0594/) and migrate to `audioop-lts` or an equivalent on newer Pythons.)
- **pyloudnorm** — [github.com/csteinmetz1/pyloudnorm](https://github.com/csteinmetz1/pyloudnorm). ITU-R BS.1770 loudness normalisation.

### Noise suppression and echo cancellation

- **WebRTC Audio Processing Module (APM)** — [source in Chromium](https://source.chromium.org/chromium/chromium/src/+/main:third_party/webrtc/modules/audio_processing/). Includes AEC3 (echo), NS (noise suppression), AGC (gain control).
- **RNNoise** — [jmvalin.ca/demo/rnnoise](https://jmvalin.ca/demo/rnnoise/); source [github.com/xiph/rnnoise](https://github.com/xiph/rnnoise). Paper: [Valin 2017, "A Hybrid DSP/Deep Learning Approach to Real-Time Full-Band Speech Enhancement"](https://arxiv.org/abs/1709.08243).
- **Speex DSP** — [github.com/xiph/speexdsp](https://github.com/xiph/speexdsp). Classic AEC / PLC / resampler library; still used in open-source media servers.
- **Krisp (commercial)** — [krisp.ai](https://krisp.ai/). Developer documentation at [docs.krisp.ai](https://docs.krisp.ai/). Includes background-voice cancellation.

## Neural audio codecs

- **EnCodec** — paper: [Défossez et al., 2022 "High Fidelity Neural Audio Compression"](https://arxiv.org/abs/2210.13438). Code: [github.com/facebookresearch/encodec](https://github.com/facebookresearch/encodec). Also available via [Hugging Face Transformers `EncodecModel`](https://huggingface.co/docs/transformers/model_doc/encodec).
- **SoundStream** — paper: [Zeghidour et al., 2021 "SoundStream: An End-to-End Neural Audio Codec"](https://arxiv.org/abs/2107.03312).
- **Mimi / Moshi** — paper: [Défossez et al., 2024 "Moshi: a speech-text foundation model for real-time dialogue"](https://arxiv.org/abs/2410.00037). Code and weights: [github.com/kyutai-labs/moshi](https://github.com/kyutai-labs/moshi).
- **Descript Audio Codec (DAC)** — [github.com/descriptinc/descript-audio-codec](https://github.com/descriptinc/descript-audio-codec); paper [Kumar et al., 2023](https://arxiv.org/abs/2306.06546). An alternative open-weights codec with strong reconstruction quality.

## Self-supervised speech representations (reading-list depth; not reimplement depth)

- **wav2vec 2.0** — [Baevski et al., 2020](https://arxiv.org/abs/2006.11477). Facebook AI Research.
- **HuBERT** — [Hsu et al., 2021](https://arxiv.org/abs/2106.07447). Facebook AI Research.
- **WavLM** — [Chen et al., 2021](https://arxiv.org/abs/2110.13900). Microsoft.
- **MMS (Massively Multilingual Speech)** — [Pratap et al., 2023](https://arxiv.org/abs/2305.13516); model card on Hugging Face at [facebook/mms-1b-all](https://huggingface.co/facebook/mms-1b-all).
- **Whisper** — [Radford et al., 2022](https://arxiv.org/abs/2212.04356); code: [github.com/openai/whisper](https://github.com/openai/whisper); fast inference: [github.com/SYSTRAN/faster-whisper](https://github.com/SYSTRAN/faster-whisper).

## Transport and orchestration framework docs touching the pipeline

- **LiveKit Agents** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). WebRTC transport; audio frame plumbing; STT/TTS plugin surface.
- **Pipecat** — [github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat); docs at [docs.pipecat.ai](https://docs.pipecat.ai/). Audio-frame-centric pipeline framework.
- **Twilio Media Streams** — [twilio.com/docs/voice/media-streams](https://www.twilio.com/docs/voice/media-streams). µ-law 8 kHz bidirectional WebSocket format.
- **Deepgram streaming API** — [developers.deepgram.com/docs/live-streaming-audio](https://developers.deepgram.com/docs/live-streaming-audio). Supported container / codec formats and chunking advice.
- **AssemblyAI streaming** — [www.assemblyai.com/docs/speech-to-text/universal-streaming](https://www.assemblyai.com/docs/speech-to-text/universal-streaming).

## Datasets useful for the exercises

- **LibriSpeech** — [openslr.org/12](https://www.openslr.org/12/). English read speech; the `dev-clean` subset is a standard clean-source choice.
- **DNS Challenge (Microsoft)** — [github.com/microsoft/DNS-Challenge](https://github.com/microsoft/DNS-Challenge). Noise and speech corpora with permissive licensing for noise-suppression research.
- **Common Voice** — [commonvoice.mozilla.org](https://commonvoice.mozilla.org/). Multilingual crowdsourced speech. Useful for diverse-speaker testing.
- **VCTK** — [datashare.ed.ac.uk/handle/10283/3443](https://datashare.ed.ac.uk/handle/10283/3443). Multi-speaker read speech; used in TTS research.

## Reference tracks on the ladder (for the boundary chapter)

- **nlp-engineer (L30)** — the AI-engineering-curriculum role that owns speech-model fine-tuning, representation extraction, and corpus-level quality work. <!-- needs-research: confirm the canonical nlp-engineer-learning repo URL within the ai-engineering-curriculum org at time of authorship -->
- **ml-engineer (L20)** — the broader ML-foundation role that owns training pipelines, loss design, distributed training. <!-- needs-research: confirm the canonical ml-engineer-learning repo URL within the ai-engineering-curriculum org at time of authorship -->

## Citation hygiene

- Vendor pricing / bitrate / latency pages change often. When a specific number matters (bitrate ceilings, chunk sizes, decoder latencies), cite the dated docs or the paper, and archive your evidence.
- For codec capability claims, the RFC or the ITU-T recommendation is authoritative. Vendor blog posts summarise; they do not define.
- WER / MOS numbers in marketing material are not peer-reviewed. Treat them as marketing until you reproduce them on your own corpus (which is what exercises 05 and 06 ask you to do).
