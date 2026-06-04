# Ecolab RAG Agent

A retrieval-augmented generation (RAG) agent for Ecolab's domains: water treatment, hygiene, and sustainability. Answers questions from a local document corpus and can look up EPA-regulated facilities by ZIP code via tool calling.

Supports two runtime profiles:
- **`cloud`** — Azure OpenAI for chat (`gpt-5.4-nano`) and embeddings (`text-embedding-3-small`, 1536 dims)
- **`local`** — Ollama for chat (`gemma4:e4b`) and embeddings (`nomic-embed-text`, 768 dims), fully offline

---

## Local Setup (target: under 15 minutes)

### Prerequisites

- macOS with Homebrew (`brew --version`)
- Python 3.10+ (`python3 --version`)
- 16 GB RAM recommended (8 GB works with `gemma3:1b` — see FAQ)

### Step 1 — Install Ollama

```bash
brew install ollama
brew services start ollama
ollama --version   # confirm: ollama version x.x.x
```

### Step 2 — Pull models (~3 GB total)

```bash
ollama pull gemma3:e4b          # LLM (~2.5 GB)
ollama pull nomic-embed-text    # Embedder (~270 MB)
```

Smoke test:
```bash
ollama run gemma3:e4b "Hello"
```

### Step 3 — Install Python dependencies

```bash
pip install -r requirements.txt
```

### Step 4 — Configure environment

```bash
cp .env.example .env
```

Edit `.env` and set:
```
LLM_PROFILE=local
```

No Azure credentials are needed for local mode. The local model vars are pre-filled:
```
LOCAL_CHAT_MODEL=gemma3:e4b
LOCAL_EMBEDDING_MODEL=nomic-embed-text
LOCAL_BASE_URL=http://localhost:11434/v1
LOCAL_COLLECTION=Corpus_local
```

### Step 5 — Ingest corpus with local embeddings

```bash
python ingest.py
```

This creates a fresh ChromaDB collection (`Corpus_local`) with 768-dim `nomic-embed-text` vectors. Do not reuse the cloud collection — the embedding dimensions are incompatible (1536 vs 768).

Expected output:
```
Found 1 file(s). Loading and chunking...
  WHO 2018 Sanitation and Hygiene guidelines.pdf → N chunk(s)
Embedding N chunk(s)...
Storing in ChromaDB at ./chroma_db...
Done. Collection has N chunk(s) total.
```

### Step 6 — Run the agent

```bash
streamlit run app.py
```

Open http://localhost:8501 in your browser. Try a question like:
> "What are the WHO recommendations for hand hygiene in healthcare settings?"

---

## Profile Switch — `LLM_PROFILE`

Set `LLM_PROFILE` in `.env` (or export it as an environment variable) to switch between profiles:

```bash
# Local mode (Ollama, fully offline after model pull)
LLM_PROFILE=local

# Cloud mode (Azure OpenAI)
LLM_PROFILE=cloud
```

| Setting | Chat model | Embedding model | ChromaDB collection |
|---------|-----------|-----------------|---------------------|
| `local` | `gemma3:e4b` via `http://localhost:11434/v1` | `nomic-embed-text` (768 dims) | `Corpus_local` |
| `cloud` | `gpt-5.4-nano` via Azure OpenAI | `text-embedding-3-small` (1536 dims) | `Corpus` |

Only the client construction and model names change. The retrieval logic, tool-call loop, and UI are identical across profiles.

**Important:** If you switch from `local` to `cloud` (or vice versa), re-run `python ingest.py` under the new profile to build the correct collection. Do not query a `cloud`-indexed collection with `local` embeddings — retrieval will silently return garbage.

---

## Cloud Setup

For `LLM_PROFILE=cloud`, set these in `.env`:

```
AZURE_OPENAI_API_KEY=<your key>
AZURE_OPENAI_ENDPOINT=https://<your-resource>.openai.azure.com/
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-5.4-nano
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
AZURE_OPENAI_API_VERSION=2024-06-01
COLLECTION=Corpus
```

Then run `python ingest.py` and `streamlit run app.py`.

---

## Hardware Tested On

| | |
|---|---|
| Machine | MacBook Pro (Apple Silicon — fill in your exact chip, e.g. M2 Pro) |
| RAM | fill in (e.g. 16 GB unified memory) |
| OS | macOS (fill in version, e.g. 14.5 Sonoma) |
| Ollama version | fill in (`ollama --version`) |
| Gemma3:e4b token throughput | fill in (tokens/sec from Ollama output) |
| nomic-embed-text throughput | fill in (chunks/sec observed during ingest) |

Ollama uses Apple Metal for GPU acceleration on Apple Silicon. E4B quantization fits comfortably in 16 GB unified memory with no swapping.

---

## Project Structure



├── agent.py          # RAG agent loop + retrieval (profile-aware)
├── app.py            # Streamlit chat UI
├── ingest.py         # Document chunking + embedding pipeline (profile-aware)
├── tools.py          # EPA Facilities API tool schema and implementation
├── test.py           # Unit tests
├── requirements.txt  # Python dependencies
├── .env              # Configuration (gitignored)
├── .env.example      # Template — copy to .env
├── data/
│   └── raw/          # Place your corpus documents here (.pdf, .txt, .md)
├── chroma_db/        # Persistent vector store (gitignored)
└── docs/
    ├── local-mode-comparison.md   # Cloud vs local benchmark results
    └── when-to-go-local.md        # Decision document







