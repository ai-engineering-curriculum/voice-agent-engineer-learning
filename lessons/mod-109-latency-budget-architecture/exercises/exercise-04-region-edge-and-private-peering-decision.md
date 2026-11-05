# exercise-04: Region, Edge, and Private Peering Decision

**Estimated effort:** 2 hours · **Output:** a deployment-placement decision document

## Objective

Given a concrete deployment scenario, make and defend placement decisions: which cloud region, which provider endpoint, whether to pay for private peering, how to route multi-region users. The deliverable is the document a Voice & Multimodal Agent Engineer would hand to infra / platform for review before provisioning.

This exercise is deliberately a *paper* exercise — you are not standing anything up. The point is to practise reasoning about trade-offs you will otherwise discover in production.

## Prerequisites

- [Chapter 4 — Region, Edge, and Private Peering](../04-region-edge-and-private-peering.md).
- Budget and architecture decisions from exercises 01 and 03.
- Access to published regional-availability documentation for the LLM / ASR / TTS providers you selected in exercise-01.

## Scenario (pick one)

1. **Launch-in-EU for a US-built product.** Your voice agent runs today in `us-east-1`. Legal and sales want you live in Western Europe within 60 days. User traffic will split roughly 60% US / 40% EU within six months.
2. **Clinical ambient scribe, multi-tenant.** Clinics in the US and Canada; HIPAA and provincial privacy laws apply. Your LLM provider supports `us-east-1` and `us-west-2`; your TTS provider is single-region (US); your ASR provider offers US and EU endpoints.
3. **Global consumer app, cost-constrained.** Users across NA, EMEA, and APAC. The product manager's target is 1.0 s p95 turn gap *globally*, with an absolute cap of two deployment regions due to budget.

## Steps

1. **Enumerate user geography.** For your scenario, list the user regions and the expected traffic share. Cite the brief if the scenario doesn't give you numbers; otherwise, write a one-sentence assumption.

2. **Enumerate provider region support.** For the specific LLM, ASR, and TTS providers you selected in exercise-01, document the regions they currently offer. Cite the provider doc for each. Tag any missing region as a risk.

3. **Choose media-server regions.** Decide how many, and which, cloud regions will host your media server. Justify in one paragraph. Reference the latency ladder from Chapter 4 ("same-region vs. cross-region vs. cross-continent").

4. **Choose provider-endpoint regions.** For each media-server region, pick the LLM / ASR / TTS provider-endpoint region you will route to. Justify each choice. Where the provider's nearest endpoint is cross-region or cross-continent, compute the one-way transit cost and state it explicitly.

5. **Compute the regional budget deltas.** For each media-server region, estimate how much of your 800/900/1000 ms turn-gap budget is consumed by network transit alone, in the hot path (orchestrator → LLM, orchestrator → TTS, media server → user). State the remaining budget available to ASR / LLM / TTS processing.

6. **Decide on private peering.** For each cloud region, answer yes/no: are you paying for AWS Direct Connect, Azure ExpressRoute, or Google Cloud Interconnect at launch? Justify in one paragraph. Reference the "jitter and tail consistency, not raw latency" argument from Chapter 4.

7. **Design the user-routing policy.** How does an inbound call reach the correct media server? Options: latency-based DNS, anycast, carrier edge (Twilio / Vonage / Zoom regional PoPs), WebRTC infrastructure's built-in routing, or an explicit region hint from the client. Pick one. Document the failure mode when the policy is wrong.

8. **Document the failover plan.** When the primary region is unavailable (region outage, provider outage, peering link down), where does the traffic go? What does the user experience? Is failover *active-active*, *active-passive*, or *cold*? Are you willing to accept a higher latency during failover?

9. **Write the deployment-placement decision document.** One to two pages. Structure:
   - Scenario summary and assumptions.
   - Table of media-server region, provider-endpoint region, measured / estimated one-way transit, peering decision.
   - Diagram of the routing policy.
   - Failover plan.
   - Open risks and next-step research items (what's still `<!-- needs-research -->` after this analysis).

## Starter guidance

- **Public-internet latency between major cloud regions is well-documented.** For architectural choices, consult up-to-date tools published by the clouds themselves (AWS CloudPing-style, Azure Latency Test, GCP network tiers) before quoting numbers. For this exercise a round-number estimate is fine, but tag it.
- **Do not confuse private peering with managed service networking.** AWS PrivateLink, Azure Private Link, and GCP Private Service Connect are different products from Direct Connect / ExpressRoute / Cloud Interconnect — they route traffic within the cloud, not from your DC to the cloud. Chapter 4 covers the latter; be explicit which you're invoking.
- **The user's device is not your region.** Mobile network latency to the nearest cloud PoP is typically 30–80 ms. Factor it in; don't assume LAN.
- **A cross-region failover that nobody has tested is not a failover.** Document the test cadence as part of the plan.

## Acceptance criteria

- [ ] User geography and traffic split documented.
- [ ] Provider regional availability documented with citations.
- [ ] Media-server region choice justified in one paragraph.
- [ ] Provider-endpoint region chosen for each media-server region, with explicit transit cost where it's cross-region.
- [ ] Per-region budget delta calculated: transit portion vs. remaining processing budget.
- [ ] Yes/no private peering decision per region with written justification.
- [ ] User-routing policy chosen with failure mode documented.
- [ ] Failover plan: where, how, what the user sees.
- [ ] One-to-two-page decision document suitable to hand to infra / platform for review.

## Stretch goals

- **Cost model.** Add a cost column to the per-region table: the monthly spend you would commit to for media-server compute, provider traffic, and private peering. Compare the three scenarios' total cost against the turn-gap budget you are trying to buy.
- **Compliance overlay.** For scenario 2 (HIPAA), identify which parts of the architecture are in scope for the BAA (Business Associate Agreement) with each provider. The deployment-placement decision and the compliance decision tend to interact; document the interaction.
- **Second-tier failover.** Extend the failover plan with a second-tier fallback that uses a *different* provider rather than a different region (e.g., ASR provider A fails, route to provider B in the same region). Document the eval work that would need to happen to make that fallback acceptable.

## Keep it

This document is the backbone of project-104's architectural brief. If you do it well here, you will reuse it there. If you do it poorly here, project-104 will find out.
