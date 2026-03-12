# IRIS — Technology Stack

---

## Deployment Model

IRIS runs as a **web application in Docker containers** orchestrated with Docker Compose. All services (frontend, backend, AI worker) are containerized. Data (job postings, uploaded programme documents) is persisted in **local host storage** mounted into the containers via Docker volumes — nothing is stored inside the containers themselves.

---

## Architecture Overview

```
Host Machine
├── docker-compose.yml
├── data/                          ← mounted into containers (NOT inside Docker)
│   ├── job_postings/              ← scraped job data (JSON)
│   │   └── <source>/<year>/
│   └── programmes/                ← uploaded academic documents
│       └── <programme_name>/
│           ├── year1/
│           ├── year2/
│           └── ...
│
└── Docker Network (iris_net)
    ├── frontend    (Next.js)       :3000
    ├── backend     (FastAPI)       :8000
    └── ai_worker   (Python)        ← background task runner
```

---

## Services

### Frontend — Next.js
| Item | Choice |
|------|--------|
| Framework | Next.js 14+ (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS |
| UI Components | shadcn/ui |
| State Management | Zustand |
| HTTP Client | Axios / fetch |

**Key UI features:**
- Programme document manager — upload documents (PDF, DOCX), browse programmes as folder tree, list documents per programme per year
- Job market scraping controls — configure target sites, trigger scraping jobs, monitor progress
- Gap analysis dashboard — alignment scores, skill gap charts, trend visualizations
- Report viewer — narrative summaries and recommendations

---

### Backend — FastAPI
| Item | Choice |
|------|--------|
| Framework | FastAPI |
| Language | Python 3.11+ |
| Task Queue | Celery + Redis |
| File Handling | python-multipart (upload), aiofiles |
| Document Parsing | PyMuPDF (PDF), python-docx (DOCX) |
| Web Scraping | Scrapy / httpx + BeautifulSoup4 |
| Data Validation | Pydantic v2 |
| API Docs | Auto-generated via FastAPI (Swagger UI) |

**Responsibilities:**
- REST API for frontend
- Programme document upload and file system management
- Trigger and monitor scraping jobs
- Trigger and monitor AI analysis tasks
- Serve gap analysis results and reports

---

### AI Worker — Python
| Item | Choice |
|------|--------|
| LLM | Gemma 3 (local, via Ollama or llama.cpp) |
| GPU Acceleration | CUDA (NVIDIA) → MPS (Apple Silicon) → CPU fallback |
| Agent Framework | LangChain / LangGraph |
| Embeddings | sentence-transformers (local) |
| NLP Utilities | spaCy, NLTK |
| Statistical Analysis | NumPy, SciPy, scikit-learn |
| Data Processing | Pandas |

**GPU Priority Logic:**
```python
import torch

if torch.cuda.is_available():
    device = "cuda"
elif torch.backends.mps.is_available():
    device = "mps"
else:
    device = "cpu"
```

---

### Infrastructure
| Item | Choice |
|------|--------|
| Containerization | Docker |
| Orchestration | Docker Compose |
| Reverse Proxy | Nginx (routes `/` → frontend, `/api` → backend) |
| Message Broker | Redis (Celery broker + result backend) |
| Local Storage | Host-mounted Docker volumes |

---

## Data Storage

IRIS uses **local file system storage** on the host machine only — no database server is required in the initial release.

### Job Postings
- Format: **JSON** files
- Location: `./data/job_postings/<source_name>/<year>/YYYY-MM-DD.json`
- Each file contains an array of normalized job posting objects

```json
{
  "scraped_at": "2026-03-12T10:00:00Z",
  "source": "example-jobs-site",
  "postings": [
    {
      "id": "abc123",
      "title": "Data Engineer",
      "company": "Acme Corp",
      "sector": "Finance",
      "career_path": "Data Engineering",
      "skills": ["Python", "SQL", "Spark"],
      "posted_date": "2026-03-10"
    }
  ]
}
```

### Academic Programme Documents
- Format: **PDF, DOCX** (as uploaded by user)
- Location: `./data/programmes/<programme_name>/<year>/`
- Parsed text and extracted competencies cached as JSON alongside source files

### Analysis Results
- Format: **JSON**
- Location: `./data/results/<programme_name>/<run_timestamp>/`

---

## Docker Compose Structure

```yaml
services:
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    depends_on: [backend]

  backend:
    build: ./backend
    ports: ["8000:8000"]
    volumes:
      - ./data:/app/data          # host storage mounted
    depends_on: [redis]
    environment:
      - DATA_DIR=/app/data

  ai_worker:
    build: ./ai_worker
    volumes:
      - ./data:/app/data          # same host storage
    depends_on: [redis, backend]
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]  # enable GPU passthrough when available

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    depends_on: [frontend, backend]
```

---

## AI Model

| Item | Detail |
|------|--------|
| Model | Gemma 3 (Google) |
| Serving | Ollama (recommended) or llama.cpp |
| Quantization | Q4_K_M for CPU/MPS; full precision or Q8 for CUDA |
| Context Window | 8k tokens (default); configurable per task |
| Hardware Priority | CUDA GPU → Apple MPS → CPU |

Ollama runs on the **host machine** (not in Docker) and is accessed by the AI worker container via `host.docker.internal` or a configured host IP. This avoids GPU passthrough complexity while still keeping services containerized.

---

## Development Environment

| Tool | Purpose |
|------|---------|
| VS Code | Primary IDE |
| Docker Desktop | Container management |
| Ollama | Local LLM serving |
| Python 3.11+ | Backend and AI worker |
| Node.js 20+ | Frontend |
| Git + GitHub | Version control |

---

## Security Considerations

- Uploaded documents are stored only on the local host file system; no cloud upload
- Backend API is not exposed publicly; Nginx acts as the sole entry point
- File upload validation: MIME type and extension checks (PDF, DOCX only)
- Scraping respects `robots.txt` and rate limits per target site configuration
- No user authentication required in the initial release (single-user local deployment)

---

## Technology Decision Log

| Decision | Rationale |
|----------|-----------|
| Gemma 3 (local) | Privacy-preserving; no API cost; runs on-device with GPU acceleration |
| Ollama for model serving | Simplest way to run Gemma 3 locally; avoids GPU-in-Docker complexity |
| FastAPI | Async-native, fast, excellent Pydantic integration, auto API docs |
| Next.js App Router | Modern React with server components; good for document management UI |
| Local JSON storage | Simplicity; no DB setup; easily inspectable and portable |
| Celery + Redis | Decouples long-running scraping/AI tasks from HTTP request cycle |
| Docker Compose | Single-command startup; reproducible across machines |
