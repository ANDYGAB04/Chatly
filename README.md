# Chatly

Chatly is a web-based conversational AI assistant built with FastAPI, LangGraph, and
Google Gemini. It provides a ChatGPT-style interface with streaming responses,
conversation history, document question answering, web search, calculator support,
and per-conversation long-term memory.

## Features

- Streaming chat responses over Server-Sent Events (SSE)
- Google Gemini model selection from the web interface
- Agentic tool use through LangGraph:
  - Current web information with Tavily Search
  - Uploaded-document search with retrieval-augmented generation (RAG)
  - Safe, simple mathematical calculations
  - Saving and recalling user memories
- Upload and index `.pdf`, `.docx`, `.txt`, `.md`, `.py`, and `.csv` files
- Conversation and message history stored in SQLite
- LangGraph checkpoints stored in SQLite so graph state can persist between requests
- Local ChromaDB vector storage for document embeddings

## Requirements

- Python 3.11 or newer
- A Google AI API key for Gemini models and embeddings
- A Tavily API key if you want to use web search

## Installation

Clone the repository and create a virtual environment:

```bash
git clone https://github.com/ANDYGAB04/Chatly.git
cd Chatly

python -m venv .venv
```

Activate the environment:

**Windows PowerShell**

```powershell
.venv\Scripts\Activate.ps1
```

**macOS/Linux**

```bash
source .venv/bin/activate
```

Install the dependencies:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## Configuration

Create a `.env` file in the project root. `.env` is ignored by Git and must never
be committed.

```dotenv
GOOGLE_API_KEY=your_google_ai_api_key
TAVILY_API_KEY=your_tavily_api_key

# Optional
GEMINI_MODEL=gemini-3.8-flash
APP_HOST=127.0.0.1
APP_PORT=8000
```

`GOOGLE_API_KEY` is used by the Gemini chat model and the Gemini embedding model.
`TAVILY_API_KEY` is required for requests that use web search. The application
defaults to `127.0.0.1:8000` and uses `gemini-3.8-flash` when no model or host/port
override is supplied.

## Running the application

Start the development server from the repository root:

```bash
python -m uvicorn app:app --reload
```

Alternatively, run the application module directly:

```bash
python app.py
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in a browser.

On first startup, Chatly creates the `data/` and `uploads/` directories and
initializes the SQLite database. Uploaded documents are embedded into the local
`chroma_db/` directory.

## Using Chatly

1. Start a new conversation in the browser.
2. Select one of the available Gemini models.
3. Enter a message and send it. Responses are streamed into the interface.
4. Use the attachment button to upload a supported document.
5. Ask about the uploaded document to search only the documents associated with
   the current conversation.
6. Ask Chatly to remember a preference or fact, then ask what it remembers later.
7. Ask for current information to let the agent use Tavily web search.

The currently offered models are:

- `gemini-3.8-flash`
- `gemini-3.5-pro`
- `gemini-3.5-flash-lite`
- `gemini-3.1-flash-lite`
- `gemini-3.1-pro`

The backend validates model names and falls back to `GEMINI_MODEL` when an unknown
model is selected.

## API endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/` | Serves the web interface |
| `GET` | `/conversations` | Lists conversations ordered by last update |
| `GET` | `/history/{thread_id}` | Returns message history for a conversation |
| `POST` | `/upload` | Uploads and indexes a document for a conversation |
| `POST` | `/chat/stream` | Streams an agent response as SSE events |

### Chat request

`POST /chat/stream` expects JSON in this shape:

```json
{
  "message": "Summarize the document I uploaded.",
  "thread_id": "conversation-id",
  "model": "gemini-3.8-flash"
}
```

The response uses `text/event-stream`. Normal response tokens are sent as:

```text
data: {"token":"Hello"}
```

The stream ends with:

```text
data: {"done":true}
```

### Upload request

`POST /upload` expects a multipart form with:

- `file`: the document to upload
- `thread_id`: the conversation that owns the document

Supported extensions are `.pdf`, `.docx`, `.txt`, `.md`, `.py`, and `.csv`.

## Project structure

```text
.
├── app.py             # FastAPI routes, SSE streaming, and application startup
├── agent.py           # LangGraph workflow and Gemini model configuration
├── database.py        # SQLAlchemy models and SQLite persistence helpers
├── rag.py             # File extraction, chunking, embeddings, and ChromaDB search
├── tools.py           # Calculator, web search, memory, and document tools
├── test.py            # Small manual LangGraph streaming smoke test
├── requirements.txt   # Python dependencies
├── templates/
│   └── index.html     # Chatly browser interface
├── data/              # Runtime SQLite databases
├── uploads/           # Uploaded source documents
└── chroma_db/         # Persistent ChromaDB vector index
```

## Testing and development

The repository currently includes a manual streaming smoke test:

```bash
python test.py
```

The test initializes the database, creates a `gemini-3.8-flash` agent, and streams
an example response. It requires the same environment variables as the application.

For local development, keep the server running with `--reload` and use the browser
interface to exercise chat, uploads, history, and tool calls.

## Data and reset

Runtime state is stored locally:

- `data/chatbot_memory.db` stores conversations, chat messages, and long-term memory.
- `data/langgraph_checkpoints.sqlite` stores LangGraph checkpoints.
- `chroma_db/` stores document embeddings and metadata.
- `uploads/` stores uploaded files.

To reset local application state, stop the server and remove those runtime files
or directories, then start the application again. Do not delete `.env` unless you
have backed up your credentials.

## Security notes

- Keep API keys in `.env` or another secret manager; never commit them.
- Do not expose the development server directly to the public internet without
  authentication, HTTPS, request limits, and upload-size/type controls.
- Uploaded files and locally persisted conversations may contain sensitive data.
  Protect the project directory and back up or delete runtime data according to
  your requirements.

## License

No license file is currently included in this repository.
