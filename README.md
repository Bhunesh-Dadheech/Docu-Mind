
# DocuMind — Multi-Tenant RAG Knowledge Assistant

A production-style Retrieval-Augmented Generation (RAG) chatbot where each authenticated user
gets an isolated knowledge base built from their own uploaded documents. Built to mirror the
architecture patterns used in production RAG systems: multi-provider auth, per-tenant vector
retrieval, streaming responses, and persistent chat history.

## Why this project

Most "chat with your PDF" demos use a single global vector store and no real auth. DocuMind
solves the harder, more realistic problem: **many users, isolated data, streamed answers,
and durable history** — the same shape of problem found in real multi-tenant RAG products.

## Architecture

```
┌─────────────┐      ┌──────────────┐      ┌───────────────────┐
│   Browser    │◄────►│  Flask API   │◄────►│  Firebase Firestore │
│ (SSE client) │      │ (JWT auth)   │      │  (chat history)     │
└─────────────┘      └──────┬───────┘      └───────────────────┘
                             │
                    ┌────────┴─────────┐
                    │  OAuth (Google/  │
                    │  GitHub) via     │
                    │  Authlib         │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │  Ingestion        │
                    │  (chunk+embed)    │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │  Per-user FAISS   │
                    │  index            │
                    │  data/vectorstores│
                    │  /<user_id>.faiss │
                    └───────────────────┘
```

## Key components

| Component | File | What it demonstrates |
|---|---|---|
| OAuth login (Google + GitHub) | `app/auth.py` | Multi-provider OAuth2 flow, issuing signed JWTs |
| JWT middleware | `app/auth.py` | Stateless per-request auth, token expiry/refresh |
| Document ingestion | `app/ingest.py` | Chunking, embedding, per-tenant FAISS index build |
| Multi-tenant retrieval | `app/rag.py` | One FAISS index per user — no cross-tenant data leakage |
| Streaming chat endpoint | `app/routes.py` | Server-Sent Events (SSE) token-by-token streaming |
| Persistent history | `app/history.py` | Firestore-backed per-user conversation threads |
| Source citations | `app/rag.py` | Retrieved chunks returned with similarity scores |

## Tech stack

- **Backend:** Python, Flask, Flask-SSE (SSE streaming)
- **Retrieval:** LangChain, FAISS, sentence-transformers embeddings
- **Auth:** Authlib (OAuth2), PyJWT
- **Storage:** Firebase Admin SDK (Firestore) for chat history, local disk for FAISS indices
- **LLM:** OpenAI/Anthropic API (pluggable via `app/llm.py`)
- **Frontend:** Minimal HTML + vanilla JS EventSource client (no framework needed to demo the API)

## Project layout

```
documind/
├── app/
│   ├── __init__.py       # Flask app factory
│   ├── auth.py           # OAuth login + JWT issue/verify
│   ├── ingest.py         # Upload -> chunk -> embed -> FAISS build
│   ├── rag.py            # Per-tenant retrieval + answer generation
│   ├── history.py        # Firestore chat history read/write
│   ├── llm.py            # LLM provider wrapper (swap OpenAI/Anthropic)
│   └── routes.py         # API endpoints
├── templates/
│   └── index.html        # Minimal chat UI with SSE client
├── static/
│   └── chat.js
├── data/
│   └── vectorstores/     # per-user FAISS indices (gitignored)
├── requirements.txt
├── .env.example
├── config.py
└── run.py
```

## Build phases (suggested order)

1. **Phase 1 — Skeleton & auth:** Flask app factory, `/login/google`, `/login/github`, JWT issue on
   callback, `/me` protected route to prove the token works.
2. **Phase 2 — Ingestion:** file upload endpoint, chunking (LangChain `RecursiveCharacterTextSplitter`),
   embedding, write a per-user FAISS index to disk.
3. **Phase 3 — Retrieval + generation:** `/chat` endpoint, similarity search scoped to the
   authenticated user's index, prompt assembly with retrieved chunks, non-streamed answer first.
4. **Phase 4 — Streaming:** convert `/chat` to SSE, stream tokens from the LLM call.
5. **Phase 5 — Persistence:** wire Firestore, save each turn (user msg, assistant msg, sources)
   per user, add a `/history` endpoint.
6. **Phase 6 — Polish:** minimal frontend, citations UI, README screenshots/GIF, deploy to
   Render/Railway free tier.

## Running locally

```bash
cp .env.example .env   # fill in OAuth client IDs/secrets, Firebase creds, LLM API key
pip install -r requirements.txt
python run.py
```

## What's stubbed vs. real

This scaffold ships with working OAuth + JWT logic, a real chunk-embed-FAISS ingestion path, and
a real SSE streaming endpoint. The LLM call in `app/llm.py` is a thin wrapper you point at your
own API key — nothing about the retrieval or auth logic is mocked.
