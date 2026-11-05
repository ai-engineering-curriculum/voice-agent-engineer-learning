# 07. The Boundary to Applied-AI and NLP Engineering

## Why this chapter

Multimodal work pulls in two adjacent specialisations that have overlapping tooling but different problem framings. If you treat every multimodal problem as a voice-agent problem, you will find yourself training feature extractors or reading multimodal-model papers when you should be scoping an escalation. If you treat every multimodal problem as "ask the big model nicely", you will miss the places where cheap in-house feature engineering is the better answer.

This chapter is the sharp boundary to two peer specialisations: the **applied-ai-engineer** (L30) who owns bespoke image / document feature engineering, and the **nlp-engineer** (L30) who owns deep multimodal-model research. Where each lives, what you escalate, and how to package the request.

## The one-paragraph handoff

> You own the **multimodal input design for a voice-agent turn** — the modality contract, the provider wiring, the frame-rate and routing policy, the grounding scheme, the eval harness. You do **not** own the training or fine-tuning of custom vision models, the research into new multimodal architectures, or bespoke document-feature extractors that live as in-house services. Your job is to pick the right off-the-shelf multimodal path, measure its behaviour, and know when the right answer is "ask the applied-AI or NLP team to build us a thing."

## The applied-AI-engineer boundary (L30 peer)

The applied-AI-engineer builds in-house ML features for specific product needs: a document classifier, an invoice-field extractor, a logo detector, a resume parser. Their output is usually a service — "POST an image, get structured fields back" — that your orchestrator can call as a tool.

### When you call them in

Signals that the right move is an applied-AI feature rather than another LLM-vision call:

- **The task is narrow and high-volume.** You have one specific thing to do (read price tags on shelf photos; extract line items from a receipt; identify the model number from a label photo) at a volume where even cheap vision-LLM calls add up. A small fine-tuned vision model, or an OCR+regex pipeline, is cheaper and more predictable.
- **Accuracy must beat what a general-purpose LLM can do.** The vision-LLM is at 88% on your task; you need 98%. A targeted model trained on your data is a plausible path.
- **Latency floor matters.** A hosted LLM vision call has a floor set by the provider. A locally-deployed detection model can shave hundreds of milliseconds.
- **Privacy prohibits sending the image to a third party.** PHI, PII, internal documents. The applied-AI engineer builds a self-hosted path.
- **The feature needs structured, verifiable output** that an LLM's free-form text doesn't give cleanly — bounding boxes with class labels, structured fields with confidence scores.

### What your side of the boundary looks like

Your orchestrator treats the applied-AI service as a **tool**. From the agent's perspective, calling `extract_receipt_fields(image_id)` is no different from calling `search_knowledge_base(query)`. The applied-AI team owns the tool's accuracy, latency, and SLOs; you own the integration and the fallback behaviour when the tool fails.

### What the escalation looks like

Package the ask the way mod-109 Chapter 6 taught for the inference team:

1. **The task** — a one-line statement of what the service should do. "Extract vendor, total, and line items from a receipt photo."
2. **The volume** — expected QPS, peak, concurrent sessions, retention requirements.
3. **The current baseline** — how a general vision LLM performs on this task on a labelled sample, with per-field accuracy numbers.
4. **The target** — the per-field accuracy you need and why (what downstream decision breaks below it).
5. **The data** — a labelled sample they can train on, or an agreement on how to collect one.
6. **The interface** — the proposed request / response shape and SLO.
7. **The fallback** — what the agent does if the service is unavailable.

If the right answer for a given task is "just prompt the big model better", say so. Not every narrow task is worth a bespoke model; the applied-AI engineer will tell you when it's not.

## The NLP-engineer boundary (L30 peer)

The NLP-engineer owns deep research on multimodal and language models: evaluating emerging architectures, probing failure modes, designing new eval benchmarks, prototyping fine-tunes of open-weight models. Their output is usually a *decision* ("we should evaluate architecture X for this domain", "model Y has a known weakness on Z"), a *dataset*, or an *evaluation*.

### When you call them in

- **You need to choose between providers on a nuanced criterion** that goes beyond public benchmarks. Which model handles multi-page documents best. Which model's grounding is most reliable. Which model's refusal behaviour is correct for a medical sub-domain.
- **You are considering a self-hosted multimodal model** (LLaVA-family, Qwen-VL, InternVL, PaliGemma) and need an architectural evaluation.
- **You see a failure mode** in production that seems systematic and want someone to characterise it: "the model is consistently misreading Arabic numerals on low-contrast backgrounds" — is that a known issue, is it fixable with a prompt change, or is it an architectural limit?
- **You want a bespoke eval benchmark** for your domain — in-house MMMU-equivalent for your specific user traffic.
- **You want to **fine-tune** a multimodal model** on your data. This is their expertise, not yours; they own the pipeline from dataset collection to deployed artefact.

### What you own anyway

- The **product-level question** — the reason you are asking.
- The **in-domain eval set** from Chapter 6. You generate it; they run it.
- The **integration decision** — if they recommend architecture X, you decide whether to ship with it. You own the voice-agent latency budget, cost budget, and safety envelope; their recommendation is an input.

### What the escalation looks like

- **"We're seeing X in production — is this an architectural limit?"** Attach a labelled sample of 50+ cases. Expect a one-week turnaround and an answer that is more nuanced than yes/no.
- **"We're considering shipping model Y — please eval it on our in-domain set."** Give them the set. The output is a scored comparison, not a recommendation; the ship decision is still yours.
- **"We want to fine-tune model Z for this task."** This is a project, not a ticket. Expect a scoping conversation, a dataset discussion, and a multi-month timeline.

## The anti-patterns on both sides

Worth being explicit about the failure modes the boundary prevents.

### "I'll just train a model"

A voice-agent engineer who reaches for a self-trained vision model without consulting the applied-AI team will almost always: (a) under-scope the labelling work, (b) over-fit on an unrepresentative training set, (c) ship a model with no eval infrastructure, (d) carry the operational burden forever. The right path is a conversation about whether to do it at all, then a handoff if yes.

### "The LLM will handle it"

Equally common and equally expensive. Every narrow task — especially high-volume, structured-output, privacy-sensitive — is a candidate for an applied-AI path. Default to the LLM, but check: is this task narrow enough to be cheaper as a dedicated service?

### "This is a research problem, not a product problem"

Any failure mode that requires actually understanding a multimodal model's internals — attention patterns, cross-modal fusion quirks, tokeniser behaviour on images — is a hand-off to the NLP engineer. You own the symptom and the mitigation; they own the diagnosis.

### "This is a product problem, not a research problem"

Some failures that look like model issues are actually your fault: a resolution choice too low, a prompt that doesn't request grounding, a router that drops vision on turns that need it. Before escalating to the NLP team, check the mod-110 Chapter 2–5 knobs.

## A joint-work shape that works

For non-trivial multimodal features, a three-way shape is common and productive:

- **Voice-agent engineer (you)**: owns the user-facing behaviour, the latency and cost budget, and the orchestration layer. Integrates whatever the other two teams build.
- **Applied-AI engineer**: owns the in-house feature extractor service (if there is one). Delivers a tool-callable endpoint with SLOs.
- **NLP engineer**: owns the evaluation, the model-choice recommendation, and any fine-tuning work. Delivers a vetted model artefact or recommendation.

A weekly sync keeps all three aligned; a shared eval harness keeps all three honest. If any of the three is missing, one of the others is doing their work badly.

## A one-line summary of the whole module

> You own the multimodal input design for a voice-agent turn: the contract, the provider wiring, the frame-rate and routing policy, the grounding scheme, and the eval. You do not own bespoke feature-extractor training or deep multimodal-model research. Measure your side, escalate theirs, and never confuse the two.

That boundary is what keeps you from spending a quarter fine-tuning a model when a routing change would have solved the problem, and equally keeps you from asking a general LLM to do a job a small trained classifier would do better.

## Summary

- **Inside applied-AI**: bespoke in-house image / document feature extractors. Called as tools from your orchestrator. Owned at L30 peer.
- **Inside NLP engineering**: deep multimodal-model research, architectural evaluation, bespoke benchmarks, fine-tuning. Owned at L30 peer.
- **Owned by you**: the modality contract, the provider wiring, the frame-rate policy, the routing heuristic, the grounding scheme, the eval harness, and the integration decisions.
- **Escalate well**: task + volume + baseline + target + data + interface + fallback. Package the request so the other team can act on it.

End of module. Next up: [mod-111-voice-agent-orchestration](../mod-111-voice-agent-orchestration/README.md).
