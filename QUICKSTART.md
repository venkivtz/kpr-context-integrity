# Quickstart

Get the KPR benchmark running in under 10 minutes.

---

## Prerequisites

- Python 3.9+
- [Ollama](https://ollama.com) installed and running locally
- The `llama3.2` model pulled in Ollama

---

## 1. Install Ollama and pull the model

```bash
# Install Ollama (macOS/Linux)
curl -fsSL https://ollama.com/install.sh | sh

# Pull the model used by the benchmark
ollama pull llama3.2:latest

# Verify it's running
ollama list
```

Ollama must be running on `http://localhost:11434` before you execute the benchmark. On most systems it starts automatically after install. If not:

```bash
ollama serve
```

---

## 2. Clone the repository

```bash
git clone https://github.com/your-org/kpr-validation-runner.git
cd kpr-validation-runner
```

---

## 3. Set up a virtual environment and install dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Key packages installed:
- `PyPDF2` — PDF reading
- `requests` — Ollama API calls
- `pdfplumber`, `pdfminer.six` — layout-aware text extraction (used in extended experiments)

---

## 4. Run the benchmark

```bash
python kpr_validation_runner.py Prot_000_tacadia.pdf
```

The protocol PDF (`Prot_000_tacadia.pdf`) must be in the same directory as the script, or pass a full path:

```bash
python kpr_validation_runner.py /path/to/Prot_000_tacadia.pdf
```

---

## 5. Read the output

The runner will print results for all five test cases, then a final matrix. Each test case shows three blocks:

```
[Executing Tech A: Naive Token Split...]
  ↳ Mathematical KPR: 1/4 = 0.25
  ↳ Live Output: "The patient can take the dose."    ← wrong

[Executing Tech B: Downstream Reranker Selection...]
  ↳ Mathematical KPR: 1/4 = 0.25
  ↳ Live Output: "The patient can take the dose."    ← still wrong

[Executing Tech C: Layout-Aware Structural Context Ingestion...]
  ↳ Mathematical KPR: 4/4 = 1.00
  ↳ Live Output: "No. The threshold for Arm B is 12 hours..."    ← correct
```

The **Mathematical KPR** is calculated before the LLM call — it tells you exactly how many required primitives were present in the context chunk. Tech A and Tech B will consistently show KPR < 1.00 on multi-conditional clauses. Tech C reaches 1.00 because layout boundaries keep related variables together.

---

## 6. Understanding the final matrix

At the end of the run you'll see:

```
FINAL ARITHMETIC PERFORMANCE MATRIX: SIDE-BY-SIDE VERIFICATION
ID   Question Synopsis         Tech A KPR   Tech B KPR   Tech C KPR   System Status
1    If an Arm B patient is...  0.50         0.50         1.00         100% Secure
2    What is the mandatory e...  0.33         0.33         1.00         100% Secure
...
```

- **Tech A KPR = Tech B KPR** on every row. This is the core proof: reranking does not recover severed variables.
- **Tech C KPR = 1.00** on every row when the source document is intact, confirming that structural ingestion preserves all required decision variables.
- **System Status** flags cases where Tech A/B produce dangerously incorrect answers despite the model being technically capable.

---

## Configuration

All configuration lives at the top of `kpr_validation_runner.py`:

```python
OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL_NAME = "llama3.2:latest"
```

To switch models, change `MODEL_NAME` to any model available in your Ollama instance (e.g. `mistral:latest`, `phi3:medium`, `gemma2:9b`). The KPR math is model-independent; only the live output text will change.

---

## Adapting to a different document

The benchmark suite is defined in the `BENCHMARK_SUITE` list. Each entry has:

```python
{
    "id": 1,
    "question": "...",              # The question to ask the LLM
    "ground_truth": "...",          # Expected correct answer (for reference)
    "primitives": ["...", "..."],   # Atomic strings whose presence/absence determines KPR
    "full_context": "...",          # The complete, structurally intact clause (Tech C input)
    "naive_fragment": "..."         # The severed fragment a naive chunker would return (Tech A/B input)
}
```

To test a different document:
1. Identify the decision-critical clauses in your document.
2. For each clause, list the **primitives** — threshold values, conditional identifiers, negation keywords, mandatory sequence markers.
3. Write a `naive_fragment` that simulates what a fixed-window chunker would return (typically the second half of a conditional clause, or a clause stripped of its section header).
4. Set `full_context` to the complete, unbroken clause.

The KPR calculation and LLM calls are fully automated from there.

---

## Troubleshooting

**`Connection Failed: ...` in live output**
Ollama is not running. Start it with `ollama serve` and retry.

**`API Error` in live output**
The model name in `MODEL_NAME` does not match an installed model. Run `ollama list` to see available models.

**`Error: 'Prot_000_tacadia.pdf' not found`**
The PDF is not in the working directory. Pass the full path as an argument or copy it to the project folder.

**KPR scores all show 0.00 for Tech C**
Check that `full_context` strings in `BENCHMARK_SUITE` match the actual text in the PDF. Primitive matching is case-insensitive substring search — check for Unicode or encoding differences in copy-pasted protocol text.

---

## Next steps

- Read `KPR_measurement_method_v2.pdf` for the full metric definition, scoring rules, and edge cases.
- Read `rag_limitations_v3.pdf` for an analysis of why standard chunking strategies systematically fail on structured regulatory documents.
- Extend the benchmark with your own domain primitives for legal, financial, or other high-stakes document types.
