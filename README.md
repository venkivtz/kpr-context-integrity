
# KPR Validation Runner

A benchmark framework for measuring **Knowledge Primitive Retention (KPR)** — a metric that quantifies how much decision-critical information survives the ingestion pipeline before reaching a language model.

Built to demonstrate a fundamental flaw in standard RAG architectures: **the context quality problem cannot be fixed downstream**. When structural primitives are severed at ingestion, no reranker, prompt engineer, or model upgrade can recover the missing information.

---

## The Problem

Most RAG pipelines treat documents as flat text blobs and split them by token count or character windows. For documents where precision matters — clinical trial protocols, legal contracts, compliance manuals, regulatory filings — this creates a silent failure mode.

Consider a clinical protocol clause:

> *"If a patient does not take their dose within **12 hours** after the scheduled dosing time (for patients in **Arm B**) or within **24 hours** (for patients in **Arm C**), the dose should be skipped."*

A naive token splitter might return only the Arm C half of this clause. The LLM receives:

> *"...within 24 hours after the scheduled dosing time (for patients in Arm C), then the dose should be skipped."*

The model now has no mathematical path to answer an Arm B question correctly — not because it is unintelligent, but because the variable was amputated before it ever arrived.

**This is the numerator bottleneck.**

---

## What is KPR?

**Knowledge Primitive Retention (KPR)** is a precision metric defined as:

```
KPR = (Primitives Present in Retrieved Chunk) / (Total Required Primitives)
```

A **knowledge primitive** is any atomic unit of information whose absence changes the answer: a threshold value, a conditional arm identifier, a time window, a negation qualifier, a mandatory sequence marker.

KPR is calculated **programmatically** — by checking whether each required primitive string exists in the retrieved text segment. This makes it:

- **Deterministic** — no LLM judge, no human annotation bias
- **Falsifiable** — the math is visible and reproducible
- **Compositional** — primitives can be audited and extended per domain

A KPR of `1.00` means the model received every variable it needs. A KPR of `0.33` means it received one-third, and the answer is statistically likely to be wrong regardless of model capability.

---

## Architecture: Three Techniques Compared

The benchmark runs each test case through three retrieval configurations:

| Technique | Method | Behavior |
|---|---|---|
| **Tech A** | Naive token split | Splits on character/token boundaries without regard for semantic units |
| **Tech B** | Downstream reranker | Applies a reranking pass over naively chunked fragments — same fractured input, reordered |
| **Tech C** | Layout-aware ingestion | Preserves structural boundaries (sections, clauses, tables) before chunking |

The key finding: **Tech B's KPR score is mathematically identical to Tech A's.** A reranker changes which broken fragment surfaces first — it cannot manufacture a primitive that was never ingested. The error is upstream; no downstream intervention corrects it.

---

## Benchmark Suite

Five test cases drawn from a real Phase 2 clinical trial protocol (`Prot_000_tacadia.pdf` — MLN0128/TAK-228 in combination with Fulvestrant, NCT02756364):

| ID | Question Domain | Key Primitives at Risk |
|---|---|---|
| 1 | Dose timing threshold by arm | `Arm B`, `12 hours`, `Arm C`, `24 hours` |
| 2 | PK/ECG mandatory execution sequence | `coincides with`, `vital sign or ECG`, `before the PK` |
| 3 | Female patient age eligibility | `Female`, `18 years of age or older`, `breast cancer` |
| 4 | Diabetes insulin exclusion | `Type 2 diabetes`, `requiring insulin modifications`, `insulin stabilization` |
| 5 | Safety follow-up time window | `30 days`, `after the last dose` |

These cases were selected because each has a naive fragment that looks plausible enough to fool a reranker but is factually incomplete in ways that would cause protocol deviations in a real clinical setting.

---

## Output

The runner produces a side-by-side matrix:

```
ID   Question Synopsis         Tech A KPR   Tech B KPR   Tech C KPR   System Status
1    If an Arm B patient is...  0.50         0.50         1.00         100% Secure
2    What is the mandatory e...  0.33         0.33         1.00         100% Secure
...
```

Followed by live LLM outputs for each technique, showing the downstream answer quality that corresponds to each KPR score.

---

## Repository Structure

```
.
├── kpr_validation_runner.py     # Main benchmark runner
├── requirements.txt             # Python dependencies
├── Prot_000_tacadia.pdf         # Source document: clinical trial protocol (MLN0128)
├── KPR_measurement_method_v2.pdf  # KPR metric definition and mathematical derivation
├── rag_limitations_v3.pdf       # Analysis: why naive RAG fails for high-stakes documents
├── README.md
└── QUICKSTART.md
```

---

## Model Backend

The runner calls a locally hosted Ollama instance running `llama3.2:latest`. This is intentional — the benchmark is designed to be **model-agnostic**. The KPR score is computed before the LLM is invoked, so model capability is held constant. The point is not that llama3.2 is weak; the point is that even a stronger model cannot answer correctly when its context is incomplete.

To use a different model, change `MODEL_NAME` at the top of `kpr_validation_runner.py`.

---

## Theoretical Grounding

The KPR framework is grounded in two observations:

**1. Information-theoretic ceiling.** A language model's output is bounded by its input. If the correct answer requires variable X and X is absent from the context window, the model must either hallucinate, refuse, or guess. This is not a model failure — it is an infrastructure failure.

**2. Reranking cannot reconstruct lost information.** Rerankers are scoring functions over existing candidates. They optimize rank order. If the correct chunk was never retrieved — because it was severed from its sibling clause at ingestion — the reranker has nothing correct to promote. KPR makes this visible by computing the numerator score before the LLM call, exposing exactly what was lost and where.

The full derivation and methodology are in `KPR_measurement_method_v2.pdf`. The empirical failure patterns motivating this work are documented in `rag_limitations_v3.pdf`.

---

## Contributing

Issues and PRs welcome, particularly for:
- Additional domain-specific primitive sets (legal contracts, regulatory filings, financial disclosures)
- Alternative chunking strategies for Tech A/B comparison baselines
- Adapters for other LLM backends (OpenAI, Anthropic, Gemini)

---

## License

MIT

