# exercise-04: Debug a SIP Trunk with Trace Logs

**Estimated effort:** 2 hours · **Output:** a filled SIP-flow table + a diagnosis write-up for three failure traces + one live-captured sngrep trace of your own call

## Objective

Translate [Chapter 4](../04-sip-for-telephony.md) into a debugging workflow by (a) reading a working INVITE / 200 / ACK / BYE trace line by line, (b) diagnosing three broken traces from a provided set, and (c) capturing a real SIP trace of your own and explaining every message in it.

## Prerequisites

- [Chapter 4 — SIP for Traditional Telephony Endpoints](../04-sip-for-telephony.md).
- Access to a SIP endpoint. Options, in increasing order of difficulty:
  - Install Asterisk or FreeSWITCH locally and dial into it from a softphone (Linphone, Zoiper, MicroSIP). This is the recommended path.
  - Register a free SIP trunk (`sip.antisip.com`, `sip2sip.info`, `iptel.org`) and place a test call to a service like `*43` (Asterisk's echo test) or to the trunk's test number.
  - Use a Twilio Elastic SIP Trunk + a softphone — pay a few cents for test minutes.
- `sngrep`, `pjsua` (from pjsip), and Wireshark on the box that terminates SIP. `tcpdump` is a sufficient fallback.

## Steps

### Part A — Annotate a working trace (30 minutes)

A working call produces, at minimum, the following messages. Below is an abridged real trace (adapted; your names and numbers will differ):

```
#1   caller -> callee     INVITE sip:+15552223333@trunk.example.com
#2   callee -> caller     100 Trying
#3   callee -> caller     180 Ringing
#4   callee -> caller     200 OK (with SDP answer)
#5   caller -> callee     ACK
#6-#N RTP packets in both directions
#N+1 caller -> callee     BYE
#N+2 callee -> caller     200 OK
```

1. For each message type (`INVITE`, `100`, `180`, `200`, `ACK`, `BYE`, `200 OK (to BYE)`), fill a table:

   | Message | From-tag | To-tag | Call-ID | CSeq | Expected content-type | Expected body |

2. For the `INVITE`'s SDP body, re-read it with the Chapter 2 lens. What codecs did your softphone offer? What codecs did the trunk accept in the `200 OK`?

3. If any 401 or 407 appeared, note the auth loop: INVITE (no auth) → 401 → INVITE (with Authorization) → 100 Trying → 180 → 200.

### Part B — Diagnose three broken traces (45 minutes)

Below are three broken SIP traces. For each, write a 1–2 paragraph diagnosis: what happened, which header or absent message proves it, and what you would change.

**Trace B.1:**
```
#1  UAC -> UAS  INVITE sip:+447700900123@trunk.example.com
#2  UAS -> UAC  100 Trying
#3  UAS -> UAC  180 Ringing
#4  UAS -> UAC  180 Ringing
#5  UAS -> UAC  180 Ringing
    ...
#20 UAC -> UAS  CANCEL
#21 UAS -> UAC  200 OK (to CANCEL)
#22 UAS -> UAC  487 Request Terminated
#23 UAC -> UAS  ACK
```

**Trace B.2:**
```
#1  UAC -> UAS  INVITE sip:+15551234567@trunk.example.com  (SDP offers Opus only)
#2  UAS -> UAC  100 Trying
#3  UAS -> UAC  488 Not Acceptable Here
#4  UAC -> UAS  ACK
```

**Trace B.3:**
```
#1  UAC -> UAS  INVITE sip:+15552224444@trunk.example.com
#2  UAS -> UAC  100 Trying
#3  UAS -> UAC  200 OK (with SDP answer, c= line is 10.0.0.5 — a private RFC 1918 address)
#4  UAC -> UAS  ACK
    ...no RTP flows either way for 20 seconds...
#5  UAC -> UAS  BYE
```

For each trace: name the failure, identify the proving line(s), and prescribe a fix. Include which Chapter-4 header / mechanism (`Via`, `Record-Route`, SDP `c=`, 4xx code, etc.) is the key.

### Part C — Capture your own trace (45 minutes)

4. Place a real SIP call from your softphone to your Asterisk / FreeSWITCH test endpoint (or to your free-trunk test number).

5. Run `sngrep` on the terminating box:
   ```bash
   sudo sngrep -d <iface> port 5060 or port 5061
   ```

6. Capture **the entire call** — INVITE through BYE. Save as `my-call.pcap` (sngrep's "Save as PCAP" option).

7. Open the PCAP in Wireshark. For **every** SIP message in your trace:
   - Identify the type and status code.
   - Record the `Via`, `From` / `To`, `Call-ID`, `CSeq`, `Contact`, and `Record-Route` headers.
   - If a response, confirm the `Via` headers retrace in reverse.

8. Find the RTP packets corresponding to the negotiated SDP. Confirm:
   - The RTP source address matches the SDP `c=` line.
   - The payload type matches the negotiated codec (0 for µ-law, 8 for A-law, 9 for G.722, dynamic for Opus).

## Starter guidance

- Asterisk dev setup: install asterisk, enable `chan_pjsip`, create a `pjsip.conf` with one endpoint, register a softphone, dial `*43` for echo. The [Asterisk dev wiki](https://wiki.asterisk.org/wiki/) is the primary source.
- `sngrep` controls: arrow keys to navigate, Enter for the message detail, F2 to save as PCAP.
- Common softphone gotcha: if your softphone is behind NAT and your SBC isn't configured for symmetric NAT traversal, the SDP `c=` line will carry your private IP and the trunk can't send RTP back. This is Trace B.3's shape.
- `pjsua` CLI is the quickest way to generate a known-good INVITE if you want a reference to diff against your own stack.
- SIP over UDP has a 1300-byte MTU-ish limit; a long SDP triggers truncation and retransmits. If you see "fragmented SDP" in Wireshark, switch the trunk to TCP.

## Acceptance criteria

- [ ] Table in Part A completed for a real working trace (yours or a provided one).
- [ ] Three diagnosis paragraphs for B.1 (should name "180 kept ringing, caller gave up, CANCEL/487"), B.2 (should name "codec mismatch, 488, offer included no G.711"), B.3 (should name "SDP `c=` line is a private IP, RTP can't route back").
- [ ] A real captured PCAP from your own call, with every SIP message annotated.
- [ ] Confirmed RTP payload type matches the SDP-negotiated codec.
- [ ] Short reflection: for each of the three broken traces, name the **one** piece of observability you would add to catch it automatically in production.

## Stretch goals

- **Force a REFER-based transfer.** Configure Asterisk to transfer a call with `Transfer()` dialplan or `AMI` action; capture the resulting `REFER`, `NOTIFY sipfrag` messages. Annotate the dialog identity across the transfer.
- **Negotiate Opus over SIP.** Configure Asterisk's PJSIP to offer Opus (`chan_pjsip` supports it via codec configuration). Capture the SDP `a=fmtp:` line and compare to the WebRTC Opus fmtp from exercise 02. What survives on the SIP side?
- **Break it on purpose.** Modify your softphone to send a bogus `From` tag that changes between messages. Observe the server's reaction (expected: it treats the second message as a new dialog or rejects it).
- **TLS trace.** Switch the softphone to `sips:` on 5061; capture the TLS handshake with `ssldump` (or Wireshark + the server's TLS key). Confirm SIP messages travel inside TLS.

## Keep it

Save your annotated PCAPs. SIP trace reading is one of those skills that atrophies between incidents — a folder of 3–4 annotated real-world traces is a cheap self-refresher. Review them before your first shift on-call for a telephony-facing service.
