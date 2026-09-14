# Enterprise IT Support Agentic RAG Copilot

An end-to-end **Forward Deployed Engineer (FDE) style project** that turns a notebook-style Agentic RAG idea into a deployable internal product, using **LangGraph, FastAPI, Pinecone, Groq, Tavily, HTML/CSS/JavaScript, and Docker**.

---

## 1. Problem This Solves

Internal IT teams maintain a lot of scattered documentation — VPN instructions, password/MFA policy, software access rules, laptop troubleshooting guides, service-desk runbooks. Employees still file repetitive support tickets because:

- They don't know which document has the answer.
- Keyword search returns too much noise.
- A plain chatbot can hallucinate a policy that doesn't exist.
- Internal docs can be incomplete, outdated, or simply silent on a topic.
- Some questions need current, external information (e.g. a live vendor outage) that no internal doc will ever contain.

**Example — should stay internal:**
> "How do I connect to the company VPN from home?"
The answer lives in the private KB, so the system should never hit the public internet for this.

**Example — should fall back to the web:**
> "What's the latest Microsoft Teams outage guidance?"
The internal KB has no current outage info, so the system should recognize that its private evidence is weak, search the web instead, grade that evidence too, and clearly flag the answer as external.

### Goal

Build an IT support copilot that:

1. Searches trusted private knowledge first.
2. Grades whether that evidence is actually good enough to answer from.
3. Falls back to web search only when the private KB is insufficient.
4. Rewrites a weak query and retries (bounded, so it can't loop forever) before giving up.
5. Generates an answer grounded only in the evidence it retrieved.
6. Returns the full decision trace, not just a final answer, for transparency and debugging.
7. Lets an authorized admin add new documents to the knowledge base without touching code.

---

## 2. Why This Is an FDE-Style Project

A Forward Deployed Engineer doesn't stop at an LLM notebook — they turn a customer's problem into something deployable, observable, and safe to hand to real users. This repo walks the same path:

```
Problem definition
      ↓
Solution architecture
      ↓
Knowledge integration (ingestion pipeline)
      ↓
Agentic RAG logic (routing, grading, retries)
      ↓
API layer (FastAPI)
      ↓
User-facing UI
      ↓
Security + audit logging
      ↓
Containerized deployment (Docker)
```

---

## 3. Architecture

```
                   ┌─────────────────────┐
                   │   Employee / User   │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ HTML / CSS / JS     │
                   │      Web UI         │
                   └──────────┬──────────┘
                              │ POST /api/chat
                              ▼
                   ┌─────────────────────┐
                   │       FastAPI       │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │     LangGraph       │
                   │ Agentic RAG control │
                   └──────────┬──────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
       ┌───────────────┐              ┌──────────────┐
       │ Private KB    │              │ Tavily Web   │
       │ (Pinecone)    │              │ Search       │
       └───────┬───────┘              └──────┬───────┘
               │                             │
               └──────────────┬──────────────┘
                              ▼
                     ┌────────────────┐
                     │   Groq LLM     │
                     │ Grounded Answer│
                     └────────────────┘
```

---

## 4. Agentic RAG Workflow

Plain RAG does roughly: `Question → Retrieve → Generate`. This project makes decisions at every step instead:

```
Question
   │
   ▼
[1] Route question
   ├── Greeting / small talk ───────────────► Direct answer (no retrieval)
   │
   └── IT support question
                │
                ▼
[2] Retrieve from private Pinecone KB
                │
                ▼
[3] Grade KB evidence
       ┌────────┴────────┐
      GOOD               WEAK
       │                 │
       ▼                 ▼
Generate from KB    [4] Tavily web search
                         │
                         ▼
                  [5] Grade web evidence
                    ┌────┴─────┐
                   GOOD        WEAK
                    │           │
                    ▼           ▼
              Generate      [6] Rewrite query
              from web           │
                                 ▼
                          Retry private KB
                                 │
                                 ▼
                       Retries exhausted?
                                 │
                                 ▼
                     "Insufficient evidence" (safe fallback,
                      routes the employee to the help desk)
```

10 LangGraph nodes total: `route_question`, `retrieve_kb`, `grade_kb`, `search_web`, `grade_web`, `rewrite_query`, `generate_from_kb`, `generate_from_web`, `direct_answer`, `insufficient`.

Every terminal node returns the answer, the source actually used (`private_kb` / `web_search` / `direct` / `insufficient_evidence`), the full trace, and citations — so nothing is a black box.

---

## 5. Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent workflow | LangGraph | Stateful routing and conditional decisions |
| LLM | Groq (`langchain-groq`) | Routing, grading, query rewriting, answer generation |
| Embeddings | FastEmbed — `sentence-transformers/all-MiniLM-L6-v2` | Local embeddings, no external embedding API |
| Vector DB | Pinecone (serverless) | Private enterprise knowledge base |
| External search | Tavily | Fallback when the company KB is insufficient |
| API | FastAPI + Uvicorn | Backend and REST endpoints |
| Frontend | HTML / CSS / JavaScript (Jinja2-rendered) | Employee-facing chat UI |
| Document parsing | pypdf, python-docx | PDF / DOCX ingestion |
| Audit | SQLite | Per-query decision-path logging |
| Packaging | Docker | Reproducible deployment |

---

## 6. Project Structure

```
Enterprise-IT-Support-Agentic-RAG-Copilot/
│
├── app/
│   ├── api/
│   │   └── routes.py          # /api/health, /api/chat, /api/ingest
│   │
│   ├── core/
│   │   ├── config.py          # environment-driven settings
│   │   └── logging.py         # logging setup
│   │
│   ├── rag/
│   │   ├── state.py           # LangGraph state + structured decision schemas
│   │   ├── vectorstore.py     # Pinecone index + retriever, dynamic dimension handling
│   │   └── workflow.py        # the full agentic RAG graph
│   │
│   ├── services/
│   │   ├── audit.py           # SQLite query audit logging
│   │   └── ingestion.py       # PDF/TXT/MD/DOCX loading + chunking
│   │
│   └── main.py                # FastAPI application entrypoint
│
├── data/
│   ├── sample_kb/              # sample IT knowledge base docs
│   └── audit.db                 # SQLite audit log
│
├── static/
│   ├── css/style.css
│   └── js/app.js
│
├── templates/
│   └── index.html
│
├── uploads/                    # documents uploaded via /api/ingest
├── Dockerfile
├── ingest_sample_kb.py         # bulk-index the sample KB
├── run.py                      # local dev entrypoint
├── requirements.txt
└── README.md
```

---

## 7. Setup

### Step 1 — Create a virtual environment

```bash
python -m venv venv
```

Windows: `venv\Scripts\activate`
macOS/Linux: `source venv/bin/activate`

### Step 2 — Install dependencies

```bash
pip install -r requirements.txt
```

### Step 3 — Configure environment

Create a `.env` file in the project root:

```env
# LLM
GROQ_API_KEY=your_groq_api_key_here
GROQ_MODEL=openai/gpt-oss-20b

# External web search
TAVILY_API_KEY=your_tavily_api_key_here

# Vector database
PINECONE_API_KEY=your_pinecone_api_key_here
PINECONE_INDEX_NAME=fde-it-support-rag
PINECONE_NAMESPACE=company-it-kb

# Embeddings
EMBEDDING_MODEL=all-MiniLM-L6-v2

# Retrieval / retry behavior
TOP_K=4
MAX_RETRIES=1

# Security
ADMIN_API_KEY=change-me-in-production

# App
APP_ENV=development
```

#### Environment Variable Reference

| Variable | Description | Required |
|---|---|---|
| `GROQ_API_KEY` | Groq API key — powers routing, grading, rewriting, generation | ✅ Yes |
| `TAVILY_API_KEY` | Tavily API key — web search fallback | ✅ Yes |
| `PINECONE_API_KEY` | Pinecone API key — private vector store | ✅ Yes |
| `PINECONE_INDEX_NAME` | Pinecone index name | ⚠️ Optional (default: `fde-it-support-rag`) |
| `PINECONE_NAMESPACE` | Pinecone namespace, isolates document sets | ⚠️ Optional (default: `company-it-kb`) |
| `GROQ_MODEL` | Groq model identifier | ⚠️ Optional (default: `openai/gpt-oss-20b`) |
| `EMBEDDING_MODEL` | FastEmbed / sentence-transformers model name | ⚠️ Optional (default: `all-MiniLM-L6-v2`) |
| `TOP_K` | Number of chunks retrieved per query | ⚠️ Optional (default: `4`) |
| `MAX_RETRIES` | Max query-rewrite retries before giving up | ⚠️ Optional (default: `1`) |
| `ADMIN_API_KEY` | Secret required by `/api/ingest` | ⚠️ Optional (default: `change-me`) |
| `APP_ENV` | `development` or `production` | ⚠️ Optional (default: `development`) |

**Security notes:** never commit `.env`; change `ADMIN_API_KEY` before deploying; keep API keys scoped to only the services that need them.

### Step 4 — Load the sample knowledge base

```bash
python ingest_sample_kb.py
```

Ingestion pipeline: `documents → load → chunk (RecursiveCharacterTextSplitter) → FastEmbed embeddings → Pinecone`.

### Step 5 — Run

```bash
python run.py
```

App: `http://127.0.0.1:8080`
API docs: `http://127.0.0.1:8080/docs`

### Run with Docker instead

```bash
docker build -t it-support-copilot .
docker run -p 8080:8080 --env-file .env it-support-copilot
```

---

## 8. Demo Scenarios

**A — Private KB success**
> "How do I connect to the company VPN from home?"
Path: `Router → KB → grade GOOD → generate_from_kb`. Trusted internal knowledge answers directly, no web call.

**B — Policy question**
> "Can IT support ask me to share my MFA code?"
Path: `Router → KB → grade GOOD → generate_from_kb`. Shows RAG answering from company-specific knowledge the model was never trained on.

**C — External / current information**
> "What's the latest Microsoft Teams outage guidance?"
Path: `Router → KB → grade WEAK → search_web → grade GOOD → generate_from_web`. Shows the agent picking a different source instead of forcing an answer from irrelevant KB chunks.

**D — Query rewrite**
> "My work communication app is acting strange after the update. What should I do?"
If both KB and the first web pass are weak, the query gets rewritten and retried (bounded by `MAX_RETRIES`) before falling back to "insufficient evidence."

**E — Direct conversation**
> "Hello!"
Path: `Router → direct_answer`. Not every message should trigger vector or web search.

---

## 9. Document Upload

`POST /api/ingest` (multipart file upload, `X-Admin-Key` header required) accepts `.pdf`, `.txt`, `.md`, `.docx`.

```
Upload → validate type → load text → chunk → FastEmbed embeddings → Pinecone index → immediately retrievable
```

This matters in practice: a real IT team doesn't want to edit Python every time a new policy doc is published.

---

## 10. API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/health` | Health check |
| `POST` | `/api/chat` | Ask a question |
| `POST` | `/api/ingest` | Upload + index a document (admin-key protected) |

**`POST /api/chat`**

Request:
```json
{ "question": "How do I reset my company password?" }
```

Response:
```json
{
  "answer": "...",
  "source_used": "private_kb",
  "trace": [
    "Router → KB",
    "Private KB retrieval → 4 chunks",
    "KB evidence grade → GOOD",
    "Answer generation → PRIVATE KB"
  ],
  "citations": [],
  "rewritten_query": "How do I reset my company password?"
}
```

---

## License

MIT — see [LICENSE](LICENSE).
