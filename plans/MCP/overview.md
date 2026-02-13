# Generix Docs MCP Server - Architecture Overview

## Getting Started

Build this project from scratch in a new directory called `generix-docs-mcp/`. Follow this order:

1. **Set up the project**: Create `pyproject.toml` with the dependencies listed below, then `uv sync`
2. **Build `storage.py` first**: ChromaDB collection management, embed + store, query. This is the foundation everything else imports.
3. **Build `server.py`**: Start with just `search_docs` and `list_documents` tools using FastMCP. Test by adding a markdown file manually and searching for it.
4. **Build `ingestion.py`**: Start with markdown/text support only. Add PDF, HTML, DOCX handlers one at a time (install optional deps as needed).
5. **Build `web.py`**: FastAPI app with list, upload, and delete routes. Remember: the web UI does NOT write to ChromaDB directly (see the ChromaDB constraint in Risks).
6. **Add git integration**: GitPython for add/commit/push in the upload flow, then the post-merge hook for auto-reindex.
7. **Add `scripts/reindex.py`**: CLI script that re-ingests all docs in `docs/` into ChromaDB. Used by git hooks and the web UI.

Read the full architecture below before starting. Pay special attention to the **ChromaDB concurrent access** constraint in the Risks section -- it drives how the two processes (MCP server + web UI) divide responsibilities.

## Quick Summary

A standalone MCP server that provides RAG-based semantic search over Generix WMS proprietary query language documentation. Programmers upload, browse, and remove documentation through a localhost web UI. Docs are stored in a shared git repo so the entire team stays in sync. Each developer runs the MCP server locally (stdio transport), and their local ChromaDB index auto-rebuilds after pulling changes.

## Scope

- **In scope**:
  - MCP server with RAG search tools (search_docs, fetch_document, list_documents, add_document, remove_document, get_stats)
  - Localhost web UI for document management (list, upload via drag-and-drop, delete)
  - Multi-format document ingestion (MD, PDF, HTML, DOCX, TXT)
  - Shared git repository for documentation files
  - Git hooks for auto-reindexing after pull
  - Cross-tool support (Claude Code, Cursor, Copilot, VS Code, JetBrains)

- **Out of scope**:
  - Centralized/hosted server (each dev runs locally)
  - User authentication (local-only, no auth needed)
  - Version-aware documentation (no per-version retrieval)
  - Web scraping/crawling of external documentation sites
  - Real-time sync between developers (sync happens via git pull)

- **Assumptions**:
  - Developers have `git` installed and configured with SSH/token auth for the shared repo
  - Python 3.10+ available on all developer machines
  - Documentation is primarily text-heavy (query language syntax, not image-heavy)

## Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  Developer Machine (local)                       │
│                                                                  │
│  ┌──────────────┐     ┌──────────────────────────────────────┐  │
│  │ Claude Code   │     │         generix-docs-mcp             │  │
│  │ Cursor        │────▶│                                      │  │
│  │ VS Code       │stdio│  server.py (MCP Server)              │  │
│  │ JetBrains     │     │    ├── search_docs(query, limit)     │  │
│  └──────────────┘     │    ├── fetch_document(source)         │  │
│                        │    ├── list_documents()               │  │
│                        │    ├── add_document(file_path)        │  │
│                        │    ├── remove_document(source)        │  │
│                        │    └── get_stats()                    │  │
│                        │                                      │  │
│  ┌──────────────┐     │  web.py (FastAPI Web UI)              │  │
│  │ Browser       │────▶│    ├── GET  / (list docs)            │  │
│  │ localhost:6280│http │    ├── POST /upload (drag-and-drop)   │  │
│  └──────────────┘     │    └── DELETE /docs/{name}            │  │
│                        │                                      │  │
│                        │  ┌─────────────┐  ┌───────────────┐  │  │
│                        │  │ storage.py   │  │ ingestion.py  │  │  │
│                        │  │ (shared)     │  │ (MD/PDF/HTML  │  │  │
│                        │  │              │  │  /DOCX/TXT)   │  │  │
│                        │  └──────┬──────┘  └───────────────┘  │  │
│                        │         │                             │  │
│                        │  ┌──────▼──────┐  ┌───────────────┐  │  │
│                        │  │  ChromaDB    │  │ docs/ (git)   │  │  │
│                        │  │  (local)     │  │  ├── file1.md │  │  │
│                        │  │  chroma_db/  │  │  ├── file2.pdf│  │  │
│                        │  └─────────────┘  └───────┬───────┘  │  │
│                        └──────────────────────────────────────┘  │
│                                                     │            │
└─────────────────────────────────────────────────────┼────────────┘
                                                      │ git push/pull
                                              ┌───────▼───────┐
                                              │  Remote Git    │
                                              │  Repository    │
                                              │  (GitHub/      │
                                              │   GitLab/etc)  │
                                              └───────────────┘
```

### Two Entry Points, Shared Core

The server has two independent entry points that share the same storage and ingestion logic:

1. **server.py** - MCP server (stdio transport). Launched by Claude Code/Cursor as a subprocess. Provides search and document management tools.
2. **web.py** - FastAPI web application on localhost:6280. Provides a visual UI for listing, uploading (drag-and-drop), and deleting documents.

Both import from a shared `storage.py` module that handles ChromaDB operations and file management.

**Why two processes**: MCP servers using stdio transport communicate via stdin/stdout. They cannot simultaneously serve HTTP. The web UI runs as a separate FastAPI process on a known port. **Important constraint**: ChromaDB is not process-safe, so only the MCP server process writes to ChromaDB. The web UI saves files to disk and triggers git operations, then signals for a reindex that runs in the MCP server's process context.

### Components Affected

| Component | Change Type | Risk |
|-----------|------------|------|
| `server.py` | New | Low - standard FastMCP tool definitions |
| `web.py` | New | Low - standard FastAPI routes + HTML template |
| `storage.py` | New | Medium - core document indexing and ChromaDB operations |
| `ingestion.py` | New | Medium - multi-format document extraction pipeline |
| `hooks/post-merge` | New | Low - shell script for auto-reindexing |
| `scripts/reindex.py` | New | Low - CLI script for manual/hook-triggered reindexing |
| `pyproject.toml` | New | Low - dependency declarations |
| `.mcp.json` (consumer projects) | Modified | Low - add MCP server configuration |

### Data Flow

#### Search Flow (MCP)

```
User asks question in Claude Code
  → Claude invokes search_docs("how to filter by warehouse")
    → storage.py: embed query + query ChromaDB
      → ChromaDB returns top-K similar chunks with metadata
    → Return ranked results [{content, source, score}, ...]
  → Claude uses results to answer
```

#### Upload Flow (Web UI)

```
Developer drags PDF into web UI
  → web.py: receives file via POST /upload
    → 1. Save file to docs/ directory (must succeed)
    → 2. git: add file → commit → push to remote (best-effort, retry later if fails)
    → 3. Trigger reindex (MCP server process re-ingests via scripts/reindex.py)
  → Web UI refreshes, shows new document in list with sync status (synced/pending/failed)
```

#### Sync Flow (Other Developers)

```
Developer B runs: git pull
  → post-merge hook fires
    → Detects changed files in docs/
    → Calls scripts/reindex.py
      → For added/modified files: extract → chunk → embed → upsert ChromaDB
      → For deleted files: delete from ChromaDB by source metadata
  → Developer B's local index is up to date
```

### Document Ingestion Pipeline

All formats are converted to markdown first, then chunked uniformly:

```
Input File → Format Detection (by extension)
  ├── .md   → read raw text
  ├── .pdf  → pymupdf4llm.to_markdown()
  ├── .html → BeautifulSoup4 → extract text
  ├── .docx → python-docx → extract text
  └── .txt  → read raw text
         │
         ▼
  MarkdownHeaderTextSplitter (split by #, ##, ### headers)
         │
         ▼
  RecursiveCharacterTextSplitter (size-limit oversized sections)
         │
         ▼
  Add metadata (source_file, section_path, format, has_code)
         │
         ▼
  ChromaDB: embed + store chunks
```

**Why markdown-first**: Heading-based chunking preserves the natural structure of documentation. A query language section like "SELECT Statement > WHERE Clause" stays together as one chunk, including both the syntax description and code examples. This is more effective than character-count chunking which would split examples mid-statement.

## Decisions Log

| # | Decision | Alternatives Considered | Rationale | Substantiation |
|---|----------|------------------------|-----------|----------------|
| 1 | FastMCP (v2+, Python) for MCP framework | Raw MCP Python SDK, TypeScript SDK | Decorator-based API is most productive. Project uses Python. Powers ~70% of MCP servers. Target v2 stable; v3 beta is compatible (same decorator API). | [FastMCP docs](https://gofastmcp.com), [FastMCP GitHub](https://github.com/jlowin/fastmcp) |
| 2 | ChromaDB for vector store | Qdrant, FAISS, Pinecone | Zero infrastructure (embedded), pip install, persistent storage. Good for <100K chunks. | [ChromaDB docs](https://docs.trychroma.com), research: mcp_for_specific_doc.md |
| 3 | sentence-transformers (all-MiniLM-L6-v2) for embeddings | OpenAI text-embedding-3-small, Ollama | Free, local, no API key needed. ChromaDB has built-in integration. Proprietary docs never leave the machine. | [ChromaDB embedding functions](https://docs.trychroma.com/docs/embeddings/embedding-functions) |
| 4 | Stdio transport (local per dev) | HTTP shared server, hybrid | User decision. Simplest deployment - no server infrastructure needed. Each dev has their own index. | User interview |
| 5 | Shared git repo for documentation | Network drive, cloud storage, embedded in project | User decision. Version-controlled, standard dev workflow, merge-friendly. | User interview |
| 6 | Two entry points (server.py + web.py) | Single process with HTTP transport, background thread | Stdio MCP cannot serve HTTP simultaneously. Two processes with shared storage, but **only MCP server writes to ChromaDB** (see Risks). **Alternative**: HTTP transport (FastMCP supports `streamable-http`) would allow a single process serving MCP at `/mcp` and web UI at `/`, eliminating the ChromaDB concurrency issue entirely. Trade-off: each developer must configure their MCP client for HTTP instead of stdio (one-time setup). | [FastMCP transports](https://gofastmcp.com/deployment/running-server), UI research, [ChromaDB constraints](https://cookbook.chromadb.dev/core/system_constraints/) |
| 7 | FastAPI + Jinja2 for web UI | NiceGUI, Gradio, Flask | Lightweight additions (~3MB). FastAPI + Jinja2 + python-multipart are explicit dependencies, not included in FastMCP. Full control over UI. | UI research, [FastMCP pyproject.toml](https://github.com/jlowin/fastmcp/blob/main/pyproject.toml) |
| 8 | pymupdf4llm for PDF extraction | Docling, Unstructured, LlamaParse, MarkItDown | Lightweight (~15MB), fast (C-based), outputs markdown directly. No ML models needed for text-heavy docs. | [Ingestion benchmarks](https://procycons.com/en/blogs/pdf-data-extraction-benchmark/) |
| 9 | MarkdownHeaderTextSplitter for chunking | RecursiveCharacterTextSplitter, semantic chunking | Header-based splitting preserves logical sections (syntax + examples together). Zero compute cost vs semantic chunking. | [LangChain docs](https://python.langchain.com/docs/how_to/markdown_header_metadata_splitter/) |
| 10 | GitPython for git operations | PyGit2, subprocess | Most popular Python git library. Clean high-level API. Good for infrequent operations (doc uploads). | [GitPython docs](https://gitpython.readthedocs.io/) |
| 11 | Git hooks (post-merge) for auto-reindex | Manual reindex, file watcher, cron | Triggered exactly when needed (after pull), no polling overhead. | [Git hooks docs](https://git-scm.com/docs/githooks) |
| 12 | `core.hooksPath` for hook distribution | pre-commit framework, install script | Simplest one-command setup: `git config core.hooksPath ./hooks`. Hooks are version-controlled. | [Git config docs](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath) |
| 13 | Git LFS for binary files | Regular git, external object storage | Keeps repo fast for clones. PDFs/DOCX can be large. | [NEEDS TESTING] Depends on actual doc set size. May not be needed if docs are primarily text. |
| 14 | Standalone repo | Part of NuevoDia, develop-here-extract-later | User decision. Clean separation, installable by any team, can be shared or open-sourced. | User interview |

## Risks and Open Questions

- **Git LFS necessity** [severity: low] - If docs are primarily text (Markdown, TXT), LFS is unnecessary overhead. Mitigation: Start without LFS, add later if binary files grow the repo beyond 100MB.
- **ChromaDB concurrent access** [severity: high, DESIGN CONSTRAINT] - ChromaDB is explicitly **not process-safe** ([ChromaDB System Constraints](https://cookbook.chromadb.dev/core/system_constraints/), [GitHub #666](https://github.com/chroma-core/chroma/issues/666)). Two processes using PersistentClient on the same path will create data races. This is a confirmed limitation, not a theoretical risk. **Chosen mitigation**: web.py handles file operations (save to disk, git add/commit/push) only. ChromaDB writes are performed exclusively by the MCP server process via reindex. The web UI triggers a reindex script that runs in the MCP server's context (or a single-writer CLI). This avoids dual-process writes entirely. **Alternative mitigations considered**: (A) Run ChromaDB as an HTTP server (third process, adds complexity), (B) Switch to HTTP transport so both MCP and web UI run in a single process (see Decision #6 note below).
- [NEEDS TESTING] **sentence-transformers cold start** - First query after server start loads the embedding model (~1-2 seconds). Proposed test: measure startup time, consider lazy loading or pre-warming. Fallback: ChromaDB can precompute embeddings, so the model is only needed for query-time embedding.
- **Git push failures** [severity: medium, DESIGN REQUIREMENT] - Network issues or auth failures during push could leave local-only commits. The web UI must track git sync status per document (synced/pending/failed) and retry pending pushes on subsequent operations. Uploads are never blocked by git failures - file save + local index always succeed first.
- **Merge conflicts on binary files** [severity: low] - Unlikely with append-only pattern but possible. Mitigation: Pull-rebase-push retry pattern; for persistent conflicts, log and alert user via web UI.
- **FastMCP version** [severity: low] - Targeting v2 stable (production-ready). v3.0 beta 2 uses the same decorator API and is a drop-in upgrade when stable. The `>=2.0` constraint in pyproject.toml allows either version.

## Design Notes

- **Uploads are idempotent (upsert)**: If a document with the same filename already exists, uploading it again replaces the old version. The web UI and `add_document` tool both follow upsert semantics - delete existing chunks by source, then re-ingest.
- **First-run cold start**: The first search after install triggers a ~90MB model download (all-MiniLM-L6-v2). The `scripts/setup.py` setup script should pre-download the model so developers are not surprised by a 30-60 second delay on first query.
- **Port configuration**: The web UI defaults to port 6280 but should be configurable via the `GENERIX_WEB_PORT` environment variable for developers with port conflicts.
- **Health check**: The `get_stats` tool should compare files on disk in `docs/` vs documents indexed in ChromaDB and report any drift (files not indexed, orphaned index entries).
- **Search quality for query language terms**: For exact syntax matches (e.g., "GROUPBY"), ChromaDB metadata filters (`where` clause) should supplement semantic search. Consider adding keyword metadata from document titles/headers to support exact-match queries alongside semantic similarity.

## Phase 2 Requirements

### Recommended Specialist Agents for Phase 2

| Specialist | Why Needed | Key References |
|-----------|-----------|----------------|
| FastMCP Specialist | Verify tool definitions, error handling, annotation patterns | [FastMCP docs](https://gofastmcp.com/servers/tools), fastmcp-research.md |
| Document Ingestion Specialist | Implement multi-format pipeline, test with real Generix docs | ingestion-research.md, pymupdf4llm docs |
| Web UI Specialist | Build the FastAPI + Jinja2 web interface with drag-and-drop | ui-research.md, FastAPI templates docs |
| Git Integration Specialist | Implement GitPython workflows, hooks, LFS setup | git-research.md, GitPython docs |

### Key Files to Read

| File | Why | Reference |
|------|-----|-----------|
| FastMCP tools documentation | Tool decorator API, annotations, error handling | [gofastmcp.com/servers/tools](https://gofastmcp.com/servers/tools) |
| FastMCP server configuration | Entry point, transport, deployment | [gofastmcp.com/deployment/server-configuration](https://gofastmcp.com/deployment/server-configuration) |
| ChromaDB Python API | Collection operations, embedding functions, metadata filtering | [docs.trychroma.com](https://docs.trychroma.com) |
| pymupdf4llm API | PDF to markdown conversion | [pymupdf.readthedocs.io](https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/) |
| LangChain MarkdownHeaderTextSplitter | Header-based chunking API | [LangChain docs](https://python.langchain.com/docs/how_to/markdown_header_metadata_splitter/) |
| GitPython tutorial | Add, commit, push patterns | [gitpython.readthedocs.io/tutorial](https://gitpython.readthedocs.io/en/stable/tutorial.html) |
| FastAPI file upload | UploadFile handling, templates | [FastAPI docs](https://fastapi.tiangolo.com/tutorial/request-files/) |

### Substantiation Summary
- **Verified**: 12 decisions backed by official documentation or working open-source code
- **Pattern-based**: 2 decisions following established MCP server patterns (tool design, project structure)
- **[NEEDS TESTING]**: 1 decision requiring validation (sentence-transformers cold start time)

### Project Structure

```
generix-docs-mcp/
├── pyproject.toml              # Dependencies + entry points
├── README.md                   # Setup instructions
├── .mcp.json.example           # Template for consumer projects
├── src/
│   └── generix_docs_mcp/
│       ├── __init__.py
│       ├── server.py           # MCP server (stdio entry point)
│       ├── web.py              # FastAPI web UI (HTTP entry point)
│       ├── storage.py          # ChromaDB operations (shared)
│       ├── ingestion.py        # Multi-format document extraction
│       └── templates/
│           └── index.html      # Web UI template (list + upload + delete)
├── hooks/
│   ├── post-merge              # Auto-reindex on git pull
│   └── post-checkout           # Auto-reindex on branch switch
├── scripts/
│   ├── reindex.py              # Manual/hook reindex script
│   └── setup.py                # One-time setup (hooks, initial index)
├── docs/                       # Documentation files (git-tracked)
│   └── .gitkeep
├── chroma_db/                  # Vector DB storage (git-ignored)
├── .gitignore                  # Ignore chroma_db/, __pycache__/
└── tests/
    ├── test_storage.py         # Storage layer tests
    ├── test_ingestion.py       # Ingestion pipeline tests
    └── test_server.py          # MCP tool tests
```

### Dependencies

```toml
[project]
name = "generix-docs-mcp"
version = "0.1.0"
description = "MCP server for Generix WMS query language documentation"
requires-python = ">=3.10"
dependencies = [
    "fastmcp>=2.0",              # MCP server framework
    "chromadb>=0.5",             # Vector database
    "sentence-transformers>=3.0", # Local embeddings
    "langchain-text-splitters>=0.3", # Text chunking
    "gitpython>=3.1.40",        # Git operations
    "fastapi>=0.115",            # Web UI framework
    "uvicorn>=0.30",             # ASGI server for web UI
    "jinja2>=3.1",               # HTML templates
    "python-multipart>=0.0.9",   # File upload support
]
# Format-specific extractors are optional (install with pip install generix-docs-mcp[all-formats])
# Core install supports markdown and plain text only

[project.optional-dependencies]
pdf = ["pymupdf4llm>=0.0.17"]
html = ["beautifulsoup4>=4.12", "lxml>=5.0"]
docx = ["python-docx>=1.1"]
all-formats = ["pymupdf4llm>=0.0.17", "beautifulsoup4>=4.12", "lxml>=5.0", "python-docx>=1.1"]

[project.scripts]
generix-docs = "generix_docs_mcp.server:main"
generix-docs-web = "generix_docs_mcp.web:main"
```

### MCP Tool Design

| Tool | Purpose | Annotations | Parameters |
|------|---------|-------------|-----------|
| `search_docs` | Semantic search over indexed docs | readOnly, idempotent | `query: str`, `n_results: int = 5` |
| `list_documents` | List all indexed docs with chunk counts | readOnly | (none) |
| `add_document` | Ingest a file into the knowledge base | not readOnly | `file_path: str` |
| `remove_document` | Remove a doc from the knowledge base | destructive | `source: str` |
| `fetch_document` | Retrieve the full text of a specific document | readOnly | `source: str` |
| `get_stats` | Knowledge base statistics | readOnly | (none) |

### Consumer Project Configuration

Projects that want to use the Generix docs MCP server add to their `.mcp.json`:

```json
{
  "mcpServers": {
    "generix-docs": {
      "type": "stdio",
      "command": "uv",
      "args": ["run", "--directory", "/path/to/generix-docs-mcp", "generix-docs"],
      "env": {
        "GENERIX_DOCS_PATH": "/path/to/generix-docs-mcp/docs",
        "GENERIX_CHROMA_PATH": "/path/to/generix-docs-mcp/chroma_db"
      }
    }
  }
}
```
