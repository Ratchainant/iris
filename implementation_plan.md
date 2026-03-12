# IRIS — Implementation Plan

---

## Principles

- Build incrementally: each phase produces a working, testable system
- Agents are developed and tested independently before integration
- Local storage and Docker setup are established early so all later phases build on a stable foundation
- The skill taxonomy is the shared contract between all agents — define it first

---

## Phase 0 — Project Foundation

**Goal:** Establish the full project scaffold, Docker environment, and shared conventions so every subsequent phase has a stable base to build on.

### Tasks

#### 0.1 Repository & Project Structure
- [ ] Create all top-level directories: `frontend/`, `backend/`, `ai_worker/`, `nginx/`, `taxonomy/`, `config/`, `scripts/`
- [x] Add `.gitignore` (ignores data content, `__pycache__`, `.env`, `node_modules`, etc.)
- [x] Add `.env.example` with all required environment variables documented
- [x] Create `data/` subdirectory structure with `.gitkeep` placeholders (`job_postings/`, `programmes/`, `results/`)

#### 0.2 Docker Compose Setup
- [ ] Write `docker-compose.yml` with services: `frontend`, `backend`, `ai_worker`, `redis`, `nginx`
- [ ] Configure host volume mounts: `./data:/app/data` for `backend` and `ai_worker`
- [ ] Add GPU passthrough config (optional device reservation for CUDA)
- [ ] Write `Dockerfile` for each service
- [ ] Verify all services start with `docker compose up`

#### 0.3 Nginx Reverse Proxy
- [ ] Configure Nginx to route `/` → frontend (`:3000`) and `/api` → backend (`:8000`)
- [ ] Write `nginx/nginx.conf`

#### 0.4 Skill Taxonomy (v1)
- [ ] Define two-level taxonomy in `taxonomy/taxonomy.json`:
  - Level 1 (Domain): Programming, Data Management, Systems & Infrastructure, Security, AI & Machine Learning, Software Engineering, Communication & Collaboration, Business & Domain Knowledge
  - Level 2 (Competency): 8–15 competencies per domain
- [ ] Write a taxonomy validation script (`scripts/validate_taxonomy.py`)

#### 0.5 Shared Data Schemas
- [ ] Define Pydantic schemas (reused across backend and AI worker):
  - `ProgrammeDocument`, `CourseCompetency`, `SkillDistribution`
  - `JobPosting`, `JobMarketDistribution`
  - `GapAnalysisResult`, `GapReport`
- [ ] Store schemas in `backend/app/schemas/`

**Deliverable:** All services start successfully with `docker compose up`; taxonomy v1 is committed; shared schemas are defined.

---

## Phase 1 — Academic Programme Ingestion

**Goal:** Allow users to upload programme documents via the web UI and have the Curriculum Analysis Agent extract a skill distribution from them.

### Tasks

#### 1.1 Backend — File Upload API
- [ ] `POST /api/programmes/{programme_name}/documents` — upload PDF or DOCX, validate MIME type and extension
- [ ] Save files to `data/programmes/<programme_name>/<year>/` on host storage
- [ ] `GET /api/programmes` — list all programmes and their uploaded documents
- [ ] `DELETE /api/programmes/{programme_name}/documents/{filename}` — remove a document

#### 1.2 Frontend — Programme Document Manager
- [ ] Programme list page: show all programmes as cards
- [ ] Programme detail page: folder tree by year; list of uploaded documents per year
- [ ] Upload form: drag-and-drop or file picker; select programme name and academic year
- [ ] Delete document with confirmation dialog

#### 1.3 Curriculum Analysis Agent
- [ ] **Document Parser**: extract raw text from PDF (PyMuPDF) and DOCX (python-docx)
- [ ] **Competency Extractor**: use Gemma 3 (via Ollama) to identify skill and knowledge statements from programme text; map them to taxonomy Level 2 competencies
- [ ] **Distribution Builder**: compute a normalized frequency/weight vector over taxonomy competencies for each academic year
- [ ] Save extracted distribution to `data/programmes/<programme_name>/distributions/<year>.json`
- [ ] Celery task: `tasks.analyse_curriculum(programme_name, year)`

#### 1.4 Backend — Analysis Trigger API
- [ ] `POST /api/programmes/{programme_name}/analyse` — enqueue Celery task
- [ ] `GET /api/programmes/{programme_name}/analysis-status` — return Celery task status
- [ ] `GET /api/programmes/{programme_name}/distributions` — return computed distributions per year

#### 1.5 Frontend — Analysis Controls
- [ ] "Analyse Programme" button per programme
- [ ] Progress indicator while task is running
- [ ] Display extracted competencies and distribution per year (bar chart)

**Deliverable:** User can upload documents, trigger analysis, and see the extracted skill distribution per academic year.

---

## Phase 2 — Job Market Harvesting

**Goal:** Scrape job postings from configured target websites and build demand distributions segmented by career path, business sector, and year.

### Tasks

#### 2.1 Scraper Configuration
- [ ] Define scraper target config in `config/scraper_config.json` (git-tracked; no credentials):
  - Target sites, URL patterns, CSS/XPath selectors, rate limits, enabled flag
- [ ] `GET /api/scraper/config` and `PUT /api/scraper/config` — read and update config via API

#### 2.2 Job Market Harvesting Agent
- [ ] **Web Scraper**: httpx + BeautifulSoup4 (or Scrapy spider per site); respect `robots.txt` and rate limits
- [ ] **Job Normalizer**: use Gemma 3 to extract structured fields from raw job text:
  - title, company, business sector, CS career path, required skills, posted date
- [ ] **Taxonomy Mapper**: map extracted skills to taxonomy Level 2 competencies
- [ ] **Aggregator**: group normalized postings by `(career_path, sector, year)`; compute demand distribution vectors
- [ ] Save raw postings to `data/job_postings/<source>/<year>/YYYY-MM-DD.json`
- [ ] Save aggregated distributions to `data/job_postings/distributions/<year>/<career_path>_<sector>.json`
- [ ] Celery task: `tasks.harvest_jobs(source, date_range)`

#### 2.3 Backend — Scraping API
- [ ] `POST /api/jobs/scrape` — enqueue scraping task with source and date range
- [ ] `GET /api/jobs/scrape-status` — return task status and progress
- [ ] `GET /api/jobs/distributions` — return available demand distributions (filterable by year, career path, sector)
- [ ] `GET /api/jobs/postings` — paginated list of raw postings with filters

#### 2.4 Frontend — Job Market Controls
- [ ] Scraper settings page: view and edit scraper config
- [ ] "Run Scraping" button with date range picker
- [ ] Scraping progress log
- [ ] Job postings browser: filter by career path, sector, year; show count and sample postings
- [ ] Demand distribution chart per segment

**Deliverable:** System can scrape job postings, normalize and classify them, and produce demand distributions viewable in the UI.

---

## Phase 3 — Gap Analysis Engine

**Goal:** Compare the academic supply distribution against the job market demand distribution and produce quantified gap scores.

### Tasks

#### 3.1 Gap Analysis Engine — Core
- [ ] **Distribution Aligner**: ensure both distributions are over the same taxonomy vector space; handle missing competencies (zero-fill)
- [ ] **Goodness-of-Fit Scoring**: implement multiple metrics:
  - Cosine similarity (overall alignment)
  - KL divergence (supply vs demand asymmetry)
  - Chi-square test (statistical significance of differences)
  - Per-competency delta (demand weight − supply weight)
- [ ] **Gap Ranker**: rank competencies by magnitude of under-supply and over-supply
- [ ] **Coverage Rate**: percentage of demanded competencies present in curriculum
- [ ] Save results to `data/results/<programme_name>/<timestamp>/gap_<year>_<career_path>_<sector>.json`
- [ ] Celery task: `tasks.run_gap_analysis(programme_name, career_path, sector, year)`

#### 3.2 Backend — Gap Analysis API
- [ ] `POST /api/analysis/run` — enqueue gap analysis task with parameters
- [ ] `GET /api/analysis/status/{task_id}` — task status
- [ ] `GET /api/analysis/results` — list all completed analyses
- [ ] `GET /api/analysis/results/{result_id}` — full result detail

#### 3.3 Frontend — Gap Analysis Runner
- [ ] Analysis configuration form: select programme, career path(s), sector(s), year(s)
- [ ] Run analysis button with task progress display
- [ ] Results list with summary scores

**Deliverable:** Gap analysis runs end-to-end and produces a JSON result file with ranked gaps and alignment scores.

---

## Phase 4 — Reporting & Visualization

**Goal:** Surface gap analysis results through clear visualizations and AI-generated narrative reports with actionable recommendations.

### Tasks

#### 4.1 Reporting Agent
- [ ] **Narrative Generator**: use Gemma 3 to write a human-readable summary of the gap analysis result:
  - Executive summary (2–3 sentences)
  - Key gaps section (top-5 under-supplied skills with context)
  - Surplus section (top-3 over-supplied skills)
  - Trend section (if multiple years available)
  - Prioritized recommendations (ordered list of curriculum actions)
- [ ] Save report text alongside the gap result JSON
- [ ] Celery task: `tasks.generate_report(result_id)`

#### 4.2 Backend — Report API
- [ ] `POST /api/reports/generate/{result_id}` — enqueue report generation
- [ ] `GET /api/reports/{result_id}` — return narrative report text and metadata

#### 4.3 Frontend — Dashboard & Reports
- [ ] **Gap Dashboard**:
  - Alignment score gauge
  - Top-N gaps: horizontal bar chart (demand vs supply side-by-side)
  - Top-N surpluses: horizontal bar chart
  - Competency heatmap: academic years × taxonomy competencies
- [ ] **Report Viewer**:
  - Rendered narrative report (Markdown)
  - Expandable sections per finding
  - Export to PDF button
- [ ] **Comparison View**: overlay multiple career paths or sectors on the same chart

#### 4.4 Trend Analysis
- [ ] Trend chart: alignment score over years for a given programme + segment
- [ ] Trend delta table: which gaps are growing vs shrinking year-over-year

**Deliverable:** Full end-to-end flow works — upload documents, scrape jobs, run analysis, view dashboard, read AI-generated report.

---

## Phase 5 — Multi-Programme & Longitudinal Comparison

**Goal:** Support comparison across multiple programmes simultaneously and track gap evolution over time.

### Tasks

#### 5.1 Multi-Programme Comparison
- [ ] Side-by-side alignment score comparison across programmes
- [ ] Shared gap heatmap: programmes × competencies
- [ ] API: `GET /api/analysis/compare?programmes=A,B&career_path=X&year=2025`

#### 5.2 Longitudinal Tracking
- [ ] Scheduled scraping: cron-triggered Celery beat task for periodic job market refresh
- [ ] Automated re-analysis when new scraping data is available
- [ ] Trend line charts spanning multiple years per programme

#### 5.3 Export & Sharing
- [ ] Export full gap report as PDF
- [ ] Export distributions and gap scores as CSV/Excel
- [ ] API: `GET /api/reports/{result_id}/export?format=pdf|csv`

#### 5.4 Taxonomy Management UI
- [ ] View and edit taxonomy domains and competencies via the UI
- [ ] Taxonomy version history

**Deliverable:** System supports institutional-scale usage with multiple programmes, automated periodic refresh, and exportable reports.

---

## Cross-Cutting Concerns

### Testing Strategy
| Layer | Approach |
|-------|----------|
| Taxonomy | Unit tests: validate structure, no duplicate competencies |
| Document Parser | Unit tests with sample PDF/DOCX fixtures |
| Competency Extractor | Snapshot tests: known input → expected taxonomy mapping |
| Scrapers | Integration tests against mock HTML fixtures |
| Gap Analysis Engine | Unit tests: known distributions → expected scores |
| API Endpoints | Integration tests with pytest + httpx |
| Frontend | Component tests with Vitest + Testing Library |

### Error Handling
- File upload: reject unsupported formats with clear error message
- Scraping: log failures per URL; continue batch; report failed URLs in task result
- AI extraction: if Gemma 3 is unavailable, return a clear error (do not silently produce empty results)
- Celery tasks: all tasks have retry limits and failure state reporting

### Logging
- Structured JSON logs from backend and AI worker (stdout → Docker log driver)
- Task start, completion, and failure events logged with task ID, duration, and parameters

---

## Milestone Summary

| Phase | Milestone | Key Deliverable |
|-------|-----------|-----------------|
| 0 | Foundation | Docker stack runs; taxonomy v1 defined |
| 1 | Curriculum Ingestion | Upload docs → view skill distribution |
| 2 | Job Market Harvesting | Scrape jobs → view demand distribution |
| 3 | Gap Analysis | Run analysis → view gap scores |
| 4 | Reporting | View dashboard + AI-generated report |
| 5 | Scale & Export | Multi-programme, trends, PDF export |

---

## Development Order Within Each Phase

For each phase, follow this order:
1. Define/update shared schemas
2. Implement backend API (with stub responses if AI not ready)
3. Implement agent/worker logic
4. Wire agent to backend via Celery task
5. Build frontend UI against live API
6. Write tests
