# RAG Document Q&A System

A Retrieval-Augmented Generation system that lets users upload documents (PDF, DOCX, TXT) and ask questions about them in natural language. Answers are generated from the document contents and returned with cited source chunks and relevance scores.

Built with FastAPI, React, Pinecone, PostgreSQL, and Google Gemini.

- **Live demo:** https://rag-document-qa-system.vercel.app
- **Write-up:** [I Built a Production RAG System in 3 Weeks — Here's What Actually Broke](https://dev.to/blessedefem/i-built-a-production-rag-system-in-3-weeks-heres-what-actually-broke-556m)

## Architecture

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │ HTTPS
       ▼
┌──────────────────────┐
│  React + TypeScript  │  (Vercel)
│  Clerk Auth          │
└────────┬─────────────┘
         │ REST API + JWT
         ▼
┌──────────────────────┐
│  FastAPI Backend     │  (Railway)
│  + Background Tasks  │
└────┬──────┬──────┬───┘
     │      │      │
     ▼      ▼      ▼
┌─────────┐┌──────────┐┌──────────┐
│Pinecone ││PostgreSQL││VirusTotal│
│(Vectors)││(Metadata)││(optional)│
└────┬────┘└──────────┘└──────────┘
     │
     ▼
┌──────────────────────┐
│ Gemini (primary)     │
│ Cohere (fallback)    │
└──────────────────────┘
```

### How a query flows

1. User uploads a document → backend extracts the text → splits it into 1,000-character chunks with 100-character overlap.
2. Each chunk is embedded with Gemini's `gemini-embedding-001` model (768 dimensions). If Gemini fails, the system falls back to Cohere (see [Known Limitations](#known-limitations)).
3. Embeddings are stored in Pinecone with metadata (filename, chunk index, user ID).
4. User asks a question → the question is embedded → Pinecone returns the most similar chunks by cosine similarity.
5. The top chunks (up to 5) and the question are sent to Gemini (`gemini-2.5-flash` on the live deployment, set by `GEMINI_MODEL`) → it generates an answer with source citations.

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React 18 + TypeScript, Vite, Tailwind + shadcn/ui | Type safety, fast builds, consistent UI components |
| Backend | FastAPI (Python 3.11) | Async support, automatic OpenAPI docs, Pydantic validation |
| Database | PostgreSQL + SQLAlchemy | Relational integrity for document metadata, per-user isolation |
| Vector DB | Pinecone | Managed vector search, no infrastructure to run |
| Embeddings | Gemini `gemini-embedding-001` (primary), Cohere (fallback) | Keeps uploads working when Gemini fails or is rate-limited |
| LLM | Gemini (`gemini-2.5-flash`, configurable via `GEMINI_MODEL`) | Fast responses, free tier |
| Auth | Clerk | JWT-based; handles signup, login and sessions |
| Caching | Redis (optional) | Caches document lists when configured |
| File scanning | VirusTotal (optional) | Scans uploads before processing when an API key is set |
| Deployment | Railway (backend) + Vercel (frontend) | Deploys on every push to GitHub |

## Limits

| Setting | Value |
|---|---|
| Max file size | 10 MB |
| Chunk size / overlap | 1,000 / 100 characters |
| Max chunks per document | 200 (longer documents are truncated, to protect API quotas) |
| Chunks sent to the LLM per question | Up to 5 |
| Embedding dimensions | 768 |
| API rate limit | 200 requests/minute per IP |

## Measured Performance

Measured on the live deployment from Lagos with browser DevTools (Nov 2026, small sample, no caching):

| Request | Time |
|---|---|
| List documents (7 documents) | 0.6–1.2 s |
| Ask a question → answer | 4.2–4.7 s |
| Upload small document → status "ready" | ~6 s |

Each request also makes a CORS preflight (`OPTIONS`) of about 0.4 s.

## Key Technical Decisions

### Why a fallback embedding provider?

Gemini's free tier has rate limits. Instead of failing the upload when Gemini errors or is rate-limited, the embedding service retries with Cohere. Trade-off: Cohere's vectors are cut to 768 dimensions to fit the same Pinecone index, and vectors from two different models aren't directly comparable (see [Known Limitations](#known-limitations)).

### Why chunking with overlap?

Fixed-size chunks (1,000 characters, 100 overlap) balance retrieval precision against context completeness. The overlap reduces the chance that an answer is split across a chunk boundary and lost.

### Why background processing?

Extracting, chunking and embedding a large document takes several seconds. Doing it inside the request would block the API and risk timeouts, so the upload returns immediately with a "processing" status and the pipeline runs as a FastAPI background task.

### Why optional virus scanning?

Any system that accepts file uploads needs a security layer. When `VIRUSTOTAL_API_KEY` is set, every upload is checked against VirusTotal before processing, and the scanner is fail-closed: a scan error rejects the upload. Without a key, `VIRUS_SCAN_REQUIRED` decides: `true` blocks uploads, `false` (the default) skips scanning. **The live demo currently runs with scanning off.**

## Security

- **Authentication:** Clerk JWT verification with user isolation — users can only access their own documents
- **File scanning:** optional VirusTotal integration, fail-closed when enabled (off on the live demo)
- **Rate limiting:** 200 requests/minute per IP (slowapi)
- **Input validation:** file type and size checks
- **SQL injection prevention:** SQLAlchemy ORM with parameterised queries
- **CORS:** only whitelisted origins (`ALLOWED_ORIGINS`)
- **Secrets:** HTTP client request logs are kept at WARNING level so API keys in request URLs never reach the logs

## API Reference

### Upload Document

```http
POST /api/documents/upload
Authorization: Bearer <clerk_jwt_token>
Content-Type: multipart/form-data

file: <binary>
```

### Ask Question

```http
POST /api/documents/answer
Authorization: Bearer <clerk_jwt_token>
Content-Type: application/json

{
  "query": "What is machine learning?",
  "top_k": 5,
  "min_score": 0.3,
  "document_id": 123
}
```

`document_id` is optional — omit it to search all your documents.

### List Documents

```http
GET /api/documents/list?skip=0&limit=100
Authorization: Bearer <clerk_jwt_token>
```

### Delete Document

```http
DELETE /api/documents/{document_id}
Authorization: Bearer <clerk_jwt_token>
```

Full interactive API docs are available at `/api/docs` when running locally.

## Local Development

### Prerequisites

- Python 3.11+
- Node.js 18+
- PostgreSQL 14+
- API keys: Gemini, Pinecone, Clerk (optional: Cohere, VirusTotal, Redis)

### Backend

```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r backend/requirements.txt
cd backend
cp .env.example .env      # Add your API keys
uvicorn app.main:app --reload --port 8000
```

### Frontend

```bash
cd frontend
npm install
echo "VITE_API_URL=http://localhost:8000" > .env
echo "VITE_CLERK_PUBLISHABLE_KEY=your_clerk_key" >> .env
npm run dev
```

### Pinecone Setup

1. Create a Pinecone account → create an index named `rag-documents`
2. Dimensions: **768**, metric: **cosine**
3. Add `PINECONE_API_KEY` to your `.env`

## Tests

```bash
cd backend
pytest tests/ -v --cov=app
```

18 tests covering authentication, document upload/list/delete, question answering, background processing and error handling. 54% code coverage.

## Known Limitations

- **Mixed embedding models:** if Gemini fails, Cohere embeddings are truncated to 768 dimensions and stored in the same index as Gemini embeddings. Vectors from different models aren't comparable, so search quality for those documents is unreliable. Better: a separate index per model, or re-embedding with one model.
- **Background tasks die on redeploy:** processing runs in-process, so a redeploy mid-upload can leave a document stuck at "Processing". Better: a real job queue (e.g. Redis + RQ) and marking documents "Failed" on error.
- **VirusTotal unknown files:** when scanning is enabled, files VirusTotal has never seen are rejected. Most private documents are unknown to VirusTotal, so this is too strict for real users. Better: submit the file for analysis instead of only looking up its hash.
- **Empty-file test:** `test_upload_empty_file` fails when an invalid VirusTotal key is configured locally — the scanner error is caught by a broad `except` and returned as a 500 instead of a 400.
- **No retry when Gemini is overloaded:** a Gemini `503 UNAVAILABLE` ("high demand") fails the question immediately and the raw provider error is shown to the user. Better: retry with backoff, fall back to another model, and show a friendly message.
- Documents over 200 chunks are truncated.
- Scanned PDFs without embedded text aren't supported (no OCR).
- No conversation memory — each question is independent.

## Project Structure

```
rag-document-qa-system/
├── backend/
│   ├── app/
│   │   ├── router/         # API endpoints
│   │   ├── services/       # Chunking, embedding, Q&A, scanning, caching
│   │   ├── middleware/     # Auth, logging
│   │   ├── models.py       # SQLAlchemy models
│   │   └── main.py         # FastAPI entry point (CORS, rate limiting)
│   ├── tests/              # pytest suite
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── components/     # React components
│   │   ├── pages/          # Route pages
│   │   └── services/       # API client
│   └── package.json
└── README.md
```

## License

MIT
