# Generix Docs MCP Server - Detailed Specification

## Overview Reference
Based on: `apps/plans/MCP/overview.md`

## Files Changed

### New Files
| File | Purpose | Pattern |
|------|---------|---------|
| `pyproject.toml` | Dependencies, entry points, build config | Standard Python packaging |
| `src/generix_docs_mcp/__init__.py` | Package marker | Empty |
| `src/generix_docs_mcp/storage.py` | ChromaDB operations (shared core) | Module-level functions with lazy init |
| `src/generix_docs_mcp/ingestion.py` | Multi-format document extraction + chunking | Pipeline pattern |
| `src/generix_docs_mcp/server.py` | MCP server with 6 tools (stdio entry point) | FastMCP decorator pattern |
| `src/generix_docs_mcp/web.py` | FastAPI web UI (HTTP entry point) | FastAPI + Jinja2 |
| `src/generix_docs_mcp/templates/index.html` | Web UI template | Jinja2 + vanilla JS |
| `hooks/post-merge` | Auto-reindex after git pull | Bash git hook |
| `hooks/post-checkout` | Auto-reindex after branch switch | Bash git hook |
| `scripts/reindex.py` | CLI reindex tool | argparse CLI |
| `scripts/setup.py` | One-time setup | CLI setup script |
| `docs/.gitkeep` | Empty docs directory | Git convention |
| `.gitignore` | Ignore patterns | Standard Python |
| `.mcp.json.example` | Consumer project template | MCP config |
| `README.md` | Setup instructions | Documentation |
| `tests/test_storage.py` | Storage layer tests | pytest |
| `tests/test_ingestion.py` | Ingestion pipeline tests | pytest |
| `tests/test_server.py` | MCP tool tests | pytest |
| `tests/test_web.py` | Web UI route tests | pytest + httpx (FastAPI TestClient) |

### Modified Files
None. This is a standalone project.

---

## Important Note on Component Specs

This spec.md is the **authoritative API definition**. The component scratch files in `~/.claude/scratch/spec-MCP/` contain supplementary research and code examples but use different function names and signatures in places. When in doubt, follow spec.md. The scratch files are useful for understanding API citations and seeing complete code patterns, but adapt function names to match what's defined here.

## Running the System

Two processes need to run for the full experience:

```bash
# Terminal 1: MCP server (started automatically by your AI tool, or manually)
uv run generix-docs

# Terminal 2: Web UI
uv run generix-docs-web
# Open http://localhost:6280
```

The MCP server is typically started automatically by Claude Code/Cursor via `.mcp.json`. The web UI must be started manually.

**ChromaDB concurrency note**: Both the MCP server and the reindex script (called by web.py and git hooks) create separate ChromaDB PersistentClient instances. ChromaDB is not process-safe, so brief concurrent access during reindex may cause issues. In practice, MCP tool calls are infrequent and short-lived, so conflicts are unlikely. If issues arise, stop the MCP server before bulk uploads, or switch to HTTP transport (see overview.md Decision #6).

## Component Specifications

### 1. `pyproject.toml`

```toml
[project]
name = "generix-docs-mcp"
version = "0.1.0"
description = "MCP server for Generix WMS query language documentation"
requires-python = ">=3.10"
dependencies = [
    "fastmcp>=2.0",
    "chromadb>=0.5",
    "sentence-transformers>=3.0",
    "langchain-text-splitters>=0.3",
    "gitpython>=3.1.40",
    "fastapi>=0.115",
    "uvicorn>=0.30",
    "jinja2>=3.1",
    "python-multipart>=0.0.9",
]

[project.optional-dependencies]
pdf = ["pymupdf4llm>=0.0.17"]
html = ["beautifulsoup4>=4.12", "lxml>=5.0"]
docx = ["python-docx>=1.1"]
all-formats = [
    "pymupdf4llm>=0.0.17",
    "beautifulsoup4>=4.12",
    "lxml>=5.0",
    "python-docx>=1.1",
]
dev = ["pytest>=8.0"]

[project.scripts]
generix-docs = "generix_docs_mcp.server:main"
generix-docs-web = "generix_docs_mcp.web:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

**Install commands:**
- Core (markdown + text only): `uv sync`
- All formats: `uv sync --extra all-formats`
- Development: `uv sync --extra all-formats --extra dev`

#### Dependencies
- Requires: Nothing (standalone project)
- Used by: All other files

---

### 2. `src/generix_docs_mcp/storage.py`

**Purpose**: ChromaDB collection lifecycle, chunk CRUD, querying, statistics.

**Critical constraint**: ChromaDB is not process-safe ([source](https://cookbook.chromadb.dev/core/system_constraints/)). Only one process may hold a `PersistentClient` at a time.

#### Constants

```python
import os

CHROMA_PATH: str = os.environ.get("GENERIX_CHROMA_PATH", "./chroma_db")
COLLECTION_NAME: str = "generix_docs"
EMBEDDING_MODEL: str = "all-MiniLM-L6-v2"
```

#### Chunk Metadata Schema

Every chunk stored in ChromaDB carries this metadata:

```python
{
    "source": str,          # Filename (e.g. "query_language.md")
    "chunk_index": int,     # Zero-based position within the document
    "format": str,          # "md", "pdf", "html", "docx", or "txt"
    "section_path": str,    # Slash-separated headers (e.g. "SELECT/WHERE Clause")
    "has_code": bool,       # True if chunk contains a fenced code block
}
```

All values are ChromaDB-compatible scalar types ([source](https://docs.trychroma.com)).

#### Public Functions

```python
def get_collection() -> chromadb.Collection:
    """Return the ChromaDB collection, initializing on first call.

    Creates PersistentClient at CHROMA_PATH. Creates collection with
    SentenceTransformerEmbeddingFunction(model_name=EMBEDDING_MODEL).
    Caches in module-level variables for subsequent calls.

    Side effects:
        - First call creates chroma_db/ directory.
        - First call on fresh install triggers ~90MB model download.

    Source: chromadb.PersistentClient - https://docs.trychroma.com/reference/python/client
    Source: SentenceTransformerEmbeddingFunction - https://github.com/chroma-core/chroma/blob/main/chromadb/utils/embedding_functions/sentence_transformer_embedding_function.py
    """


def add_document_chunks(
    source: str,
    chunks: list[str],
    metadatas: list[dict[str, str | int | float | bool]],
) -> int:
    """Delete existing chunks for source, then add new chunks (upsert).

    Args:
        source: Source filename.
        chunks: List of text strings, one per chunk.
        metadatas: List of metadata dicts, one per chunk. Each must contain "source" key.

    Returns: Number of chunks added.
    Raises: ValueError if chunks/metadatas length mismatch or empty.

    Uses collection.delete(where={"source": source}) then collection.add().
    Source: https://docs.trychroma.com/reference/python/collection
    """


def query(
    query_text: str,
    n_results: int = 5,
    where: dict | None = None,
) -> list[dict[str, str | float | dict]]:
    """Semantic search. Returns list of {content, source, score, metadata}.

    Score = 1.0 - distance (higher is better).
    Uses collection.query(query_texts=[query_text], n_results=n_results, include=["documents","metadatas","distances"]).
    Returns empty list if collection is empty.
    Source: https://docs.trychroma.com/reference/python/collection
    """


def delete_by_source(source: str) -> int:
    """Delete all chunks for a source. Returns count deleted.
    Raises ValueError if no chunks exist for source.
    Source: collection.delete(where={"source": source})
    """


def list_documents() -> list[dict[str, str | int]]:
    """List unique sources with chunk counts. Returns [{source, chunks}].
    Sorted alphabetically. Uses collection.get(include=["metadatas"]) and aggregates.
    """


def get_stats(docs_path: str | None = None) -> dict:
    """Knowledge base statistics with optional drift detection.

    Returns: {total_chunks, total_documents, sources, collection_name,
              embedding_model, chroma_path, not_indexed, orphaned}

    If docs_path provided, compares files on disk vs indexed sources.
    """


def fetch_document_text(source: str) -> str:
    """Reconstruct full text from chunks, ordered by chunk_index.
    Raises ValueError if no chunks exist for source.
    """
```

#### Helper

```python
def _make_chunk_id(source: str, chunk_index: int) -> str:
    """MD5 hash of "{source}:{chunk_index}" for deterministic IDs."""
```

#### Dependencies
- Requires: `chromadb`, `sentence-transformers`
- Used by: `server.py`, `scripts/reindex.py`

---

### 3. `src/generix_docs_mcp/ingestion.py`

**Purpose**: Multi-format text extraction, markdown conversion, chunking pipeline.

#### Constants

```python
CHUNK_SIZE: int = 1000
CHUNK_OVERLAP: int = 150
SUPPORTED_EXTENSIONS: set[str] = {".md", ".txt", ".pdf", ".html", ".htm", ".docx"}

HEADERS_TO_SPLIT_ON: list[tuple[str, str]] = [
    ("#", "h1"),
    ("##", "h2"),
    ("###", "h3"),
]
```

#### Optional Dependency Flags

```python
try:
    import pymupdf4llm
    HAS_PDF = True
except ImportError:
    HAS_PDF = False

try:
    from bs4 import BeautifulSoup
    HAS_HTML = True
except ImportError:
    HAS_HTML = False

try:
    import docx
    HAS_DOCX = True
except ImportError:
    HAS_DOCX = False
```

#### Public Functions

```python
def ingest_file(
    file_path: str | os.PathLike,
) -> tuple[list[str], list[dict[str, str | int | float | bool]]]:
    """Extract text, chunk, and generate metadata for a file.

    Pipeline:
    1. Detect format by extension
    2. Extract text → convert to markdown
    3. MarkdownHeaderTextSplitter (split by #, ##, ###)
    4. RecursiveCharacterTextSplitter (size-limit oversized sections)
    5. Generate per-chunk metadata

    Args:
        file_path: Path to the document file.

    Returns:
        (chunks, metadatas) tuple ready for storage.add_document_chunks().

    Raises:
        FileNotFoundError: File does not exist.
        ValueError: Unsupported extension.
        ImportError: Optional dependency not installed.
    """


def get_supported_extensions() -> dict[str, bool]:
    """Map extension to availability. {".md": True, ".pdf": False, ...}"""
```

#### Private Functions

```python
def _detect_format(file_path: str) -> str:
    """Extension to format string. Maps .htm to "html"."""


def _extract_markdown(file_path: str) -> str:
    """Read .md file as-is."""


def _extract_text(file_path: str) -> str:
    """Read .txt file as-is."""


def _extract_pdf(file_path: str) -> str:
    """pymupdf4llm.to_markdown(file_path). Returns markdown string.
    Source: https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/api.html
    """


def _extract_html(file_path: str) -> str:
    """BeautifulSoup with lxml parser. Converts <h1>-<h6> to markdown headings.
    Removes <script>, <style>, <nav>, <footer>.
    Source: https://beautiful-soup-4.readthedocs.io/en/latest/
    """


def _extract_docx(file_path: str) -> str:
    """python-docx paragraph iteration. Maps Heading styles to markdown #.
    Source: https://python-docx.readthedocs.io/en/latest/api/document.html
    """


def _chunk_markdown(
    text: str, source: str, format_str: str,
) -> tuple[list[str], list[dict]]:
    """Two-stage chunking pipeline.

    Stage 1: MarkdownHeaderTextSplitter(headers_to_split_on, strip_headers=False)
        Source: https://github.com/langchain-ai/langchain/blob/master/libs/text-splitters/langchain_text_splitters/markdown.py

    Stage 2: RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)
        Uses split_documents() to preserve header metadata through second stage.
        Source: https://reference.langchain.com/v0.3/python/text_splitters/

    Generates metadata per chunk: source, chunk_index, format, section_path, has_code.
    """


def _build_section_path(header_metadata: dict[str, str]) -> str:
    """Convert {"h1": "SELECT", "h2": "WHERE"} to "SELECT/WHERE"."""
```

#### Dependencies
- Requires: `langchain-text-splitters`
- Optional: `pymupdf4llm`, `beautifulsoup4`+`lxml`, `python-docx`
- Used by: `server.py`, `scripts/reindex.py`

---

### 4. `src/generix_docs_mcp/server.py`

**Purpose**: MCP server entry point. Thin adapter layer over storage.py and ingestion.py.

#### Complete File

```python
"""Generix WMS Documentation MCP Server."""

import os
from pathlib import Path
from typing import Annotated

from fastmcp import FastMCP
from fastmcp.exceptions import ToolError
from mcp.types import ToolAnnotations
from pydantic import Field

from generix_docs_mcp import storage, ingestion

DOCS_PATH = Path(os.environ.get("GENERIX_DOCS_PATH", "./docs"))

mcp = FastMCP(
    name="Generix Docs",
    instructions=(
        "Search and manage Generix WMS query language documentation. "
        "Use search_docs for natural language queries. "
        "Use fetch_document to read a full document. "
        "Use list_documents to see what is indexed. "
        "Use add_document to ingest new files. "
        "Use remove_document to delete indexed documents. "
        "Use get_stats to check knowledge base health."
    ),
)


@mcp.tool(
    annotations=ToolAnnotations(
        readOnlyHint=True, destructiveHint=False,
        idempotentHint=True, openWorldHint=False,
    ),
)
def search_docs(
    query: Annotated[str, Field(description="Natural language search query")],
    n_results: Annotated[int, Field(ge=1, le=20, description="Number of results")] = 5,
) -> list[dict]:
    """Search Generix WMS documentation using semantic similarity.

    Returns ranked document chunks with content, source file, and relevance score.
    """
    if not query.strip():
        raise ToolError("Query cannot be empty.")
    results = storage.query(query_text=query, n_results=n_results)
    if not results:
        return [{"message": "No results found.", "query": query}]
    return [
        {"content": r["content"], "source": r["source"], "score": round(r["score"], 4)}
        for r in results
    ]


@mcp.tool(
    annotations=ToolAnnotations(
        readOnlyHint=True, destructiveHint=False,
        idempotentHint=True, openWorldHint=False,
    ),
)
def fetch_document(
    source: Annotated[str, Field(description="Source filename (use list_documents to see available)")],
) -> str:
    """Retrieve the full text of a specific indexed document."""
    if not source.strip():
        raise ToolError("Source cannot be empty.")
    try:
        return storage.fetch_document_text(source=source)
    except ValueError:
        raise ToolError(f"No document found with source '{source}'.")


@mcp.tool(
    annotations=ToolAnnotations(
        readOnlyHint=True, destructiveHint=False,
        idempotentHint=True, openWorldHint=False,
    ),
)
def list_documents() -> list[dict]:
    """List all documents in the knowledge base with chunk counts."""
    docs = storage.list_documents()
    if not docs:
        return [{"message": "No documents indexed. Use add_document to ingest files."}]
    return docs


@mcp.tool(
    annotations=ToolAnnotations(
        readOnlyHint=False, destructiveHint=False,
        idempotentHint=False, openWorldHint=False,
    ),
)
def add_document(
    file_path: Annotated[str, Field(description="Absolute path to file (.md, .txt, .pdf, .html, .docx)")],
) -> str:
    """Add a document to the knowledge base. Replaces existing if same filename (upsert)."""
    path = Path(file_path)
    if not path.is_file():
        raise ToolError(f"File not found: {file_path}")
    if path.suffix.lower() not in ingestion.SUPPORTED_EXTENSIONS:
        raise ToolError(f"Unsupported format '{path.suffix}'.")
    try:
        source = path.name
        chunks, metadatas = ingestion.ingest_file(file_path=path)
        count = storage.add_document_chunks(source=source, chunks=chunks, metadatas=metadatas)
        fmt = metadatas[0]["format"] if metadatas else "unknown"
    except (ValueError, ImportError) as e:
        raise ToolError(str(e))
    return f"Added '{source}': {count} chunks indexed ({fmt} format)."


@mcp.tool(
    annotations=ToolAnnotations(
        readOnlyHint=False, destructiveHint=True,
        idempotentHint=False, openWorldHint=False,
    ),
)
def remove_document(
    source: Annotated[str, Field(description="Source filename to remove")],
) -> str:
    """Remove a document from the knowledge base. Does NOT delete file from disk."""
    if not source.strip():
        raise ToolError("Source cannot be empty.")
    try:
        count = storage.delete_by_source(source=source)
    except ValueError:
        raise ToolError(f"No document found with source '{source}'.")
    return f"Removed '{source}': {count} chunks deleted."


@mcp.tool(
    annotations=ToolAnnotations(
        readOnlyHint=True, destructiveHint=False,
        idempotentHint=True, openWorldHint=False,
    ),
)
def get_stats() -> dict:
    """Knowledge base statistics with health check (disk vs index drift)."""
    stats = storage.get_stats(docs_path=str(DOCS_PATH))
    health = "healthy"
    if stats.get("not_indexed") or stats.get("orphaned"):
        health = "drift_detected"
    stats["health"] = health
    stats["docs_path"] = str(DOCS_PATH)
    return stats


def main():
    """Entry point for generix-docs CLI command. Runs stdio MCP server."""
    mcp.run()


if __name__ == "__main__":
    main()
```

**Sources:**
- FastMCP tool decorator: [gofastmcp.com/servers/tools](https://gofastmcp.com/servers/tools)
- ToolAnnotations: `from mcp.types import ToolAnnotations` ([MCPcat guide](https://mcpcat.io/guides/adding-custom-tools-mcp-server-python/))
- ToolError: `from fastmcp.exceptions import ToolError` ([FastMCP source](https://github.com/jlowin/fastmcp/blob/main/src/fastmcp/exceptions.py))
- mcp.run(): [gofastmcp.com/deployment/running-server](https://gofastmcp.com/deployment/running-server)

#### Dependencies
- Requires: `storage.py`, `ingestion.py`, `fastmcp`, `mcp`, `pydantic`
- Used by: `.mcp.json` consumer projects

---

### 5. `src/generix_docs_mcp/web.py`

**Purpose**: FastAPI web application for document management on localhost.

**Critical**: Does NOT import `storage.py`. Triggers `scripts/reindex.py` via subprocess for ChromaDB writes.

#### Configuration

```python
BASE_DIR = Path(__file__).resolve().parent.parent.parent  # generix-docs-mcp/
DOCS_DIR = Path(os.environ.get("GENERIX_DOCS_PATH", str(BASE_DIR / "docs")))
REPO_DIR = Path(os.environ.get("GENERIX_REPO_PATH", str(BASE_DIR)))
REINDEX_SCRIPT = BASE_DIR / "scripts" / "reindex.py"
WEB_PORT = int(os.environ.get("GENERIX_WEB_PORT", "6280"))
ALLOWED_EXTENSIONS = {".md", ".pdf", ".html", ".htm", ".docx", ".txt"}

# In-memory git sync status (ephemeral, lost on restart)
_sync_status: dict[str, str] = {}
```

#### Routes

| Endpoint | Method | Parameters | Returns | Side Effects |
|----------|--------|------------|---------|-------------|
| `/` | GET | none | HTMLResponse | Reads DOCS_DIR |
| `/upload` | POST | `file: UploadFile` | `{status, filename, sync_status}` | Write file, git commit+push, reindex subprocess |
| `/docs/{name}` | DELETE | `name: str` | `{status, deleted}` | Delete file, git commit+push, reindex --delete |
| `/api/docs` | GET | none | `{docs: [...]}` | Reads DOCS_DIR |

#### Helper Functions

```python
def _get_documents() -> list[dict]:
    """List files in DOCS_DIR with metadata: name, size_human, modified_iso, sync_status."""

def _format_size(size_bytes: int) -> str:
    """Format bytes as human-readable (e.g., '1.2 MB')."""

def _git_add_commit_push(filename: str, action: str) -> None:
    """Git add/remove + commit + push. Updates _sync_status to 'synced' or 'failed'.
    Uses GitPython: Repo(REPO_DIR), Actor, index.add/remove, index.commit, remote.push.
    Source: https://gitpython.readthedocs.io/en/stable/tutorial.html
    """

def _push_with_retry(repo: Repo, max_retries: int = 3) -> None:
    """Push with pull-rebase retry on conflict.
    Source: remote.push().raise_if_error() - https://gitpython.readthedocs.io/
    """

def _trigger_reindex(filename: str | None = None, delete: bool = False) -> None:
    """Trigger scripts/reindex.py via subprocess.run().
    Commands: --file NAME (single), --delete NAME (remove), no args (full).
    """
```

#### Upload Flow

```
POST /upload with file
  1. Validate filename and extension
  2. Sanitize filename (Path.name, no path traversal)
  3. Write bytes to DOCS_DIR / filename (must succeed)
  4. Set _sync_status to "pending"
  5. _git_add_commit_push(filename, "Add"/"Update") (best-effort)
  6. _trigger_reindex(filename=filename) (subprocess)
  7. Return JSON with sync_status
```

#### Delete Flow

```
DELETE /docs/{name}
  1. Check file exists in DOCS_DIR
  2. _git_add_commit_push(name, "Remove") -- uses index.remove(working_tree=True)
  3. _trigger_reindex(filename=name, delete=True)
  4. Remove from _sync_status
  5. Return JSON confirmation
```

#### Entry Point

```python
def main() -> None:
    uvicorn.run(app, host="127.0.0.1", port=WEB_PORT, log_level="info")
```

**Sources:**
- FastAPI UploadFile: [fastapi.tiangolo.com/tutorial/request-files](https://fastapi.tiangolo.com/tutorial/request-files/)
- Jinja2Templates: [fastapi.tiangolo.com/advanced/templates](https://fastapi.tiangolo.com/advanced/templates/)
- GitPython Repo/Actor/index: [gitpython.readthedocs.io/en/stable/tutorial.html](https://gitpython.readthedocs.io/en/stable/tutorial.html)

#### Dependencies
- Requires: `fastapi`, `uvicorn`, `jinja2`, `python-multipart`, `gitpython`
- Does NOT require: `storage.py`, `chromadb`
- Used by: `generix-docs-web` CLI command

---

### 6. `src/generix_docs_mcp/templates/index.html`

Complete Jinja2 HTML template. All CSS and JS inlined (zero external dependencies).

**Features:**
- Document table: filename, size, upload date, sync status (color-coded badges)
- Drag-and-drop upload zone (~25 lines vanilla JS using HTML5 DnD API)
- Click-to-browse file input
- Delete button per row with `confirm()` dialog
- AJAX refresh via `GET /api/docs` (no full page reload)
- Sync status badges: green (synced), yellow (pending), red (failed)

**Jinja2 context variable:** `docs` (list of dicts from `_get_documents()`)

**Sources:**
- HTML5 Drag and Drop: [MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API/File_drag_and_drop)
- FormData + fetch: [MDN](https://developer.mozilla.org/en-US/docs/Web/API/FormData)
- Jinja2 template syntax: [jinja.palletsprojects.com](https://jinja.palletsprojects.com/en/3.1.x/templates/)

Full template code is in the web-git component spec at `~/.claude/scratch/spec-MCP/web-git-spec.md`, section 2.2.

#### Dependencies
- Requires: `web.py` to render it
- Used by: `GET /` route

---

### 7. `hooks/post-merge`

Bash script. Fires after `git pull`. Detects docs/ changes and triggers reindex.

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
REINDEX_SCRIPT="${REPO_ROOT}/scripts/reindex.py"

changed_files="$(git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD)"

if echo "$changed_files" | grep -q "^docs/"; then
    echo "[generix-docs] Detected changes in docs/ after pull. Re-indexing..."
    if [ -f "$REINDEX_SCRIPT" ]; then
        cd "$REPO_ROOT"
        uv run python "$REINDEX_SCRIPT"
        echo "[generix-docs] Re-indexing complete."
    else
        echo "[generix-docs] WARNING: reindex script not found at $REINDEX_SCRIPT"
    fi
else
    echo "[generix-docs] No doc changes detected after pull."
fi
```

**Must be executable**: `chmod +x hooks/post-merge`

**Source:** [git-scm.com/docs/githooks#_post_merge](https://git-scm.com/docs/githooks#_post_merge)

---

### 8. `hooks/post-checkout`

Bash script. Fires after branch switch. Only triggers on branch checkout (flag=1), not file checkout.

```bash
#!/usr/bin/env bash
set -euo pipefail

PREV_HEAD="$1"
NEW_HEAD="$2"
CHECKOUT_TYPE="$3"

if [ "$CHECKOUT_TYPE" != "1" ]; then exit 0; fi
if [ "$PREV_HEAD" = "$NEW_HEAD" ]; then exit 0; fi

REPO_ROOT="$(git rev-parse --show-toplevel)"
REINDEX_SCRIPT="${REPO_ROOT}/scripts/reindex.py"

changed_files="$(git diff --name-only "$PREV_HEAD" "$NEW_HEAD")"

if echo "$changed_files" | grep -q "^docs/"; then
    echo "[generix-docs] Branch switch changed docs/. Re-indexing..."
    if [ -f "$REINDEX_SCRIPT" ]; then
        cd "$REPO_ROOT"
        uv run python "$REINDEX_SCRIPT"
        echo "[generix-docs] Re-indexing complete."
    else
        echo "[generix-docs] WARNING: reindex script not found at $REINDEX_SCRIPT"
    fi
fi
```

**Must be executable**: `chmod +x hooks/post-checkout`

**Source:** [git-scm.com/docs/githooks#_post_checkout](https://git-scm.com/docs/githooks#_post_checkout)

---

### 9. `scripts/reindex.py`

CLI script that re-ingests docs into ChromaDB. The single writer to ChromaDB (besides server.py).

```
Usage:
  uv run python scripts/reindex.py                  # Full reindex
  uv run python scripts/reindex.py --file NAME      # Single file
  uv run python scripts/reindex.py --delete NAME    # Remove from index
```

**Called by:** git hooks (full reindex), web.py via subprocess (single-file or delete).

#### Complete File

```python
"""Re-index documentation into ChromaDB."""

import argparse
import os
import sys
from pathlib import Path

# Add src to path so we can import the package
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "src"))

from generix_docs_mcp import storage, ingestion

DOCS_DIR = Path(os.environ.get("GENERIX_DOCS_PATH", str(Path(__file__).resolve().parent.parent / "docs")))


def reindex_all() -> None:
    """Full reindex: delete everything, re-ingest all files in docs/."""
    if not DOCS_DIR.exists():
        print(f"[reindex] docs directory not found: {DOCS_DIR}")
        return

    # Get current indexed sources to delete orphans
    existing = {d["source"] for d in storage.list_documents()}

    indexed = set()
    for file_path in sorted(DOCS_DIR.iterdir()):
        if file_path.is_file() and file_path.suffix.lower() in ingestion.SUPPORTED_EXTENSIONS:
            try:
                reindex_file(file_path.name)
                indexed.add(file_path.name)
            except Exception as e:
                print(f"[reindex] ERROR processing {file_path.name}: {e}")

    # Remove orphaned entries (indexed but no longer on disk)
    orphaned = existing - indexed
    for source in orphaned:
        try:
            storage.delete_by_source(source)
            print(f"[reindex] Removed orphaned index: {source}")
        except ValueError:
            pass

    print(f"[reindex] Complete. Indexed: {len(indexed)}, Removed orphans: {len(orphaned)}")


def reindex_file(filename: str) -> None:
    """Re-index a single file (upsert semantics)."""
    file_path = DOCS_DIR / filename
    if not file_path.is_file():
        print(f"[reindex] File not found: {file_path}")
        return

    chunks, metadatas = ingestion.ingest_file(file_path=file_path)
    count = storage.add_document_chunks(source=filename, chunks=chunks, metadatas=metadatas)
    print(f"[reindex] Indexed {filename}: {count} chunks")


def delete_file(filename: str) -> None:
    """Remove a file from the index."""
    try:
        count = storage.delete_by_source(source=filename)
        print(f"[reindex] Deleted {filename}: {count} chunks removed")
    except ValueError:
        print(f"[reindex] {filename} not found in index")


def main() -> None:
    parser = argparse.ArgumentParser(description="Re-index documentation")
    parser.add_argument("--file", help="Single file to re-index (filename only)")
    parser.add_argument("--delete", help="File to remove from index (filename only)")
    args = parser.parse_args()

    if args.delete:
        delete_file(args.delete)
    elif args.file:
        reindex_file(args.file)
    else:
        reindex_all()


if __name__ == "__main__":
    main()
```

#### Dependencies
- Requires: `storage.py`, `ingestion.py`
- Used by: `hooks/post-merge`, `hooks/post-checkout`, `web.py`

---

### 10. `scripts/setup.py`

One-time setup after cloning:
1. `git config core.hooksPath ./hooks`
2. Pre-download embedding model (~90MB)
3. Build initial index from existing docs/

Full code is in the web-git component spec at `~/.claude/scratch/spec-MCP/web-git-spec.md`, section 6.1.

---

### 11. `.mcp.json.example`

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

---

### 12. `.gitignore`

```gitignore
chroma_db/
__pycache__/
*.py[cod]
*$py.class
.venv/
.idea/
.vscode/
*.swp
*.swo
.DS_Store
Thumbs.db
.env
.env.local
dist/
build/
*.egg-info/
.pytest_cache/
.coverage
htmlcov/
```

---

## Test Plan

| Test | Type | What It Verifies |
|------|------|-----------------|
| `test_storage_add_and_query` | Unit | Add chunks, query returns them with correct scores |
| `test_storage_upsert_semantics` | Unit | Re-adding same source replaces old chunks |
| `test_storage_delete_by_source` | Unit | Delete removes all chunks for a source |
| `test_storage_fetch_document_text` | Unit | Reconstructed text matches chunks in order |
| `test_storage_get_stats` | Unit | Stats reflect actual collection state |
| `test_storage_drift_detection` | Unit | get_stats with docs_path detects not_indexed and orphaned |
| `test_ingestion_markdown` | Unit | .md file extracts and chunks correctly |
| `test_ingestion_text` | Unit | .txt file extracts correctly |
| `test_ingestion_pdf` | Unit | .pdf extracts via pymupdf4llm (skip if not installed) |
| `test_ingestion_html` | Unit | .html converts headings to markdown |
| `test_ingestion_docx` | Unit | .docx maps heading styles to markdown |
| `test_ingestion_unsupported` | Unit | Raises ValueError for .xyz files |
| `test_ingestion_missing_dep` | Unit | Raises ImportError with install hint |
| `test_ingestion_chunk_metadata` | Unit | Metadata has correct keys and types |
| `test_server_search_docs` | Integration | search_docs returns results from indexed content |
| `test_server_list_documents` | Integration | list_documents returns indexed docs |
| `test_server_add_document` | Integration | add_document ingests and indexes a file |
| `test_server_remove_document` | Integration | remove_document deletes from index |
| `test_server_fetch_document` | Integration | fetch_document returns full text |
| `test_server_get_stats` | Integration | get_stats returns correct statistics |
| `test_web_upload_valid_file` | Integration | Upload succeeds, file written to DOCS_DIR |
| `test_web_upload_invalid_extension` | Integration | Upload of .xyz returns 400 |
| `test_web_delete_existing` | Integration | DELETE removes file from disk |
| `test_web_delete_nonexistent` | Integration | DELETE of missing file returns 404 |
| `test_web_api_docs` | Integration | GET /api/docs returns JSON list of documents |

---

## Implementation Order

1. **`pyproject.toml`** - Set up the project and install dependencies. Run `uv sync --extra all-formats --extra dev` to verify.

2. **`storage.py`** - Core data layer. Implement `get_collection()` first, then `add_document_chunks()` and `query()`. Write `test_storage.py` and verify with a manual test (add a chunk, query for it).

3. **`ingestion.py`** - Start with `_extract_markdown()` and `_extract_text()` only. Implement the two-stage chunking pipeline. Write `test_ingestion.py`. Then add PDF, HTML, DOCX extractors one at a time.

4. **`server.py`** - MCP server. Start with `search_docs` and `list_documents`. Test by running `uv run generix-docs` and connecting from Claude Code. Then add the remaining 4 tools.

5. **`scripts/reindex.py`** - CLI reindex tool. Test with `uv run python scripts/reindex.py` after placing a .md file in docs/.

6. **`web.py` + `templates/index.html`** - Web UI. Start with `GET /` (list) and `POST /upload` (no git). Then add git integration, delete, and AJAX refresh.

7. **Git integration** - Add `_git_add_commit_push()` and `_push_with_retry()` to web.py. Create the hooks. Test with `git config core.hooksPath ./hooks` and a pull.

8. **`scripts/setup.py`** - One-time setup script. Test the full first-time developer experience.

9. **`.mcp.json.example` + `.gitignore` + `README.md`** - Documentation and config templates.

---

## Phase 3 Requirements

### Recommended Team for Implementation

| Agent | Why Needed | Files to Touch |
|-------|-----------|---------------|
| Backend developer | Core data layer + MCP server | `storage.py`, `ingestion.py`, `server.py`, `pyproject.toml` |
| Frontend developer | Web UI + templates | `web.py`, `templates/index.html` |
| DevOps developer | Git integration + hooks + scripts | `hooks/*`, `scripts/*`, git integration in `web.py` |

### Component Spec Reference

Detailed code for each component (complete function implementations, full HTML template, complete bash scripts) is available in the scratch files:

- `~/.claude/scratch/spec-MCP/core-spec.md` - storage.py + ingestion.py with all API citations
- `~/.claude/scratch/spec-MCP/server-spec.md` - server.py complete file with all 6 tools
- `~/.claude/scratch/spec-MCP/web-git-spec.md` - web.py, index.html, hooks, scripts with all code
