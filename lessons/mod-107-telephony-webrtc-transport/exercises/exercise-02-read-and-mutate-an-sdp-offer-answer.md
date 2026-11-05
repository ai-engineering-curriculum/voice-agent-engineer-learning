# exercise-02: Read and Mutate an SDP Offer / Answer

**Estimated effort:** 2 hours · **Output:** an annotated SDP + a parser-driven mutation script + a before/after capture of a renegotiation

## Objective

Make [Chapter 2](../02-sdp-offer-answer.md) concrete by **reading** a real SDP line by line, **mutating** it deliberately with a parser (not regex), and **observing** the effect on the peer. You will leave this exercise able to look at an unknown SDP and name every line in <60 seconds, and to make targeted Opus-parameter changes without breaking ICE or DTLS.

## Prerequisites

- [Chapter 2 — Reading and Mutating an SDP Offer / Answer](../02-sdp-offer-answer.md).
- The browser-client testbed from [exercise 01](exercise-01-ship-a-browser-webrtc-client-with-getusermedia-and-opus.md).
- Either `npm i sdp-transform` (Node / browser) or `pip install aiortc` (the aiortc SDP parser is sufficient). A pure-regex approach is explicitly out of scope for Part B.

## Steps

### Part A — Annotate a real SDP (30 minutes)

1. From exercise 01, capture the full SDP offer and answer your browser produced. Save as `offer.sdp` and `answer.sdp`.

2. For `offer.sdp`, **annotate every single line** with a one-sentence explanation of what it means. Use inline comments or a side-by-side table. Include:
   - The session-level lines (`v=`, `o=`, `s=`, `t=`, `a=group:BUNDLE`, `a=msid-semantic`).
   - The `m=audio` line: parse the preference list.
   - Every `a=rtpmap:` and `a=fmtp:` line.
   - The ICE lines (`a=ice-ufrag`, `a=ice-pwd`, `a=candidate`).
   - The DTLS lines (`a=fingerprint`, `a=setup`).
   - The BUNDLE / mid / direction lines.

3. For `answer.sdp`, do the **diff against the offer**. Which codecs survived? Which fmtp parameters survived? Which direction attribute did the peer choose?

### Part B — Mutate with a parser (45 minutes)

4. Write a small script (`mutate.js` or `mutate.py`) that:
   - Parses an SDP offer with `sdp-transform` or aiortc.
   - Takes a target Opus `fmtp` as a JSON config: `{ useinbandfec: 1, usedtx: 0, maxaveragebitrate: 24000, stereo: 0 }`.
   - Rewrites the Opus `a=fmtp:` line to that config (adding fields the original didn't have, replacing ones it did).
   - Serialises back to text.
   - Validates by re-parsing and checking the fmtp is as requested.

5. Point your exercise-01 browser client at a modified `createOffer()` flow:
   - Call `createOffer()`.
   - Pass the SDP to your script's mutation function.
   - Pass the result back to `setLocalDescription()`.
   - Send the mutated offer on the signalling channel.

6. Verify at the server peer that the mutated `fmtp` line was received unchanged. Capture `pc.remoteDescription.sdp` on the server; grep for `fmtp`; confirm the config you sent.

### Part C — Observe the effect (30 minutes)

7. Pick two config variants and run a 60-second call for each:
   - **Variant A: `maxaveragebitrate=24000, useinbandfec=1`** — low-bitrate, loss-resilient.
   - **Variant B: `maxaveragebitrate=96000, useinbandfec=0`** — high-bitrate, no FEC.

8. For each variant, record from `pc.getStats()`:
   - `outbound-rtp.bytesSent` growth over the window → average bitrate in kbps.
   - `outbound-rtp.targetBitrate` (if exposed).
   - `inbound-rtp.jitter` on the receive side.

9. Compare: did the encoder actually run near the requested ceiling? If not, why (congestion control, VBR)?

### Part D — Direction attribute negotiation (15 minutes)

10. Modify your client to offer `sendonly` for the audio transceiver. Observe what direction the server's answer picks. Then do the reverse: offer `recvonly`. Then `inactive`.

11. Record, for each of the four direction attributes (`sendrecv`, `sendonly`, `recvonly`, `inactive`):
    - What direction the answer picks.
    - Whether `ontrack` fires on each side.
    - Whether audio actually flows.

## Starter guidance

- `sdp-transform` returns a structured object with `media[].fmtp[]` as `{ payload, config }` where `config` is the string. Parse it into a dict, mutate, serialise with `.map(f => \`${f.payload} ${f.config}\`).join(';')`.
- The server-side SDP after renegotiation is at `pc.remoteDescription.sdp` on your server peer. Save it to a file for diffing.
- Chrome's `webrtc-internals` logs the full offer and answer; use that as the ground-truth tap even when your signalling layer is opaque.
- `maxaveragebitrate` is a **ceiling**, not a target. Variable bitrate (VBR) is default; your observed bitrate may run below the ceiling even for loud speech.
- The direction attribute mirrors across the offer/answer boundary (offer `sendonly` → answer `recvonly`, not `sendonly`). Mis-mirroring is one of the two most common SDP bugs.

## Acceptance criteria

- [ ] Annotated `offer.sdp` with one-line explanation per line.
- [ ] Offer/answer diff: which codecs survived, which fmtp params survived, which direction the peer chose.
- [ ] `mutate.js` or `mutate.py` script, using a parser (not regex). Readable, one function, <150 lines.
- [ ] Two variants (low-bitrate + FEC, high-bitrate no FEC) each run for 60 s; measured bytesSent / bitrate table captured; discussed whether the requested bitrate was actually hit.
- [ ] Four-row table for direction attributes: offered direction, answered direction, `ontrack` behaviour, audio flow outcome.
- [ ] Short reflection: name three things you should *not* mutate (fingerprint, ice-*, BUNDLE) and why each would break the session.

## Stretch goals

- **Reorder the codec preference list.** The browser offers `opus, isac, pcmu, pcma` in that order; rewrite to `pcmu, pcma, opus` and observe what the server picks. Does your server-side peer actually drop to G.711?
- **Strip all non-Opus codecs from the offer.** Rewrite the `m=audio` line and the matching `a=rtpmap:` / `a=fmtp:` lines. Confirm the peer can still negotiate.
- **Add a header extension.** Introduce the `a=extmap:` line for the audio-level header extension ([RFC 6464](https://datatracker.ietf.org/doc/html/rfc6464)) if it isn't already present. Verify `inbound-rtp.audioLevel` starts populating in `getStats()`.
- **Compare to `setCodecPreferences()`.** Do the same codec-pinning via `RTCRtpTransceiver.setCodecPreferences()` instead of SDP mutation. Diff the resulting offer SDP — is it identical?
- **Break it on purpose.** Rewrite one byte of `a=fingerprint:` and send the mutated offer. What error does the peer surface? (Expected: DTLS handshake failure.)

## Keep it

The mutation script is a diagnostic tool, not a production pattern — but it will save you an evening the next time a browser upgrade removes a fmtp field you relied on. Keep it in the toolbox alongside your `chrome://webrtc-internals` dumps from exercise 01.
