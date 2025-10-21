# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for querying course materials. The system uses ChromaDB for vector storage, Anthropic's Claude API for AI generation with tool calling, and provides a FastAPI web interface.

## Common Commands

### Development
```bash
# Install dependencies
uv sync

# Run the application
cd backend
uv run uvicorn app:app --reload --port 8000

# Or use the convenience script
./run.sh
```

### Access Points
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

### Core Components

The system follows a **component-based architecture** with clear separation of concerns:

1. **RAGSystem** (`backend/rag_system.py`): Main orchestrator that coordinates all components
   - Initializes and manages: DocumentProcessor, VectorStore, AIGenerator, SessionManager, ToolManager
   - Processes queries through tool-augmented AI generation
   - Handles document ingestion from single files or folders

2. **VectorStore** (`backend/vector_store.py`): ChromaDB wrapper with two-collection design
   - `course_catalog`: Stores course metadata (title, instructor, lessons) for semantic course name matching
   - `course_content`: Stores chunked course content with metadata (course_title, lesson_number, chunk_index)
   - Smart search: Resolves fuzzy course names via semantic search, then filters content by course/lesson

3. **Tool System** (`backend/search_tools.py`): Anthropic tool calling integration
   - `CourseSearchTool`: Implements the search_course_content tool with optional course_name and lesson_number filters
   - `ToolManager`: Registers tools and handles execution during AI interaction
   - Tool results are tracked for source attribution

4. **AIGenerator** (`backend/ai_generator.py`): Claude API integration
   - Uses claude-sonnet-4-20250514 model
   - Implements tool calling workflow: initial response → tool execution → final response
   - Maintains conversation history for context-aware responses
   - System prompt emphasizes brief, educational responses with one search per query

5. **DocumentProcessor** (`backend/document_processor.py`): Document parsing and chunking
   - Parses structured course documents (format: Course Title/Link/Instructor → Lesson N: Title → content)
   - Sentence-based chunking with configurable size (800 chars) and overlap (100 chars)
   - Adds lesson context prefixes to chunks for better semantic search

6. **SessionManager** (`backend/session_manager.py`): Conversation state management
   - Tracks user-assistant message exchanges per session
   - Maintains limited history (MAX_HISTORY=2 exchanges, configurable in `backend/config.py`)

### Data Flow

**Query Processing:**
```
User Query → FastAPI endpoint → RAGSystem.query()
  → AIGenerator.generate_response() with tools
  → Claude decides to call search_course_content tool
  → CourseSearchTool.execute()
    → VectorStore.search() [resolves course name → searches content]
  → Results formatted and returned to Claude
  → Claude generates final response
  → SessionManager updates history
  → Response + sources returned to user
```

**Document Ingestion:**
```
File → DocumentProcessor.process_course_document()
  → Extracts Course + Lessons + CourseChunks
  → VectorStore.add_course_metadata() [to course_catalog]
  → VectorStore.add_course_content() [to course_content]
```

### Configuration

All settings centralized in `backend/config.py`:
- `ANTHROPIC_API_KEY`: Set via .env file (required)
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: "./chroma_db"

### Frontend

Simple vanilla JavaScript SPA in `frontend/`:
- `index.html`: Main interface
- `script.js`: API communication and UI updates
- `style.css`: Styling

## Key Implementation Details

### Two-Collection Strategy
The system uses separate collections to optimize for different search patterns:
- Course catalog enables fuzzy course name matching ("MCP" → "Model Context Protocol")
- Course content stores the actual material with precise filtering by resolved course name and lesson number

### Tool-Based Search
Unlike traditional RAG where context is always retrieved, this system uses Anthropic's tool calling to let Claude decide when to search. This reduces unnecessary lookups for general knowledge questions.

### Lesson Context in Chunks
Chunks include lesson metadata in their content (e.g., "Course X Lesson Y content: ...") to improve semantic search relevance when users ask lesson-specific questions.

### Startup Document Loading
FastAPI `startup_event` in `backend/app.py` automatically loads documents from `../docs` on server start, skipping already-indexed courses to avoid duplication.

## Troubleshooting

### macOS with Anaconda Python

If you encounter dependency conflicts when using `uv sync` on macOS (especially with older macOS versions):

**Option 1: Use system installation**
```bash
uv pip install --system chromadb==1.0.15 anthropic==0.58.2 "sentence-transformers==5.0.0" fastapi==0.116.1 uvicorn==0.35.0 python-multipart==0.0.20 python-dotenv==1.1.1

# If you get NumPy 2.x compatibility errors:
uv pip install --system "numpy<2"

# If you get Keras/TensorFlow errors:
uv pip install --system tf-keras
```

**Option 2: Run with Anaconda Python directly**
```bash
cd backend
/path/to/anaconda3/bin/python -m uvicorn app:app --reload --port 8000
```
- always use uv to run the server do not use pip directly
- make sure to use uv to manage all dependencies
- use uv to run Python files