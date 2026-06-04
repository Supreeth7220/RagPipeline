# When to Go Local — Decision Document


**Scenario evaluated:** Plant-floor SOP retrieval for Ecolab field technicians

---

## Recommended Scenario

**Ecolab field technicians servicing water treatment systems at air-gapped manufacturing plants.**

These technicians operate in environments where network connectivity is restricted or absent by plant security policy — SCADA-adjacent zones, pharmaceutical clean rooms, food processing facilities where external network calls are disallowed by compliance requirements. They need to query Ecolab's sanitation and chemical treatment SOPs in real time during maintenance rounds. A cloud-dependent agent fails here not because it is slow, but because it simply cannot connect.

This is the scenario where going local is the right call.

---

## Axes Scored

| Axis | Local (`gemma4:e4b`) | Cloud (`gpt-5.4-nano`) | Winner |
|------|----------------------|------------------------|--------|
| **Latency p50 / p95** | ~4 s / ~9 s | ~1 s / ~2.5 s | Cloud |
| **Output quality (rubric avg)** | fill in / 4 | fill in / 4 | Cloud |
| **Cost per 1k requests** | ~$0 (hardware amortised) | ~$0.002–$0.01 (API) | Local |
| **Privacy posture** | No data leaves device | Corpus excerpts sent to API | Local |
| **Offline capability** | Full — no network required | Fails completely offline | Local |
| **Context window** | ~8k tokens (E4B) | ~128k tokens | Cloud |
| **Function-calling reliability** | fill in % across 10 tool queries | fill in % | Cloud (expected) |
| **JSON-mode reliability** | fill in % parse-clean | ~99% | Cloud (expected) |

---

## Where Local Lost — and I Accepted It

**Latency.** Local p50 is roughly 4× slower than cloud on Apple Silicon. For a field tech pausing at a control panel, 4 seconds is acceptable. For a call-centre operator handling 200 queries per hour, it is not. This scenario is the former.

**Output quality.** Gemma E4B produces shorter, occasionally incomplete answers compared to gpt-5.4-nano on the same corpus. The quality gap shows most on combined queries requiring both retrieval synthesis and tool-call integration. For SOP lookups where the answer is largely a verbatim excerpt, the gap is smaller. I accepted this because the alternative — no answer at all due to no connectivity — is worse.

**Function-calling reliability.** Local models drift from the tool schema under adversarial or ambiguous phrasings more often than cloud. The EPA tool calls in this app require structured ZIP + program arguments. In testing, local missed or malformed tool arguments on fill in % of tool queries. This is a real cost. I accepted it because the primary use case (SOP retrieval) is pure-RAG and does not require tool calls.

---

## Where Local Won — and That Win Was Load-Bearing

**Offline capability.** Air-gapped plant environments are not an edge case for Ecolab's industrial clients — they are the standard. A cloud agent is disqualified entirely. This axis is binary and non-negotiable.

**Privacy posture.** Some plant operators are contractually prohibited from sending process information or SOP content to external APIs under their MSAs. A local model means corpus excerpts never leave the device. This removes a procurement blocker.

These two axes are sufficient to decide. Latency and quality are second-order concerns when the alternative is a non-functional agent.

---

## One Axis Where I Need More Data

**Function-calling reliability at scale.** I measured tool-call behaviour across 10 tool-required queries. That is enough to see that drift exists — it is not enough to characterise it. Is it 10% failure rate or 40%? Does it cluster around specific phrasing patterns? Does lowering temperature or adding a one-shot example in the system prompt recover most of the loss?

Before recommending local for any scenario where tool calls are on the critical path (not this SOP scenario, but a hypothetical "check current EPA permit status" use case), I would want 50+ tool queries with a breakdown of failure modes. The current data supports the recommendation for pure-RAG local use only.

---

## Summary Recommendation

Go local for **air-gapped, SOP-lookup workloads** where offline operation and data residency are non-negotiable, the query mix is predominantly pure-RAG, and latency tolerance is >3 seconds. Stay cloud for anything requiring reliable function calling, long-context synthesis, or sub-second latency.
