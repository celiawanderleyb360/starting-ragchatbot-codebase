# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run the application (from project root)
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# Run a single backend module directly
cd backend && uv run python <module>.py
```

The app serves at `http://localhost:8000` (web UI) and `http://localhost:8000/docs` (Swagger).

## Environment

Requires Python 3.13+ and uv. Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`.

**Always use `uv` to run commands and manage dependencies. Never use `pip` directly.**

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials. FastAPI backend, vanilla JS frontend, ChromaDB for vector storage, Claude for generation.

### Query Flow

User question → `frontend/script.js` POSTs to `/api/query` → `app.py` endpoint → `RAGSystem.query()` orchestrates:
1. `SessionManager` retrieves conversation history (formatted as plain text, appended to system prompt)
2. `AIGenerator` sends query + history + tool definitions to Claude API
3. Claude either answers directly OR calls `search_course_content` tool
4. If tool used: `ToolManager` → `CourseSearchTool` → `VectorStore.search()` → ChromaDB semantic query → results sent back to Claude in a second API call (without tools) for synthesis
5. Sources are tracked on `CourseSearchTool.last_sources`, collected by `ToolManager.get_last_sources()`, then reset

### Document Ingestion (startup)

`app.py` startup event → `RAGSystem.add_course_folder("../docs")` → for each `.txt` file:
1. `DocumentProcessor.process_course_document()` parses header metadata (title, link, instructor) then splits on `Lesson N:` markers
2. Each lesson's text is chunked via `chunk_text()` — sentence-aware splitting, 800 char chunks, 100 char overlap
3. `VectorStore` stores into two ChromaDB collections:
   - `course_catalog` — one doc per course (title as document text + metadata), used for fuzzy course name resolution
   - `course_content` — one doc per chunk with metadata (course_title, lesson_number, chunk_index)
4. Deduplication: courses already in ChromaDB (matched by title) are skipped on subsequent startups

### Key Design Decisions

- **Tool-based RAG**: Claude decides when/whether to search via Anthropic tool use, rather than always retrieving. The single tool `search_course_content` accepts optional `course_name` and `lesson_number` filters.
- **Course name resolution**: Partial course names are resolved via semantic search against `course_catalog` before filtering `course_content`.
- **Conversation history**: Stored in-memory by `SessionManager` as plain text, capped at `MAX_HISTORY * 2` messages, injected into the system prompt (not as message turns).
- **Two Claude API calls per tool use**: First call returns tool_use, tool executes, second call (without tools) synthesizes the final answer.
- **Persistent vector store**: ChromaDB persists to `backend/chroma_db/`, so embeddings survive restarts.
- **Frontend**: No framework — vanilla JS with `marked.js` for markdown rendering. All state is a single `currentSessionId` variable.

### Configuration

All tunable parameters are in `backend/config.py` as a `Config` dataclass: model names, chunk size/overlap, max results, max history, ChromaDB path. The `config` singleton is imported throughout the backend.

### Document Format

Course files in `docs/` follow this structure:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <title>
Lesson Link: <url>
<content>

Lesson 1: <title>
<content>
```
