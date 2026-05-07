# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install dependencies:**
```bash
uv sync
```

**Run the server:**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app is served at `http://localhost:8000` (web UI) and `http://localhost:8000/docs` (FastAPI Swagger UI).

**Environment setup:**
```bash
cp .env.example .env
# Then add your ANTHROPIC_API_KEY to .env
```

## Architecture

This is a full-stack RAG (Retrieval-Augmented Generation) chatbot with a FastAPI backend and a plain HTML/JS/CSS frontend. The frontend is served as static files from the backend itself.

**Request flow for a user query:**
1. `frontend/script.js` POSTs to `/api/query`
2. `backend/app.py` receives the request and calls `RAGSystem.query()`
3. `RAGSystem` builds a prompt and calls `AIGenerator.generate_response()` with Claude tools
4. Claude decides whether to call the `search_course_content` tool (via `CourseSearchTool` → `VectorStore.search()`)
5. Tool results are fed back to Claude for final answer generation
6. The response and sources are returned to the frontend

**Key backend modules:**
- `rag_system.py` — Top-level orchestrator; wires together all components
- `ai_generator.py` — Anthropic SDK wrapper; handles tool execution loop (one round of tool use max)
- `vector_store.py` — ChromaDB wrapper with two collections: `course_catalog` (course metadata) and `course_content` (chunked lesson text)
- `document_processor.py` — Parses `.txt`/`.pdf`/`.docx` course files into `Course`/`Lesson`/`CourseChunk` models and chunks text
- `search_tools.py` — Defines the `search_course_content` tool and `ToolManager` registry used by Claude
- `session_manager.py` — In-memory conversation history (last `MAX_HISTORY=2` exchanges)
- `config.py` — All tunable parameters (model, chunk size, embedding model, ChromaDB path)

**Course document format** (for files in `docs/`):
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 1: <lesson title>
...
```

On startup, `app.py` auto-loads all `.txt`/`.pdf`/`.docx` files from `../docs` into ChromaDB (skipping already-indexed courses by title).

**ChromaDB** is stored locally at `backend/chroma_db/` (created on first run). To force a full re-index, call `VectorStore.clear_all_data()` or delete the `chroma_db/` directory.

**Embedding model:** `all-MiniLM-L6-v2` via `sentence-transformers` (downloaded automatically on first run).
