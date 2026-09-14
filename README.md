# Enterprise IT Support Agentic RAG Copilot

A self-correcting, agentic RAG (Retrieval-Augmented Generation) copilot for enterprise IT support. It answers employee IT questions (VPN, MFA, password resets, software access, device setup, security policy) from a private company knowledge base, and automatically falls back to live web search — with evidence grading and query rewriting at every step — when the internal KB isn't good enough.

## Features

- **Agentic routing** — classifies each question as a KB lookup or a direct/casual reply, so greetings and small talk skip retrieval entirely.
- **Corrective RAG loop** — a 10-node LangGraph workflow that retrieves from the private KB, grades the evidence with an LLM judge, and only answers when confident.
- **Automatic web fallback** — if private KB evidence is weak, the agent searches the web (Tavily), grades that evidence too, and rewrites the query and retries (up to a configurable retry limit) before answering.
- **Transparent reasoning trace** — every response returns the step-by-step trace (router → retrieval → grading → generation) plus citations, so answers are auditable, not a black box.
- **Document ingestion API** — admin-authenticated endpoint to upload PDF / DOCX / TXT / MD files, which are chunked and embedded into a Pinecone index for retrieval.
- **Query audit logging** — every question, its source (`private_kb` / `web_search` / `direct` / `insufficient_evidence`), and its full trace are persisted to SQLite for compliance/review.
- **Simple web UI** — a lightweight chat interface (Jinja2 + vanilla JS) for asking questions and viewing sources.

## Architecture

```
User question
     │
     ▼
 route_question  ──► direct_answer (greetings/small talk)
     │
     ▼ (IT/support question)
 retrieve_kb (Pinecone)
     │
     ▼
  grade_kb ──good──► generate_from_kb
     │
    weak
     ▼
 search_web (Tavily)
     │
     ▼
  grade_web ──good──► generate_from_web
     │
    weak (retries left)          weak (no retries left)
     ▼                                   ▼
 rewrite_query ──► retrieve_kb      insufficient (safe fallback message)
```

Every terminal node returns an answer, the source used, a full execution trace, and citations.

## Tech Stack

| Layer | Technology |
|---|---|
| API / Web server | FastAPI, Uvicorn, Jinja2 |
| Agent orchestration | LangGraph, LangChain |
| LLM inference | Groq (`langchain-groq`) |
| Vector database | Pinecone (serverless) |
| Embeddings | FastEmbed (`sentence-transformers/all-MiniLM-L6-v2`) |
| Web search | Tavily |
| Document parsing | pypdf, python-docx |
| Audit storage | SQLite |
| Deployment | Docker |

## Project Structure

```
app/
├── api/routes.py          # /api/health, /api/chat, /api/ingest
├── core/config.py         # settings (env-driven)
├── core/logging.py        # logging setup
├── rag/workflow.py        # LangGraph agent graph (nodes + edges)
├── rag/state.py           # agent state schema
├── rag/vectorstore.py     # Pinecone index + retriever
├── services/ingestion.py  # file loading + chunking
├── services/audit.py      # SQLite audit logging
└── main.py                 # FastAPI app entrypoint
static/                     # CSS/JS for the chat UI
templates/index.html        # chat UI page
data/sample_kb/             # sample IT knowledge base docs
ingest_sample_kb.py         # script to bulk-index the sample KB
run.py                      # local dev entrypoint
Dockerfile
```

## Getting Started

### Prerequisites

- Python 3.11+
- API keys: [Groq](https://console.groq.com), [Tavily](https://tavily.com), [Pinecone](https://www.pinecone.io)

### Installation

```bash
git clone <repo-url>
cd Enterprise-IT-Support-Agentic-RAG-Copilot
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Configuration

Create a `.env` file in the project root:

```env
APP_NAME=Enterprise IT Support Agentic RAG Copilot
APP_ENV=development

GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=openai/gpt-oss-20b

TAVILY_API_KEY=your_tavily_api_key

PINECONE_API_KEY=your_pinecone_api_key
PINECONE_INDEX_NAME=fde-it-support-rag
PINECONE_NAMESPACE=company-it-kb

EMBEDDING_MODEL=all-MiniLM-L6-v2
TOP_K=4
MAX_RETRIES=1

ADMIN_API_KEY=change-me
```

### Index the sample knowledge base

```bash
python ingest_sample_kb.py
```

### Run locally

```bash
python run.py
```

The app will be available at `http://127.0.0.1:8080`.

### Run with Docker

```bash
docker build -t it-support-copilot .
docker run -p 8080:8080 --env-file .env it-support-copilot
```

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/health` | Health check |
| `POST` | `/api/chat` | Ask a question — body: `{ "question": "How do I reset my VPN password?" }` |
| `POST` | `/api/ingest` | Upload and index a document — multipart file upload, requires `X-Admin-Key` header |

**Example:**

```bash
curl -X POST http://127.0.0.1:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I set up MFA?"}'
```

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
