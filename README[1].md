# Mini RAG (Track B) — "Al Engineer Assessment"

**Live stack**: Next.js (frontend/API) · Pinecone (hosted vector DB) · OpenAI (embeddings + LLM) · Cohere Rerank  
**Status**: Ready to deploy to Vercel.  
**Date**: 2025-09-04

## Goal
A small RAG app: users paste/upload text; we chunk + embed; store in a cloud vector DB; retrieve + rerank; answer with an LLM and show inline citations mapping to source snippets.

---

## 1) Vector Database (Hosted)
- **Provider**: Pinecone (Serverless)
- **Index name**: `mini-rag` (configurable via `PINECONE_INDEX`)
- **Dimensionality**: **1536** (for `text-embedding-3-small`)
- **Upsert strategy**: Deterministic chunk IDs in the form `{source}::{position}`. Re-ingest of the same source will overwrite chunks with the same IDs; suitable for incremental updates.

## 2) Embeddings & Chunking
- **Embedding model**: OpenAI `text-embedding-3-small` (1536 dims).
- **Chunking**: target **1000 tokens** with **10% overlap** (approximate with 4 chars/token — i.e., ~4000 chars size, 400 chars overlap).
- **Metadata** stored per chunk: `source`, `title`, `section`, `position`, `text` (snippet).

## 3) Retriever + Reranker
- **Retriever**: Top-K similarity search from Pinecone; default `RETRIEVAL_TOP_K=20`.
- **Reranker**: Cohere **Rerank v3** (`rerank-english-v3.0`) to select the final `RERANK_TOP_K=8` passages before answering.

## 4) LLM & Answering
- **LLM**: OpenAI **gpt-4o-mini** (switchable).
- **Grounding**: System prompt forces answers only from provided context. Inline citations use the format `[n]` and map to source snippets shown under the answer.
- **No-answer**: If nothing relevant is retrieved, the API returns a graceful fallback message.

## 5) Frontend
- **UX**: Left panel for paste/upload & ingest; right panel to ask; answers with citations and basic request timing.
- **Telemetry**: Shows request duration; pricing notes and token estimation heuristic.

## 6) Hosting & Docs
- **Hosting**: Vercel (free tier). `vercel.json` provided; API routes run on Node 18.
- **Secrets**: Keep API keys server-side in Vercel project settings or `.env.local` (use `.env.example`).
- **README**: Includes architecture, parameters, providers, and quick start.
- **Remarks**: See end of file for trade-offs.

---

## Quick Start (Local)

```bash
# 1) Clone and install
pnpm i   # or: npm i / yarn

# 2) Create .env.local from template
cp .env.example .env.local
# Fill in OPENAI_API_KEY, COHERE_API_KEY, PINECONE_API_KEY, PINECONE_INDEX

# 3) (Optional) Create Pinecone index if it doesn't exist
# The app auto-creates serverless index if missing.
# Ensure region in env matches your account.

# 4) Run
pnpm dev  # or: npm run dev / yarn dev
# Open http://localhost:3000
```

## Deploy to Vercel

1. Push this repo to GitHub.
2. Import the repo in Vercel.
3. Add environment variables from `.env.example` in Vercel Project → Settings → Environment Variables.
4. Deploy. First load should come up without console errors.

---

## API Overview

### `POST /api/ingest`
Body:
```json
{ "text": "<raw text>", "title": "Optional", "section": "Optional", "source": "Acme Handbook" }
```
- Chunks, embeds (OpenAI), and **upserts** into Pinecone.

### `POST /api/ask`
Body:
```json
{ "query": "How do I request PTO?" }
```
- Retrieves Top-K from Pinecone, reranks with Cohere, then answers with OpenAI and **inline citations**.

---

## Architecture Diagram

```
[Frontend] --paste/upload--> [/api/ingest] --chunk--> [OpenAI Embeddings] --upsert--> [Pinecone Index]
     |                                                                                         ^
     |                                                                                         |
     +-- query ----------------> [/api/ask] --embed--> [Pinecone TopK] --rerank(Cohere) --+    |
                                                     --context--> [OpenAI LLM] --answer-->+----+
```

---

## Minimal Eval

Add your domain text, then test with these 5 Q/A pairs (replace with your content):

1. **Q**: What are the core working hours?  
   **A**: Should cite the policy section and hours.  
2. **Q**: How do I file an expense report?  
   **A**: Should cite the finance/expenses section.  
3. **Q**: Who owns the on-call rotation?  
   **A**: Should cite the engineering handbook.  
4. **Q**: How is PTO accrued?  
   **A**: Should cite the HR/benefits section.  
5. **Q**: What’s the incident severity scale?  
   **A**: Should cite the SRE/incident response doc.

**Note**: After answering all 5, report a subjective success rate (e.g., 4/5 correct, 1 borderline). Precision is the fraction of returned citations that were actually relevant; recall is how many of the gold answers were found in the retrieved set.

---

## Index Config / Schema

- **Index**: `mini-rag`
- **Dimension**: 1536
- **Vector**: `values: number[1536]`
- **Metadata**: 
  - `source: string`
  - `title?: string`
  - `section?: string`
  - `position: number`
  - `text: string`

Upserts overwrite identical IDs (`{source}::{position}`).

---

## Settings Summary

- **Chunking**: 1000 tokens, 10% overlap (approx. 4 chars/token heuristic).
- **Retriever**: TopK=20
- **Reranker**: Cohere `rerank-english-v3.0`, TopN=8
- **LLM**: OpenAI `gpt-4o-mini`, temperature 0.2

---

## Remarks (Limits & Trade-offs)

- **Token counting** uses a heuristic to keep deps light. For stricter limits, switch to a tokenizer lib.
- **Serverless cold starts** can add latency on free tiers.
- **MMR**: We rely on semantic similarity → reranker rather than explicit MMR; acceptable for small corpora.
- **Costs/limits**: Watch provider quotas; swap to free tiers as needed (e.g., Groq for LLM inference).

---

## What I'd Do Next

- Add hybrid retrieval (sparse + dense), filters, and MMR.
- Add file uploads (PDF/Docx) with server-side parsing.
- Add per-document namespaces and delete/reindex tools.
- Add streaming answers and token/cost meters using provider usage APIs.
- Add lightweight eval harness and leaderboard.
