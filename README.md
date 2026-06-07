# 🧠 Second Brain
> A unified, multimodal AI memory assistant that transforms unstructured data into an intelligent, queryable knowledge system — deployed and running on GCP.

---

## 📌 Abstract

**Second Brain** is a personalized AI-powered knowledge system designed to ingest, process, and retrieve information from diverse data sources — PDFs, Word documents, web pages, plain text, and voice recordings. By combining **Retrieval-Augmented Generation (RAG)**, persistent vector storage, real-time transcription, and an interactive neural map UI, the system lets you query your own knowledge base through natural language — effectively acting as a long-term AI memory layer.

---

## 🏗️ System Architecture

```
User Input (Text / Audio / URL / PDF / DOCX)
        ↓
FastAPI Ingestion Layer  (api.py)
        ↓
Parsing & Cleaning
  ├── pypdf       → PDF text extraction
  ├── python-docx → DOCX paragraph extraction
  ├── BeautifulSoup → URL scraping (strips nav/header/footer/scripts)
  └── GCP Speech-to-Text → Audio transcription (WEBM_OPUS / LINEAR16)
        ↓
Priority Scoring  (ai-layer.py :: score_priority)
  ├── Content signals  — urgency / task / medium keyword hits
  ├── Source type boost — LOG/AUDIO +1 | WEB -1 | PDF/DOCX neutral
  └── Recency boost    — <24h: +2 | <72h: +1 | older: 0
        ↓
Date Extraction  (extract_dates_from_text)
  — Absolute, relative, quarter, month/year, last/next weekday, seasons
        ↓
Chunking  (LlamaIndex TokenTextSplitter — 600 tokens, 60 overlap)
        ↓
Embedding  (HuggingFace all-MiniLM-L6-v2)
        ↓
ChromaDB  (PersistentClient — one collection per workspace)
        ↓
Query / Retrieval  (top-4 cosine similarity search)
        ↓
LLM Synthesis  (Groq SDK — Llama 3.3 70B Versatile, temp 0.2)
        ↓
Context-Aware Response  →  Frontend (second-brain.html)
```

---

## ⚙️ Tech Stack

| Layer | Technology |
|---|---|
| Backend framework | FastAPI + Uvicorn |
| LLM | Groq — `llama-3.3-70b-versatile` |
| Embeddings | HuggingFace `all-MiniLM-L6-v2` |
| Vector DB | ChromaDB (PersistentClient) |
| RAG framework | LlamaIndex (VectorStoreIndex, TokenTextSplitter) |
| Speech-to-Text | Google Cloud Speech-to-Text API |
| PDF parsing | pypdf |
| DOCX parsing | python-docx |
| Web scraping | BeautifulSoup4 + requests |
| Date parsing | python-dateutil |
| Frontend | Vanilla JS + Canvas API (single HTML file, PWA) |
| Deployment | GCP VM |

---

## 🚀 Features

### 🧩 Multi-Modal Data Ingestion

Five ingestion routes, all feeding the same ChromaDB pipeline:

| Method | Endpoint | What it accepts |
|---|---|---|
| File upload | `POST /api/upload` | PDF, DOCX, TXT, MD, CSV |
| Raw text | `POST /api/index` | Pre-extracted text + source metadata |
| URL scrape | `POST /api/ingest-url` | Any public webpage |
| Audio | `POST /api/ingest-audio` | MP3, WAV, OGG, WEBM (microphone recordings) |
| Chat log | Auto | Every conversation turn is re-indexed as a LOG node |

---

### 🧠 Priority Scoring System

Every ingested document is automatically scored on a 3-tier priority system and displayed as a **colour-coded ring** around its graph node.

| Priority | Ring colour | Criteria |
|---|---|---|
| 1 — HIGH | 🔴 Red | Urgency keywords (`urgent`, `deadline`, `asap`, etc.) **or** raw score ≥ 5 |
| 2 — MEDIUM | 🟡 Yellow | Task/action keywords (`meeting`, `todo`, `follow-up`, etc.) **or** raw score ≥ 2 |
| 3 — LOW | 🟢 Green | Reference material — no urgency/task signals, old docs, web articles |

Priority is also **dynamic**: every 5 queries on a node boosts its score by 1 point, allowing frequently accessed reference docs to climb to MEDIUM or HIGH over time.

---

### 🔍 Contextual Retrieval (RAG)

- Top-4 similarity search against the workspace's ChromaDB collection
- Query embedded with the same `all-MiniLM-L6-v2` model used at index time
- Retrieved chunks injected directly into the Groq system prompt
- Strict grounding: model responds with *"I could not find that in your indexed documents"* if the answer isn't in context

---

### 🕸️ Neural Knowledge Graph

The frontend renders an animated canvas-based graph where:

- **YOU** core node is the workspace anchor
- Each ingested document appears as an orbiting node labeled by file type (PDF, DOCX, NOTE, LOG, AUDIO, WEB)
- **Priority ring** (thin 1px line with gap) shows urgency at a glance — red / yellow / green
- **Semantic edges** drawn between nodes with cosine similarity ≥ 0.72 (computed live via `/api/connections`)
- Nodes drift with physics simulation (repulsion + gentle damping), zoom-to-fit on load
- Click any node to inspect metadata; right-click to delete

---

### 📅 Timeline & Calendar

- `POST /api/timeline` — synthesizes a chronological markdown timeline from all date references extracted across your documents
- `GET /api/calendar/{workspace_id}` — returns a monthly calendar view showing which dates have indexed content
- Supports date-range filtering: `from 2025-01-01 to 2025-12-31`
- Ask the chat interface directly: *"show timeline"* / *"generate timeline"*

---

### 💬 Conversational AI Interface

- `POST /api/chat` — workspace-scoped, RAG-grounded responses via Groq
- Timeline queries auto-detected and routed to `generate_timeline()` before hitting the LLM
- Every exchange persisted to a per-workspace `.jsonl` chat log and re-indexed as a LOG node so past conversations are queryable

---

### 📦 PWA Support

`second-brain.html` ships with a manifest and service worker for full Progressive Web App install — network-first fetch strategy, offline shell fallback, API calls bypass cache entirely.

---

## 🧱 Project Structure

```
Second-Brain/
│
├── ai-layer/
│   ├── ai-layer.py          # Core AI engine: parsers, embeddings, RAG, priority scoring
│   ├── api.py               # FastAPI server — all HTTP endpoints
│   ├── gcp_key.json         # GCP service account key (not committed)
│   ├── .env                 # API keys and config (not committed)
│   ├── .env.example         # Template
│   └── chat_logs/           # Per-workspace .jsonl conversation logs
│
├── chroma_db/               # Persistent ChromaDB vector storage
│
├── second-brain.html        # Frontend — single-file PWA (canvas graph + chat + ingestion)
├── manifest.json            # PWA manifest
├── service-worker.js        # PWA service worker
│
├── assets/
│   └── architecture.png
│
├── requirements.txt
└── README.md
```

---

## 🔐 Environment Setup

Create a `.env` file in `ai-layer/`:

```env
GROQ_API_KEY=your_groq_api_key
GOOGLE_APPLICATION_CREDENTIALS=./gcp_key.json
DATABASE_PATH=./chroma_db
HOST=0.0.0.0
PORT=8000
```

Place your GCP service account JSON key at `ai-layer/gcp_key.json`. The path is resolved relative to `ai-layer.py` regardless of working directory.

---

## ⚡ Installation

```bash
git clone https://github.com/your-username/Second-Brain.git
cd Second-Brain/ai-layer
pip install -r requirements.txt
```

---

## ▶️ Running the Project

```bash
cd ai-layer
python api.py
```

Then open:
```
http://127.0.0.1:8000
```

Swagger UI (auto-generated API docs):
```
http://127.0.0.1:8000/docs
```

---

## 🌐 API Reference

### Ingestion

| Method | Endpoint | Body / Params | Description |
|---|---|---|---|
| `POST` | `/api/upload` | `form: workspace_id, file` | Upload PDF / DOCX / TXT |
| `POST` | `/api/index` | `{workspace_id, text, source_id, source_type}` | Index pre-extracted text |
| `POST` | `/api/ingest-url` | `{workspace_id, url}` | Scrape and index a URL |
| `POST` | `/api/ingest-audio` | `form: workspace_id, file` | Transcribe audio and index |

### Query & Chat

| Method | Endpoint | Body / Params | Description |
|---|---|---|---|
| `POST` | `/api/chat` | `{workspace_id, message, app_identity?, custom_rules?}` | RAG-grounded chat |
| `POST` | `/api/timeline` | `{workspace_id, start_date?, end_date?}` | Generate markdown timeline |
| `GET` | `/api/calendar/{workspace_id}` | `?month=&year=` | Monthly calendar of indexed dates |

### Graph & Nodes

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/nodes/{workspace_id}` | List all nodes with priority + query count |
| `GET` | `/api/connections/{workspace_id}` | Pairwise cosine similarity edges (threshold 0.60) |
| `POST` | `/api/nodes/{workspace_id}/increment` | Increment query count for a node |
| `GET` | `/api/chatlog/{workspace_id}` | Retrieve last N chat log entries |

### Management

| Method | Endpoint | Description |
|---|---|---|
| `DELETE` | `/api/node/{workspace_id}?source_id=` | Delete a single node (or clear chat log) |
| `DELETE` | `/api/workspace/{workspace_id}` | Nuke entire workspace + ChromaDB collection |
| `GET` | `/api/health` | Service health check |
| `GET` | `/api/info` | API version and endpoint listing |

---

## 📈 Future Enhancements

- 🔊 Text-to-Speech — close the full voice loop (speak queries, hear responses)
- 🧾 Source citations — surface which document chunks grounded each response
- 🧠 Memory summarization — compress old chat logs into condensed embeddings
- 🔒 Auth layer — multi-user workspace isolation
- ☁️ Cloud Run deployment — containerized, autoscaled GCP hosting
- 🔗 Browser extension — ingest pages directly from Chrome

---

## 👥 Team

Built by **Mizin Sadikh**, **Diya**, **Achsa**, and **Vargeese**
Guided by **Basil Scaria**
Muthoot Institute of Technology and Science, Kochi
