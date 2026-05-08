# 🌑 Singularity

> **Production-grade, multi-modal RAG system running entirely on Google Colab T4 GPU — no paid infra, no cloud APIs, no compromises.**

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/true-vision-singularity-rag/blob/main/OFFICIAL_True_Vision_Singularity_RAG_FIXEDX.ipynb)
![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688)
![Streamlit](https://img.shields.io/badge/Frontend-Streamlit-FF4B4B)
![License](https://img.shields.io/badge/License-MIT-green)

---

## What Is This?

Singularity RAG is an enterprise-style **Retrieval-Augmented Generation** system built from scratch — hybrid retrieval, a vision pipeline, a knowledge graph, session memory, and a full observability layer — all inside a single Colab notebook, tunnelled to the public internet via ngrok.

It was designed to answer the question: *how close can a single free T4 GPU get to a production-grade NotebookLM-style system?* The answer is: surprisingly close.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Streamlit UI (app.py)               │
│         Upload · Chat · History · Debug Tabs            │
└────────────────────────┬────────────────────────────────┘
                         │  SSE / REST
┌────────────────────────▼────────────────────────────────┐
│                  FastAPI Backend (main.py)               │
│    /upload  /chat  /sessions  /feedback  /health        │
└──┬─────────┬───────────┬──────────────┬─────────────────┘
   │         │           │              │
   ▼         ▼           ▼              ▼
ingestion  retrieval   llm.py      knowledge_graph.py
   │         │           │
   ▼         ▼           ▼
FAISS     BM25 +      Llama 3.2:3b
(dense)   FAISS        via Ollama
          hybrid
             │
             ▼
        Cross-Encoder
         Reranker
             │
             ▼
       memory.py
  (reflection + decay)
```

**Ingestion → Retrieval → Reranking → Generation → Memory** — every stage is observable via `tracer.py`.

---

## Core Features

### Retrieval
- **Hybrid BM25 + FAISS** dense retrieval fused with **Reciprocal Rank Fusion (RRF)**
- **Cross-encoder reranking** (`ms-marco-MiniLM-L-6-v2`) for precision at top-k
- **Query rewriting** via LLM with hard timeout enforcement (toggle in UI)
- **Confidence scoring** pipeline across the full retrieval chain

### Document Ingestion
- Supports: **PDF, DOCX, TXT, CSV, XLSX, PPTX, Markdown, HTML**
- Superior PDF table extraction via **pdfplumber** (markdown table format)
- DOCX in-order table placement with **caption matching** (solves "Table IV" queries)
- **Author / frontmatter** tagging with reranker score boost
- SHA-256 based deduplication across documents

### Vision Pipeline
- **BLIP image captioning** (`Salesforce/blip-image-captioning-base`) — FP16 on T4
- Extracts embedded images from PDFs and indexes their captions as searchable chunks
- Graceful fallback to placeholder captions on VRAM pressure

### Memory & Knowledge Graph
- **Reflection layer** — self-evaluates answer quality, confidence, gap detection, intent trajectory
- **Temporal memory** — timestamp-aware retrieval, date-range filtering, recency preference
- **Forgetting/decay layer** — decay functions, relevance pruning, deduplication over time
- **Knowledge graph** — entity extraction → graph nodes/edges → multi-hop traversal for context enrichment
  - Entity types: `PERSON`, `ORG`, `CONCEPT`, `METRIC`, `DATE`, `LOCATION`, `PRODUCT`, `EVENT`
  - JSON-persisted, no external DB required

### Observability
- Full **TraceLogger** (`tracer.py`) covering every pipeline stage
- ANSI colour output in Colab + JSON trace files on disk
- 6-tab **ipywidgets debugger** in Cell 8: live traces, chunk inspection, graph viewer, memory stats

### Infrastructure
- **Llama 3.2:3b** via **Ollama** (local, no API key)
- **all-MiniLM-L6-v2** sentence embeddings via `sentence-transformers` (GPU batch=128)
- **FAISS** vector store (CPU, avoids VRAM pressure from embedding + LLM co-load)
- **ngrok** tunnels for both UI (`:8501`) and API (`:8000`) — public HTTPS URLs
- SSE streaming for real-time token delivery

---

## File Structure

```
OFFICIAL_True_Vision_Singularity_RAG_FIXEDX.ipynb   ← Single notebook, run top-to-bottom

/content/singularity/           ← Written at runtime by Cell 5
├── ingestion.py                ← Multi-format parser, BLIP vision, FAISS store
├── retrieval.py                ← Hybrid BM25+FAISS, RRF, cross-encoder reranker
├── llm.py                      ← Ollama LLM, query rewriting, streaming
├── memory.py                   ← Reflection, temporal memory, decay/pruning
├── knowledge_graph.py          ← Entity extraction, graph traversal
├── main.py                     ← FastAPI backend (all endpoints)
├── app.py                      ← Streamlit frontend
├── tracer.py                   ← Pipeline observability / trace logger
├── uploads/                    ← Uploaded documents (session-scoped)
├── sessions/                   ← Chat history per session
├── feedback/                   ← User feedback logs
├── vector_store/               ← FAISS index + metadata.json
├── doc_meta/                   ← Per-document metadata
└── knowledge_graph/            ← Persisted graph JSON
```

---

## Quick Start

### 1. Open in Colab
Click the badge at the top, or manually upload `OFFICIAL_True_Vision_Singularity_RAG_FIXEDX.ipynb`.

### 2. Set runtime
`Runtime → Change runtime type → T4 GPU`

### 3. Get a free ngrok token
Sign up at [dashboard.ngrok.com](https://dashboard.ngrok.com/get-started/your-authtoken) (free, no card required), copy your authtoken.

### 4. Paste your token
In **Cell 7**, replace the placeholder token with your own:
```python
ngrok.set_auth_token('YOUR_TOKEN_HERE')
```

### 5. Run all cells
`Runtime → Run all`

The final cell prints two URLs:
```
🌐  OPEN THIS  →  https://xxxx.ngrok-free.app   ← Streamlit UI
🔌  API        →  https://yyyy.ngrok-free.app   ← FastAPI docs
```

---

## Cell Breakdown

| Cell | What it does | Time |
|------|-------------|------|
| 1 | Verify T4 GPU + CUDA | ~5s |
| 2 | Install all pip dependencies | ~3 min (cached) |
| 3 | Install Ollama + pull Llama 3.2:3b | ~2 min |
| 4 | Create `/content/singularity/` directory tree | ~2s |
| 5 | Write all 7 Python source files to disk | ~5s |
| 6 | Syntax-check all source files | ~5s |
| 7 | Set ngrok authtoken + open tunnels | ~10s |
| 8 | 🚀 Launch FastAPI + Streamlit + ipywidgets debugger | — |
| 9 | (Optional) Health check + GPU stats | anytime |

---

## Requirements

| Requirement | Notes |
|-------------|-------|
| Google Colab (free tier) | T4 GPU runtime required |
| ngrok account (free) | For public tunnel URLs |
| ~6 GB VRAM | T4 has 15 GB — comfortable headroom |
| ~4 GB disk | Ollama model + pip packages |

No OpenAI API key. No cloud storage. No paid services.

---

## Supported Document Formats

| Format | Tables | Images | Notes |
|--------|--------|--------|-------|
| PDF | ✅ pdfplumber | ✅ BLIP captions | Heading detection via font size/bold flags |
| DOCX | ✅ inline placement | ✅ BLIP captions | Caption-matched table boost in reranker |
| TXT / MD | — | — | Chunked directly |
| CSV | ✅ structured rows | — | pandas-based |
| XLSX | ✅ multi-sheet | — | openpyxl |
| PPTX | ✅ slide tables | — | python-pptx, per-slide sections |
| HTML | — | — | BeautifulSoup text extraction |

---

## Known Limitations

- **Session duration**: Free Colab disconnects after ~12h of inactivity. All uploaded files live in `/content/` and are lost on disconnect.
- **Concurrent users**: Single-server setup — not designed for high concurrency.
- **VRAM**: BLIP + LLM cannot co-exist in VRAM simultaneously on T4; vision model unloads LLM temporarily during image captioning (ingestion-time only).
- **Ollama on Colab**: Occasionally the Ollama server takes longer than 30s to start on first boot — rerun Cell 3 if this happens.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| LLM | Llama 3.2:3b via Ollama |
| Embeddings | all-MiniLM-L6-v2 (sentence-transformers) |
| Dense retrieval | FAISS IndexFlatIP |
| Sparse retrieval | BM25 (rank_bm25) |
| Fusion | Reciprocal Rank Fusion |
| Reranker | ms-marco-MiniLM-L-6-v2 (cross-encoder) |
| Vision | BLIP image captioning (Salesforce, HuggingFace) |
| PDF parsing | PyMuPDF + pdfplumber |
| Backend | FastAPI + uvicorn |
| Frontend | Streamlit |
| Tunnelling | pyngrok |
| GPU | NVIDIA T4 (Google Colab) |

---

## Contributing

PRs welcome. If you're extending the vision pipeline, memory layer, or adding new document parsers — open an issue first so we can align on the architecture.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---
