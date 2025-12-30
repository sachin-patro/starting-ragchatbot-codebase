# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup
```bash
# Install dependencies (uses uv package manager)
uv sync

# Create environment file
cp .env.example .env
# Add your ANTHROPIC_API_KEY to .env
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access points:
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

### Environment Requirements
- Python 3.13+
- `uv` package manager
- Valid `ANTHROPIC_API_KEY` in `.env` file

## System Architecture

This is a **Retrieval-Augmented Generation (RAG) system** for querying course materials. The architecture uses a tool-based agentic pattern where Claude autonomously decides when to search course content.

### Core Flow Pattern

**User Query → Claude (with tools) → Tool Execution → Claude (with results) → Response**

Unlike traditional RAG systems that immediately search on every query, this system:
1. Gives Claude the `search_course_content` tool but doesn't force its use
2. Claude analyzes the query and decides if searching is necessary
3. If searching, Claude executes the tool with appropriate parameters
4. Claude synthesizes the final answer using retrieved context

### Component Responsibilities

**RAGSystem** (`rag_system.py`) - Main orchestrator
- Coordinates all components
- Manages tool registry via `ToolManager`
- Delegates to specialized components rather than implementing logic directly

**DocumentProcessor** (`document_processor.py`) - Text extraction & chunking
- Expects course documents with specific format: `Course Title:`, `Course Instructor:`, `Lesson N:` headers
- Chunks at 800 chars with 100 char overlap (configurable in `config.py`)
- Preserves sentence boundaries during chunking
- Returns `Course` and `CourseChunk` objects

**VectorStore** (`vector_store.py`) - Dual ChromaDB collections
- `course_catalog` collection: Course metadata for semantic course name resolution
- `course_content` collection: Chunked course material for content search
- Semantic course name matching allows partial matches (e.g., "MCP" finds "Introduction to MCP Servers")
- Filtering by `course_name` and/or `lesson_number` supported

**AIGenerator** (`ai_generator.py`) - Claude API wrapper
- Pre-builds static system prompt and base API parameters (performance optimization)
- Temperature 0 for deterministic responses
- Handles tool execution loop: receives tool_use blocks → executes via ToolManager → sends results back
- Returns final text response after tool execution completes

**ToolManager + CourseSearchTool** (`search_tools.py`) - Tool registry pattern
- `Tool` abstract base class defines interface
- `CourseSearchTool` implements search with semantic course name matching
- `ToolManager` maintains registry and routes execution
- Tools track `last_sources` for UI source attribution

**SessionManager** (`session_manager.py`) - Conversation state
- Stores last 2 message exchanges per session (configurable via `MAX_HISTORY`)
- Sessions are in-memory (not persisted)
- Session IDs follow pattern: `session_1`, `session_2`, etc.

### Key Design Decisions

**Why tool-based search instead of direct RAG?**
- Enables agentic behavior: Claude can answer general questions without searching
- Allows multi-turn reasoning: Claude can search multiple times with different queries
- More flexible: Claude can choose to search specific courses or lessons

**Why two ChromaDB collections?**
- Catalog collection enables semantic course name resolution (user says "MCP", finds "Introduction to MCP Servers")
- Separates metadata queries from content queries for cleaner architecture

**Why store sources in tools?**
- Tools track `last_sources` attribute to provide source attribution to the frontend
- `ToolManager.get_last_sources()` retrieves and `reset_sources()` clears after each query
- Decouples source tracking from main response flow

**Why session-based history with limit?**
- Prevents token bloat while maintaining context
- Default 2 exchanges balances context vs cost

### FastAPI Application (`app.py`)

**Endpoints:**
- `POST /api/query`: Main query endpoint (auto-creates session if not provided)
- `GET /api/courses`: Returns course statistics

**Startup behavior:**
- Loads all documents from `../docs` folder
- Deduplicates by course title (won't reload existing courses)
- Supports `.pdf`, `.docx`, `.txt` formats

**Static file serving:**
- Serves frontend from `../frontend` directory
- `DevStaticFiles` class adds no-cache headers for development

### Configuration (`config.py`)

All tuneable parameters:
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: ./chroma_db

### Course Document Format

Documents MUST follow this structure:
```
Course Title: [Title]
Course Link: [URL]
Course Instructor: [Name]

Lesson 0: Introduction
[Content...]

Lesson 1: [Title]
Lesson Link: [URL]
[Content...]
```

The `DocumentProcessor` parses this format to extract structured metadata.

### Adding New Tools

To add a new tool:
1. Create class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` (returns Anthropic tool schema)
3. Implement `execute(**kwargs)` (tool logic)
4. Register in `RAGSystem.__init__()` via `self.tool_manager.register_tool()`

Example:
```python
class NewTool(Tool):
    def get_tool_definition(self):
        return {
            "name": "tool_name",
            "description": "What the tool does",
            "input_schema": {...}
        }

    def execute(self, **kwargs):
        # Tool logic here
        return "result"
```

### Frontend Architecture

**Technology:** Vanilla JavaScript (no frameworks)

**Key files:**
- `index.html`: Two-column layout (sidebar + chat)
- `script.js`: API calls, DOM manipulation, markdown rendering
- `style.css`: Responsive design with collapsible sections

**State management:**
- `currentSessionId`: Active conversation session
- Suggested questions pre-populate common queries
- Source attribution shown in collapsible details

### Data Persistence

**ChromaDB storage:**
- Location: `./chroma_db` (configurable)
- Persisted to disk (survives restarts)
- Two collections as described above

**Session storage:**
- In-memory only (cleared on restart)
- No database persistence

### Common Workflows

**Adding new course documents:**
1. Place files in `docs/` folder
2. Restart server (startup event loads new documents)
3. Existing courses are automatically skipped (deduplication by title)

**Modifying chunking strategy:**
1. Update `CHUNK_SIZE` and `CHUNK_OVERLAP` in `config.py`
2. Clear existing data: `rag_system.add_course_folder(docs_path, clear_existing=True)`
3. Or manually delete `./chroma_db` directory

**Changing AI model:**
1. Update `ANTHROPIC_MODEL` in `config.py`
2. Restart server (no re-indexing needed)

**Debugging search results:**
1. Check ChromaDB collections via `vector_store.py` methods
2. Test search directly: `rag_system.vector_store.search(query="test")`
3. Inspect tool execution in `AIGenerator._handle_tool_execution()`
