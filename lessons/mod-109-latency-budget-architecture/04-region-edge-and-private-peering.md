# 04. Region, Edge, and Private Peering

## Why this chapter

A streaming graph that could hit 800 ms if the LLM provider is next door will miss by half a second if the LLM is in a different continent. Every voice-agent architecture eventually makes four placement decisions:

1. Which **cloud region** the media server runs in.
2. Which **region of the LLM / ASR / TTS provider** receives the request.
3. Whether the path between (1) and (2) traverses the **public internet** or a **private peering** link.
4. Where the **user's device** is, relative to all of the above.

The first three are things you decide. The fourth is a thing you plan for.

This chapter is about the latency cost of each decision and when to pay for which mitigation.

## The rough numbers you have to internalise

A short, memorise-able latency ladder you can reason from (one-way, approximate):

- **Same-datacentre, same-rack** — sub-millisecond.
- **Same-region, cross-AZ** — a few milliseconds (typically 1–3 ms within a cloud region).
- **Same-country, cross-region** — tens of milliseconds (e.g., US-East ↔ US-West typically 60–80 ms one-way).
- **Cross-continent** — 70–150 ms one-way (e.g., US-East ↔ Western Europe ~70–90 ms; US-West ↔ Asia-Pacific ~100–150 ms).
- **Cross-continent via a bad path** — >200 ms one-way, especially on consumer internet without CDN routing.

These are **network path** numbers, not application latency. Add TLS handshake, provider ingress processing, and queueing on top. For a budget calculation, "a cross-continent hop costs you 100–150 ms one-way, and you take it both ways" is the right order-of-magnitude intuition.

For authoritative, up-to-date numbers, every major cloud publishes latency matrices and status pages. Use those for architectural choices, not round numbers from folklore. <!-- needs-research: link to current AWS / Azure / GCP region-latency reference tools (e.g., CloudPing.co, Azure Latency Test) — these are third-party and should be verified before citing -->

## Rule 1: Colocate inference with the media server

Your voice-agent's media server — the WebRTC SFU, the SIP gateway, the WebSocket terminator, whichever transport you ended up with — is where the user's audio lands first. **Everything downstream of that should be in the same region as the media server.**

- The media server already serves the user from a chosen region to minimise ingress / egress RTT.
- LLM / ASR / TTS traffic is high-throughput, bidirectional, and streaming. Putting any of it a region away from the media server adds a round trip to *every* audio chunk going in and every token / audio chunk coming out.
- The most expensive mistake is running the media server in the user's region but the agent orchestrator and provider calls in a single "home" region — common when a voice product is bolted onto an existing text-LLM architecture.

The cheap version of "colocate" is "deploy the orchestrator in the same cloud region as the media server, use the provider's same-region endpoint, done". The expensive version is self-hosting the inference on GPUs in your own region. Chapter 6 discusses when the latter is actually worth it.

## Rule 2: Pick the provider region the same way you pick yours

Every major LLM / ASR / TTS provider offers regional endpoints:

- **OpenAI / Azure OpenAI** — Azure offers regional deployments of OpenAI models across multiple geographies.
- **Anthropic** — Claude is available directly and via AWS Bedrock and Google Vertex AI; regional availability depends on the surface.
- **Google (Vertex AI / Gemini)** — regional endpoints via Vertex; regional availability varies by model.
- **AWS Bedrock** — region-specific availability per model.
- **Specialist providers (Deepgram, ElevenLabs, Cartesia, LMNT)** — some offer multiple regions and some operate out of a single region. Check provider documentation for current region availability before committing an architecture.

What this means operationally:

- **If the provider has an endpoint in your media-server region, use it.** This is the single most impactful region-level decision.
- **If the provider does not,** your budget takes a one-way transit cost to the nearest region. Factor it in up front, do not discover it in production.
- **If the provider's model routing may hop regions internally** (some do, for availability / capacity reasons), you cannot assume your "same-region endpoint" is actually single-region at all times. Instrument your TTFT distribution and look for bimodality — it is often a sign of occasional cross-region failover.

## Rule 3: Private peering buys latency *and* consistency, not just latency

Each major cloud provides a private-peering product:

- **AWS Direct Connect** — dedicated physical connections from your network to AWS, with optional partner integrations and regional endpoints.
- **Azure ExpressRoute** — private connectivity into Azure that does not traverse the public internet.
- **Google Cloud Interconnect** — Dedicated and Partner Interconnect products for private connectivity into Google Cloud.

The latency win from private peering, in isolation, is often **smaller than beginners expect** — the public-internet path between major carriers and major clouds is already very good on the median. The real wins:

- **Lower jitter and lower tail latency.** The public path can have bad days. A dedicated circuit does not.
- **Deterministic path.** You can size and monitor the link.
- **Compliance and data-locality.** For regulated voice applications (clinical, financial, regulated geographies) the control-plane and audit answers are easier.

The cost:

- Capital commitment, usually monthly.
- Operational overhead — somebody has to own a BGP session.
- It is a hub-and-spoke architecture: you peer with *one cloud* (or a few via Megaport-style fabric), not with every provider's data centre.

**When private peering is probably worth it**: you are a regulated voice product (healthcare, financial services), your provider supports it, you operate at consistent volume, your tail latency is a product differentiator, or you're already running on-prem contact-centre infrastructure that's peering with cloud anyway. **When it probably is not**: a new product with variable traffic, or a product whose TTFT variance is dominated by provider-side queueing rather than network transit.

As a pattern, **do not engineer private peering into v1**. Start with same-region public endpoints, measure, and only peer when the measurement says the link is your bottleneck.

## Rule 4: The user's region is a product decision

The user's device is where you cannot place a server. But the user's geography *is* a knob you can turn at the product layer:

- **Multi-region deployment.** Deploy media servers in multiple regions and route each call to the nearest one by latency-based routing (DNS-based, or anycast at the edge, or the provider's own routing). WebRTC infrastructure vendors handle this out of the box.
- **Telephony edge.** Twilio, Vonage, and other CPaaS providers operate PoPs (points of presence) regionally. A call originating in APAC can hit an APAC media gateway before you ever see it, meaning the public-internet leg from the caller is tens of milliseconds, not hundreds.
- **Locale-aware routing.** When your user's region and your provider's region don't match (e.g., EU user, but your TTS provider is US-only), you may route *the voice traffic* to a US media server to minimise the orchestrator-to-provider hop at the cost of a worse user-to-media-server hop. Model both options and measure.

A warning: **do not conflate region with data residency**. The region your traffic travels through is a latency question; the region your data is stored and processed in is a compliance question (mod-108 and GDPR). Private peering helps with both but the two problems are not the same.

## A decision table

For any voice-agent deployment, you should be able to answer these six questions on request:

| Question | Default answer | When to deviate |
|---|---|---|
| Where is the media server? | Same region as the bulk of users | Multi-region for geographic spread |
| Where is the orchestrator? | Same region as the media server | Rarely; only when an upstream system forces it |
| Where is the LLM provider endpoint? | Same region as the orchestrator | If model/region availability forces cross-region |
| Where is the ASR / TTS endpoint? | Same region as the orchestrator | Same caveat |
| Private peering? | Not until measurement justifies it | Regulated vertical, or measured network tail |
| Fallback when the near region is down? | Documented and tested cross-region failover | Never: a cold failover is worse than a down region |

If you cannot answer all six, the deployment has unresolved budget risk.

## The cost of a cross-region hop, concretely

A concrete walkthrough so the numbers stop being abstract:

- **Scenario**: EU user, media server in EU-West, orchestrator in EU-West, LLM provider endpoint only in US-East. One-way transit EU-West ↔ US-East is typically 70–90 ms.
- **Cost on turn gap**:
  - Prompt send: +~80 ms (orchestrator → provider).
  - First token response: +~80 ms (provider → orchestrator).
  - Every subsequent token: no incremental budget cost if the orchestrator is streaming to TTS downstream. TPOT is amortised during playback.
- **Net budget cost**: ~160 ms added to the critical path between prompt send and first TTS byte. That is the difference between a 650 ms turn gap and an 810 ms turn gap — the difference between shipping and not shipping, for a strict 800 ms product target.

The same walkthrough for TTS traffic would show a second ~160 ms added before the first audio byte reaches the user. If that is unaffordable, you must either (a) move the orchestrator/provider closer to the user, (b) move the user's media server closer to the provider (worse for user-facing RTT), or (c) renegotiate the budget. There is no fourth option.

## Summary

- **Colocate inference with the media server.** Same cloud region, as a default.
- **Use the provider's same-region endpoint** wherever one exists. If it doesn't, budget a cross-region transit cost and track TTFT bimodality.
- **Private peering** (Direct Connect / ExpressRoute / Cloud Interconnect) buys jitter and tail consistency more than raw latency. Add it when measurement justifies it, not before.
- **The user's region is a product decision**: multi-region media servers and telephony-edge PoPs reduce user-facing RTT.
- **A cross-continent hop costs you 100–150 ms one-way**, taken twice on the critical path. That number should be in every architecture-brief trade-off you write.

Next chapter: enforcing the whole budget as an engineering gate — synthetic callers, CI gates, kill-switches.
