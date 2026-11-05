# 04. SIP for Traditional Telephony Endpoints

## Why this chapter

If your voice agent only ever talks to browsers and mobile apps, you can skip this chapter. The moment it answers a phone call — any phone call, anywhere — SIP is in the stack, even if a provider hides it behind a WebSocket or a REST API. When a Twilio call fails, when a Vonage trunk silently stops delivering INVITEs, when your Asterisk box drops the leg, SIP trace logs are the ground truth.

This chapter is enough SIP to debug a trunk. Writing a full SIP stack is `nlp-engineer`-adjacent; owning one in production isn't.

## The one-line description

SIP ([RFC 3261](https://datatracker.ietf.org/doc/html/rfc3261)) is HTTP-shaped signalling for setting up, modifying, and tearing down real-time multimedia sessions. The actual audio travels over RTP; SIP is just the handshake and the control plane.

Mental model: **SIP is to RTP what HTTP is to a video file**. SIP says "please establish a call with these parameters"; RTP carries the audio once the call is up.

## The actors

- **User Agent Client (UAC)** — the endpoint initiating a request (the caller).
- **User Agent Server (UAS)** — the endpoint receiving a request (the callee).
- **Proxy** — forwards SIP messages between UAs. Does not store state beyond a transaction.
- **Registrar** — accepts `REGISTER` messages that bind a SIP URI (`sip:alice@example.com`) to an IP:port where the UA is reachable.
- **Back-to-Back User Agent (B2BUA)** — looks like two user agents glued together; Twilio, carriers, and most SBCs are B2BUAs.
- **Session Border Controller (SBC)** — the security / routing device that sits at the edge of a network, almost always a B2BUA in practice.
- **Soft-switch / media gateway** — the device that terminates SIP and bridges to another protocol (SS7, WebRTC, another SIP trunk with different auth).

For a voice agent, the usual diagram is:

```
PSTN caller → carrier → SIP trunk → SBC → your soft-switch or media server → agent
                                                 (SIP↔WebRTC bridge)
```

## The INVITE / 180 / 200 / ACK / BYE dance

The canonical call-setup flow:

```
UAC                                              UAS
 |                                                |
 |-- INVITE sip:bot@example.com --------------->  |   "Please call this URI with this SDP."
 |                                                |
 |  <-- 100 Trying ----------------------------   |   "Got it, working on it."
 |  <-- 180 Ringing ---------------------------   |   "The phone is ringing."
 |  <-- 200 OK (SDP answer) -------------------   |   "Answered. Here's my SDP."
 |                                                |
 |-- ACK ---------------------------------------> |   "Confirmed, let's talk."
 |                                                |
 |====== RTP audio flows in both directions =====|
 |                                                |
 |-- BYE --------------------------------------> |   "Hanging up."
 |  <-- 200 OK -------------------------------   |
```

Four observations worth internalising:

- **The SDP offer/answer travels inside SIP.** `INVITE` carries the offer in its body; `200 OK` carries the answer. This is the same SDP you read in Chapter 2, same `m=audio`, same `a=rtpmap` — SIP is just a different envelope than WebRTC's JavaScript API.
- **Provisional responses (1xx) don't commit the session.** `100 Trying` and `180 Ringing` mean "still working." Only `200 OK` seals the deal.
- **`ACK` is a separate request, not a response.** It confirms receipt of the final `200`. A missing `ACK` is one of the two most common "the call rings but silence follows" bugs.
- **`BYE` is also a request** that gets its own `200 OK`. Call-tear-down is symmetric; either side can initiate.

## Status codes you'll actually see

SIP responses are grouped like HTTP responses:

| Class | Meaning | Examples |
|---|---|---|
| 1xx Provisional | Still working | 100 Trying, 180 Ringing, 183 Session Progress |
| 2xx Success | Done | 200 OK |
| 3xx Redirection | Try somewhere else | 302 Moved Temporarily |
| 4xx Client error | Request is bad | 401 Unauthorized, 403 Forbidden, 404 Not Found, 407 Proxy Authentication Required, 408 Request Timeout, 486 Busy Here, 487 Request Terminated |
| 5xx Server error | Server failed | 500 Server Internal Error, 503 Service Unavailable |
| 6xx Global failure | No endpoint will take this | 603 Decline, 604 Does Not Exist Anywhere |

The ones that are specifically telephony-shaped and worth memorising:

- **401 Unauthorized** — the registrar wants digest credentials. First-INVITE auth loop: INVITE → 401 → INVITE with `Authorization` header → 200. Normal.
- **407 Proxy Authentication Required** — same idea but from the proxy, not the endpoint.
- **403 Forbidden** — the carrier won't let you dial that number (international restrictions, blacklisted prefix, your account doesn't have outbound permission).
- **486 Busy Here** — endpoint is on another call. Voicemail systems return this to force overflow.
- **487 Request Terminated** — the caller hung up before the callee answered. Your SIP-side INVITE was CANCELed.
- **488 Not Acceptable Here** — codec negotiation failed. The offered codecs don't overlap with the callee's. You offered Opus-only to a G.711-only trunk, say.
- **503 Service Unavailable** — the carrier is overloaded or your trunk is down. Retries belong to the originating side with backoff.

## The header fields that matter for debugging

A real INVITE looks like:

```
INVITE sip:+15551234567@carrier.example.com SIP:2.0
Via: SIP/2.0/UDP 203.0.113.10:5060;branch=z9hG4bK-abc123
Via: SIP/2.0/UDP 198.51.100.5:5060;branch=z9hG4bK-def456
Max-Forwards: 69
From: "AgentBot" <sip:bot@your.pbx.example.com>;tag=1234
To: <sip:+15551234567@carrier.example.com>
Call-ID: 55555-aaaa-bbbb@your.pbx.example.com
CSeq: 1 INVITE
Contact: <sip:bot@your.pbx.example.com:5060>
Content-Type: application/sdp
Content-Length: 310

v=0
o=- 0 0 IN IP4 198.51.100.5
...
```

- **`Via`** — the route the message took. Each proxy prepends its own `Via`; the response retraces the stack in reverse. **If the response comes back with fewer or different `Via` headers than you sent, you have a routing loop or a dropped proxy.** This is how you trace a stuck call.
- **`From` / `To`** — logical endpoints, with per-dialog `tag=` parameters. Both tags together identify the dialog uniquely.
- **`Call-ID`** — the stable identifier for this call across retries and renegotiations. Always include `Call-ID` in your observability logs so a single call's traces are easy to gather.
- **`CSeq`** — monotonic command sequence; the registrar and the server use it to detect retransmits.
- **`Contact`** — the direct address at which the sender is reachable, bypassing the proxies. Used for in-dialog requests.
- **`Max-Forwards`** — hop count; proxies decrement it. 0 → response `483 Too Many Hops`.
- **`Route` / `Record-Route`** — the loose-routing mechanism. A proxy that wants to stay in the signalling path inserts a `Record-Route`; subsequent requests use `Route` headers pointing at those proxies. If your in-dialog BYE goes the wrong direction, `Record-Route` ordering is where you look.

## Transport: UDP, TCP, TLS, WebSocket

SIP runs over four transports, selected by the URI scheme and the port:

- **UDP/5060** — original transport. Fire-and-forget; SIP itself handles retransmits. Packet-size limited.
- **TCP/5060** — reliable; used when signalling exceeds the UDP MTU (long SDP bodies, lots of codecs offered).
- **TLS/5061** (also written `sips:`) — TCP + TLS; used on open internet between SBCs.
- **WS/WSS** — SIP over WebSocket, specified in [RFC 7118](https://datatracker.ietf.org/doc/html/rfc7118). Used by JsSIP, SIP.js, and in-browser SIP clients.

The practical implication for a voice agent: your SBC-facing side almost always speaks TLS; your internal side can be TCP or UDP within a trusted network; browser clients talk WSS.

## Authentication

SIP authentication is HTTP Digest ([RFC 7616](https://datatracker.ietf.org/doc/html/rfc7616), updates the original SIP-Digest reference). The flow:

1. UA sends `REGISTER` or `INVITE` with no credentials.
2. Registrar/proxy returns `401 Unauthorized` (or `407` for a proxy) with a `WWW-Authenticate` challenge: `nonce`, `realm`, `algorithm=MD5` (or SHA-256 in newer deployments).
3. UA computes the response hash and retries with an `Authorization` header.
4. On success, the registrar returns `200 OK`.

Nothing here is high-security by modern standards — MD5 is still the default on many trunks, nonces are often long-lived. That's why TLS on the signalling transport is table stakes and why most providers additionally verify by IP ACL.

## Registration

`REGISTER` binds a logical SIP URI to the current IP:port of a UA, so that calls to `sip:alice@example.com` can reach whichever device Alice is currently using.

```
REGISTER sip:registrar.example.com SIP:2.0
From: sip:alice@example.com
To: sip:alice@example.com
Contact: sip:alice@198.51.100.5:5061;transport=TLS
Expires: 3600
```

The registrar records the binding with the given TTL; the UA re-registers before expiry. "SIP phone stopped working at 3 AM" is almost always "registration expired and re-registration failed."

For a voice agent, you usually **don't** register — the agent's trunk is static IP-authenticated and configured once in the provider dashboard. You will see REGISTER traffic if your agent impersonates a SIP user, or if you host a PBX.

## SIP ↔ WebRTC: the bridge

The two sides don't speak the same codec, the same transport, or the same security. Something has to translate.

| Layer | SIP | WebRTC |
|---|---|---|
| Transport | UDP/TCP/TLS 5060/5061 | UDP ICE/DTLS/SRTP |
| Signalling | SIP INVITE/200/ACK/BYE | SDP over WebSocket, no retransmits |
| Codec | G.711 / G.722 / sometimes Opus | Opus mandatory; others optional |
| Security | TLS on signalling, SRTP optional | SRTP always; DTLS-keyed |
| NAT traversal | Via headers; sometimes ICE/STUN | ICE/STUN/TURN mandatory |

The bridging device is either:

- **A soft-switch** (Asterisk, FreeSWITCH, Kamailio) with WebRTC modules. Terminates SIP on the carrier side, exposes WebRTC on the browser side, transcodes codecs in the middle.
- **A media server with native SIP** (LiveKit SIP, Daily SIP ingress, Janus SIP plugin). The SIP leg becomes a participant in the room.
- **A provider abstraction** (Twilio ConversationRelay, Vonage Voice API, Amazon Connect). The provider owns the bridge; you consume WebSockets or HTTP APIs.

A voice-agent team almost always builds against the provider abstraction or the media-server-native SIP; building the soft-switch yourself is a significant op burden you take on only if you need carrier-grade control or you are the carrier.

## Debugging — the `sngrep` loop

You cannot reason about a stuck SIP call without a trace. The standard tools:

- **[sngrep](https://github.com/irontec/sngrep)** — ncurses ladder diagrams of SIP messages. Install on your SBC/soft-switch; `sngrep -d eth0 port 5060` and you see every message.
- **[pjsip's `pjsua`](https://docs.pjsip.org/en/latest/pjsua2/intro.html)** — reference-implementation CLI; useful for sending test INVITEs.
- **[Wireshark](https://www.wireshark.org/)** — full packet capture with SIP dissector. More detail than sngrep, less ergonomic.
- **Provider-side logs.** Twilio's [Call Resource](https://www.twilio.com/docs/voice/api/call-resource) exposes SIP responses and [Voice Insights](https://www.twilio.com/docs/voice/insights) gives call-quality traces. Vonage, Plivo, Sinch have equivalents.
- **FreeSWITCH / Asterisk full SIP tracing** — `sofia global siptrace on` (FreeSWITCH), `sip set debug on` (Asterisk).

The debugging checklist:

1. Did the INVITE arrive? If no — DNS, firewall, IP ACL, TLS. If yes — step 2.
2. Did the receiver respond with 1xx or 4xx+? If 4xx+ — status code names the problem.
3. Did the 200 OK come back? If no — SDP mismatch (488), ringing-but-no-answer (487), authentication failure (401/407).
4. Did ACK get sent? Check for `ACK` in sngrep.
5. Did RTP start flowing? Check for RTP packets on the negotiated ports. One-way RTP is almost always an NAT/firewall/SDP-ip-line mismatch.
6. Did BYE land correctly? Look at `Via` and `Record-Route`.

## Summary

- SIP is HTTP-shaped signalling for RTP sessions. INVITE / 180 / 200 / ACK / BYE is the whole call.
- The SDP you write for WebRTC is the same SDP that travels inside INVITE and 200 OK.
- `Via`, `Record-Route`, `From`-tag / `To`-tag, and `Call-ID` are the fields you debug with.
- 401/407 digest-auth loops are normal; 488 is codec mismatch; 487 is caller-hung-up-first.
- The SIP↔WebRTC bridge is either a soft-switch you own, a media server with native SIP, or a provider abstraction. Pick based on how much telephony ops you want.

Next chapter: PSTN providers — Twilio, Vonage, Amazon Connect, Azure Communication, Plivo, Sinch, Zoom Phone — and how to pick one.
