# TurnZero — Architecture

This document explains *why* the stack is shaped the way it is. The README tells you what the layers are; this tells you the engineering decisions and the tradeoffs behind them.

---

## 1. The design constraint that drives everything: the latency budget

A natural phone conversation has a turn-taking rhythm. Humans start to feel the gap at around 500ms and find it broken past ~1.2s of silence. For a voice agent the practical target is **end-to-end turn latency under 2 seconds** — measured from the moment the caller stops speaking to the moment the agent starts speaking.

That single number is the forcing function for the entire architecture. Every layer gets a slice of the budget, and the budget is gone the instant any layer stops being careful.

### Illustrative turn budget

| Stage | Layer | Target | Why |
|-------|-------|--------|-----|
| Endpointing (detect end of speech) | L2 PERCEIVE | ~200 ms | Too eager = cut the caller off; too slow = dead air |
| Final transcription | L2 PERCEIVE | ~150 ms | Streaming partials mean most of this is already done |
| Intent + first-token | L3 THINK | ~600 ms | First token, not full response — we stream |
| Retrieval (when needed) | L4 RECALL | ~300 ms | Runs in parallel with intent where possible |
| First-sentence synthesis | L5 SPEAK | ~350 ms | Speak sentence one while generating sentence two |
| Network / transport | L0–L1 | ~200 ms | μ-law frames, persistent connection |
| **Total to first audio** | | **~1.8 s** | Under the 2s ceiling, with headroom |

> These are design targets for the reference implementation, not a benchmark of any production system. The point is the *shape* of the budget and how the layers share it.

The key trick visible in this table: **nothing waits for "done."** The turn is a pipeline of overlapping streams, not a sequence of blocking calls.

---

## 2. Streaming-first, everywhere

A naive pipeline does this:

```
record fully -> transcribe fully -> prompt LLM -> wait for full response -> synthesise fully -> play
```

Every arrow is a blocking wait, and the latencies add up to something conversational death.

TurnZero overlaps them:

- **L2 PERCEIVE** emits partial transcripts continuously; by the time the caller stops, the transcription is essentially already done.
- **L3 THINK** streams tokens. We extract the **first complete sentence** as soon as it exists and hand it down to SPEAK while the rest of the response is still being generated.
- **L5 SPEAK** synthesises and begins playback on sentence one while sentence two is still forming upstream.
- A short, cached **hold/acknowledgement** can be played non-blocking at turn start to mask the first few hundred milliseconds when a retrieval is required.

The result is that perceived latency is dominated by *first audio out*, not *full response ready*.

---

## 3. Layer responsibilities and the boundaries between them

The discipline that keeps this maintainable: **each layer owns exactly one responsibility and exposes a narrow contract to the layer above.** This is the OSI lesson applied to a voice turn — you can swap the STT provider at L2 or the TTS provider at L5 without the layers above caring.

- **L0 CARRIER** — the telephony edge. PSTN/SIP gateway, carrier-grade routing, session persistence across reconnects. Below this line is the phone network; above it is our problem.
- **L1 STREAM** — audio transport. Bi-directional media pipeline, persistent connection, sub-frame synchronisation. Converts between the wire format (μ-law 8kHz) and what the engines upstream expect. *(Design note: in-process audio conversion massively outperforms shelling out to an external transcoder per frame — this layer is a hot path.)*
- **L2 PERCEIVE** — speech recognition. Streaming, language auto-detect across 11+ Indic languages, and code-switch detection (Hindi-English mid-sentence is the norm, not the exception, for Indian callers).
- **L3 THINK** — intelligence. Intent orchestration, predictive response generation, and a **confidence score** attached to every decision. The confidence score is what the trace plane and the defer logic key off.
- **L4 RECALL** — knowledge retrieval. A 3-tier adaptive retrieval pipeline (hybrid lexical + semantic), context assembly, and a relevance pipeline that decides what actually makes it into the prompt. Retrieval is loaded once at startup into memory, not per-call.
- **L5 SPEAK** — voice synthesis. Multi-provider routing (fail over / pick by language), Indic voice profiles, and streaming delivery so audio starts before the full text exists.
- **L6 CONVERSE** — turn management. This is where the conversation *feels* human: interruption handling (barge-in), silence management, and hold orchestration. Owns the turn state machine.
- **L7 CONNECT** — integration. Side-effects that outlive the turn: CRM sync, calendar booking, messaging. Deliberately the top layer so that business integrations never sit in the latency hot path.

---

## 4. Barge-in: why CONVERSE is a first-class layer

In a real conversation people interrupt. If the agent keeps talking over the caller, the illusion collapses instantly.

So **CONVERSE (L6)** continuously watches the inbound audio energy/VAD *even while the agent is speaking*. When it detects the caller has started talking, it:

1. Immediately stops outbound playback at L5.
2. Flushes the remainder of the queued response.
3. Hands the new inbound audio to PERCEIVE as a fresh turn.

Barge-in is the reason turn management can't be buried inside the LLM call — it has to be a layer that sits above synthesis and can pre-empt it.

---

## 5. The Trace Plane — observability as a contract, not an afterthought

Most systems log *events*: "call started", "transcript received", "response sent". That tells you what happened. It does not tell you **why the agent said what it said** — and on a confident-but-wrong answer, "why" is the only question that matters.

The Trace Plane is a cross-cutting rail. Each layer writes a small structured **receipt** for the turn:

```jsonc
{
  "turn_id": "…",
  "perceive": { "transcript": "…", "lang": "hi-en", "asr_confidence": 0.91 },
  "think":    { "intent": "check_balance", "intent_confidence": 0.78 },
  "recall":   { "hits": 3, "top_relevance": 0.64, "context_used": true },
  "decision": { "action": "answer", "response_confidence": 0.81 },
  "guardrail":{ "fired": false },
  "defer":    { "deferred": false, "reason": null },
  "latency_ms": { "perceive": 320, "think": 590, "recall": 280, "speak": 340, "total": 1790 }
}
```

Two design rules make this useful rather than decorative:

1. **It must not cost latency.** Receipts are assembled from values the layers already computed and emitted *off the live path* (async write). The turn never blocks on a trace write.
2. **It powers a real decision, not just a dashboard.** When `intent_confidence` or `top_relevance` falls below a threshold, the **defer** path triggers — the agent escalates, asks a clarifying question, or hands to a human, instead of confidently inventing an answer. The trace is what makes "I'm not sure, let me check" a *designed* behaviour rather than an accident.

For a regulated buyer (banking, BFSI, BPO), this receipt is the audit story: every answer can be reconstructed and explained after the fact.

---

## 6. Tradeoffs and things deliberately left out

- **This reference favours clarity over completeness.** The production system has far more around reconnection, billing, multi-tenancy and provider failover; none of that is here, because it obscures the architecture rather than illustrating it.
- **Provider choices are pluggable on purpose.** The layer contracts mean STT/TTS/LLM vendors are swappable. The repo picks one of each for the demo; the boundary is the point, not the vendor.
- **Confidence thresholds are illustrative.** Real thresholds are tuned per deployment against the eval harness in `/evals`, not hardcoded universally.

---

## 7. How to read the code next

Start at `src/orchestrator/` — that's CONVERSE and the trace plane, the heart of the turn. Then follow a turn downward: `telephony/` (L0) -> `voice/` (L1/L2) -> `llm/` (L3) -> `retrieval/` (L4) -> `voice/` (L5) and back up through CONVERSE to CONNECT. The `evals/` harness shows how confidence thresholds and latency are measured.
