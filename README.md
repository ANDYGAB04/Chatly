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
- Docker image and CI/CD pipeline (GitHub Actions, Amazon ECR, EC2) for deployment

## Requirements

- Python 3.11 or newer
- A Google AI API key for Gemini models and embeddings
- A Tavily API key if you want to use web search
- Docker (only for container or EC2 deployment)

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

# Optional: LangSmith tracing
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=your_langsmith_api_key
LANGSMITH_PROJECT=your_project_name
```

`GOOGLE_API_KEY` is used by the Gemini chat model and the Gemini embedding model.
It is **required at startup**: the embedding client is created when `rag.py` is
imported, so the application will crash immediately if the key is missing or empty.
`TAVILY_API_KEY` is required for requests that use web search. The application
defaults to `127.0.0.1:8000` and uses `gemini-3.8-flash` when no model or host/port
override is supplied. The `LANGSMITH_*` variables are optional and only enable
tracing.

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

## Running with Docker

The container serves the app with Uvicorn on port **8080**, bound to `0.0.0.0`.

Build the image:

```bash
docker build -t chatly .
```

Run it, passing the configuration as environment variables:

```bash
docker run -d \
  --name agentic-chatbot \
  --restart unless-stopped \
  -p 8080:8080 \
  -e GOOGLE_API_KEY="your_google_ai_api_key" \
  -e TAVILY_API_KEY="your_tavily_api_key" \
  chatly
```

Then open [http://localhost:8080](http://localhost:8080).

Check that it is running and read its logs:

```bash
docker ps
docker logs --tail 50 agentic-chatbot
```

If the container shows `Restarting` in `docker ps`, it is crash-looping. The logs
will show the cause. The most common one is a missing or empty `GOOGLE_API_KEY`.

### Persisting data

By default, everything the app writes (`data/`, `uploads/`, `chroma_db/`) lives
inside the container and is **lost when the container is removed**, which happens
on every redeploy. To keep it, mount volumes:

```bash
docker run -d \
  --name agentic-chatbot \
  -p 8080:8080 \
  -v chatly-data:/app/data \
  -v chatly-uploads:/app/uploads \
  -v chatly-chroma:/app/chroma_db \
  -e GOOGLE_API_KEY="your_google_ai_api_key" \
  -e TAVILY_API_KEY="your_tavily_api_key" \
  chatly
```

## Deployment (GitHub Actions, Amazon ECR, EC2)

The workflow in `.github/workflows/` deploys on every push to `main`:

```text
push to main
   │
   ├─ Continuous-Integration  (GitHub-hosted runner, ubuntu-latest)
   │     build Docker image → push to Amazon ECR as :latest
   │
   └─ Continuous-Deployment   (self-hosted runner on EC2)
         pull image from ECR → stop/remove old container
         → run new container on port 8080 → prune old images
```

### 1. AWS setup

- Create an **ECR repository** (for example `chatly`) in your chosen region.
- Launch an **EC2 instance** (Ubuntu) and install Docker.
- Add the `ubuntu` user to the docker group so the runner can use Docker, then
  log out and back in:

  ```bash
  sudo usermod -aG docker ubuntu
  ```

- Create an IAM user with ECR permissions (push for CI, pull for CD) and generate an
  access key for it.
- In the instance's **security group**, add an inbound rule for **TCP 8080**
  (restrict the source to your IP where possible).

### 2. Self-hosted runner on EC2

In the GitHub repository go to **Settings → Actions → Runners → New self-hosted
runner** (Linux, x64) and follow the download and `./config.sh` commands on the
instance. Then install it as a service so it survives disconnects and reboots:

```bash
cd ~/actions-runner
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

The runner should appear as **Idle** in the repository's runner list.

### 3. Repository secrets

Add these under **Settings → Secrets and variables → Actions → Repository
secrets**. Names are case-sensitive and must match the workflow exactly. A
misnamed secret does not fail the run; it silently passes an empty value to the
container.

| Secret | Description |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | IAM secret access key |
| `AWS_DEFAULT_REGION` | AWS region of the ECR repository and instance, e.g. `eu-north-1` |
| `ECR_REPO` | ECR repository **name only** (e.g. `chatly`), not the full URI |
| `GOOGLE_API_KEY` | Google AI API key |
| `GOOGLE_MODEL` | Gemini model name |
| `TAVILY_API_KEY` | Tavily API key |
| `LANGSMITH_TRACING` | `true` or `false` |
| `LANGSMITH_ENDPOINT` | LangSmith endpoint URL |
| `LANGSMITH_API_KEY` | LangSmith API key |
| `LANGSMITH_PROJECT` | LangSmith project name |

### 4. Deploy and verify

Push to `main`, or re-run the latest workflow from the **Actions** tab. After the
`Continuous-Deployment` job succeeds, on the instance:

```bash
docker ps                          # status should be "Up", not "Restarting"
docker logs --tail 20 agentic-chatbot
curl -I http://localhost:8080
```

Then open `http://<instance-public-ip>:8080`. Note that a stopped and restarted
instance gets a new public IP unless you attach an Elastic IP.

### Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Browser shows `ERR_CONNECTION_REFUSED` | Nothing is listening on 8080: the container is not running or is crash-looping. Check `docker ps` and `docker logs`. |
| Browser times out | The security group does not allow inbound TCP 8080. |
| Container is `Restarting` with "API key required" | `GOOGLE_API_KEY` is empty. Check the secret name, then **re-run the workflow**, since secrets are only injected at deploy time. |
| `docker: permission denied` in the CD job | The runner user is not in the `docker` group. Add it and restart the runner service. |
| CD job stays queued | The self-hosted runner is offline or its labels do not match `runs-on`. |
| `ModuleNotFoundError` in the logs | A package is missing from `requirements.txt`. |

### Shutting down and cleaning up

To avoid charges when you are done:

1. Stop or terminate the EC2 instance. Stopping pauses compute billing but the
   disk still costs a little; terminating deletes everything.
2. Delete leftover EBS volumes, Elastic IPs, and snapshots.
3. Delete the ECR repository.
4. Disable the workflow (or remove the runner), so a push to `main` does not queue
   a deployment with no runner to execute it.
5. Delete or rotate the IAM access key and any API keys stored as secrets.

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
├── Dockerfile         # Container image definition
├── .github/
│   └── workflows/     # CI/CD pipeline (build, push to ECR, deploy to EC2)
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
or directories, then start the application again. In Docker, remove the container
(and any mounted volumes) to reset. Do not delete `.env` unless you have backed up
your credentials.

## Security notes

- Keep API keys in `.env`, GitHub Actions secrets, or another secret manager; never
  commit them.
- Do not expose the application directly to the public internet without
  authentication, HTTPS, request limits, and upload-size/type controls. A public
  EC2 port 8080 has none of these, so restrict the security group to your own IP
  unless you have added them.
- Environment variables passed with `docker run -e` are visible through
  `docker inspect`. Do not paste that output anywhere public.
- Rotate the AWS access key and API keys if they are ever exposed, and delete the
  AWS key when the deployment is shut down.
- Uploaded files and locally persisted conversations may contain sensitive data.
  Protect the project directory and back up or delete runtime data according to
  your requirements.

## License

No license file is currently included in this repository.
