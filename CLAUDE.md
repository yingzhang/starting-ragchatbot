# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for course materials using ChromaDB for vector storage and Anthropic's Claude API for intelligent responses. The system uses tool-calling with semantic search to answer questions about course content.

## Running the Application

### Standard Method
```bash
./run.sh
```

### Alternative (using venv instead of uv)
If `uv` has network issues downloading Python:
```bash
cd backend
python3 -m venv venv
source venv/bin/activate
pip install "numpy<2"  # CRITICAL: Must use numpy 1.x for compatibility
pip install chromadb==1.0.15 anthropic==0.58.2 sentence-transformers==5.0.0 fastapi==0.116.1 uvicorn==0.35.0 python-multipart==0.0.20 python-dotenv==1.1.1
uvicorn app:app --reload --port 8000
```

### Environment Setup
Create `.env` in root directory:
```
ANTHROPIC_API_KEY=your_key_here
```

## Code Quality Tools

This project uses code quality tools to maintain consistent formatting and catch potential issues:

- **black**: Automatic code formatting (line length: 88)
- **isort**: Import statement organization
- **flake8**: Style guide enforcement and linting
- **mypy**: Static type checking

### Running Quality Checks

**Auto-format code** (modifies files):
```bash
cd backend
source venv312/bin/activate
isort .
black .
```

**Check code quality** (read-only):
```bash
cd backend
source venv312/bin/activate
isort --check-only .
black --check .
flake8 .
mypy .
```

### Installation

Code quality tools are listed in `pyproject.toml` under `[dependency-groups]`. Install with:
```bash
cd backend
source venv312/bin/activate
pip install black flake8 isort mypy
```

## Architecture

### System Components

**Core Orchestrator** (`rag_system.py`):
- `RAGSystem` class coordinates all components
- Manages document ingestion workflow
- Handles query processing with tool execution
- Methods: `add_course_document()`, `add_course_folder()`, `query()`, `get_course_analytics()`

**Data Models** (`models.py`):
- `Course`: Represents course with title (unique ID), link, instructor, and lessons list
- `Lesson`: Contains lesson_number, title, and lesson_link
- `CourseChunk`: Text chunk with course_title, lesson_number, chunk_index metadata

### RAG Pipeline Flow

1. **Document Ingestion** (`document_processor.py`)
   - Reads course documents with specific format (Course Title, Link, Instructor, Lessons)
   - Chunks text into overlapping segments (800 chars, 100 char overlap)
   - Adds lesson context to first chunk of each lesson
   - Returns `Course` object and list of `CourseChunk` objects

2. **Vector Storage** (`vector_store.py`)
   - Two ChromaDB collections: `course_catalog` (metadata) and `course_content` (chunks)
   - Uses `all-MiniLM-L6-v2` sentence transformer for embeddings
   - Semantic course name resolution via vector search on catalog
   - Search flow: fuzzy course name → exact title filter → content retrieval

3. **Tool-Based Search** (`search_tools.py`)
   - `CourseSearchTool` implements Tool interface with `get_tool_definition()` and `execute()`
   - Claude uses `search_course_content` tool with auto tool choice
   - Supports fuzzy course name matching and lesson filtering
   - `ToolManager` tracks sources (with URLs) for UI display

4. **AI Generation** (`ai_generator.py`)
   - Tool execution with agentic loop: query → tool use → results → final response
   - System prompt enforces: one search maximum, no meta-commentary, brief responses
   - Temperature 0, max_tokens 800
   - `_handle_tool_execution()` manages multi-turn tool calling workflow

5. **Session Management** (`session_manager.py`)
   - Maintains conversation history (MAX_HISTORY=2 message pairs)
   - Session-based context for follow-up questions
   - Each session stores `Message` objects with role and content

### Key Design Patterns

**Tool-Based RAG vs Direct Retrieval**: The system uses Claude's tool calling rather than injecting search results into prompts. This gives Claude control over when to search vs use general knowledge.

**Two-Tier Vector Search**: Course metadata in `course_catalog` enables semantic course name resolution ("MCP" → "Introduction to Model Context Protocol"), then content search in `course_content` with exact title filtering.

**Chunk Contextualization**: First chunk of each lesson includes "Lesson N content:" prefix. All chunks store `course_title` and `lesson_number` metadata for filtering.

### Extending the Tool System

To add new tools for Claude to use:

1. **Create Tool Class** in `search_tools.py`:
   ```python
   class MyCustomTool(Tool):
       def get_tool_definition(self) -> Dict[str, Any]:
           return {
               "name": "my_tool_name",
               "description": "What this tool does",
               "input_schema": {
                   "type": "object",
                   "properties": {"param": {"type": "string", "description": "..."}},
                   "required": ["param"]
               }
           }

       def execute(self, param: str) -> str:
           # Tool implementation
           return "result"
   ```

2. **Register Tool** in `rag_system.py:__init__()`:
   ```python
   custom_tool = MyCustomTool()
   self.tool_manager.register_tool(custom_tool)
   ```

3. Tool will be automatically included in Claude's tool definitions and available during query processing

## Frontend Architecture

**Technology**: Vanilla JavaScript (no framework)

**Structure**:
- `frontend/index.html`: Single-page application interface
- `frontend/script.js`: API communication and DOM manipulation
- `frontend/style.css`: Styling and layout

**Integration**: FastAPI serves static files from `../frontend` at root path (`/`). API endpoints are available at `/api/*`. Single server handles both frontend and backend on port 8000.

## Document Format

**Supported File Types**: `.txt`, `.pdf`, `.docx`

Expected format for course documents in `docs/`:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [title]
...
```

**Document Loading**:
- On server startup, documents from `../docs` are automatically loaded into ChromaDB
- Existing courses are skipped (idempotent loading - safe to restart)
- To force rebuild: use `clear_existing=True` in `add_course_folder()` call (in `app.py:100`)

## Development Workflow

### Adding New Documents
1. Place course document in `docs/` folder (`.txt`, `.pdf`, or `.docx`)
2. Restart server - new courses auto-load, existing courses skip
3. Verify via `GET /api/courses` or check console output

### Rebuilding Vector Database
To completely rebuild ChromaDB from scratch:
1. Stop the server
2. Delete `backend/chroma_db/` directory
3. Restart server - all docs will be reprocessed

Alternatively, modify `app.py:100` to use `clear_existing=True`:
```python
courses, chunks = rag_system.add_course_folder(docs_path, clear_existing=True)
```

### Manual Testing via API
```bash
# Query the system
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is RAG?", "session_id": "test_session"}'

# Get course statistics
curl http://localhost:8000/api/courses
```

### Project Structure
```
.
├── backend/
│   ├── app.py              # FastAPI application & API endpoints
│   ├── rag_system.py       # Main orchestrator
│   ├── document_processor.py  # Document parsing & chunking
│   ├── vector_store.py     # ChromaDB interface
│   ├── search_tools.py     # Tool definitions & execution
│   ├── ai_generator.py     # Claude API interaction
│   ├── session_manager.py  # Conversation history
│   ├── models.py           # Data models (Course, Lesson, CourseChunk)
│   ├── config.py           # Configuration settings
│   └── chroma_db/          # Vector database (auto-generated)
├── frontend/
│   ├── index.html          # Web UI
│   ├── script.js           # Frontend logic
│   └── style.css           # Styling
├── docs/                   # Course documents (auto-loaded)
└── .env                    # Environment variables
```

## Configuration (`config.py`)

- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2"
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation turns
- `CHROMA_PATH`: "./chroma_db"

## Access Points

When application is running:
- **Web Interface**: http://localhost:8000
- **API Documentation**: http://localhost:8000/docs

### API Endpoints (`app.py`)

- `POST /api/query`: Submit query
  - Request: `{query: string, session_id?: string}`
  - Response: `{answer: string, sources: string[], session_id: string}`
- `GET /api/courses`: Get course analytics
  - Response: `{total_courses: int, course_titles: string[]}`

## Common Issues

**HuggingFace Connection Timeout**: On startup, sentence-transformers downloads `all-MiniLM-L6-v2` model. If huggingface.co is blocked:
```bash
export HF_ENDPOINT=https://hf-mirror.com
```

**NumPy Version Conflict**: torch/onnxruntime require numpy 1.x. Always install with:
```bash
pip install "numpy<2"
```
