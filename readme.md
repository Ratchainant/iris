# IRIS — Skill Gap Intelligence System

> **Codename:** IRIS
> **Domain:** Text Analytics · Agentic AI
> **Status:** Planning

---

## Overview

IRIS is an agentic AI system designed to quantify and explain the gap between the skills and knowledge delivered by an academic programme and those demanded by the current job market. By comparing two independent evidence sources — academic curriculum data and real-world job postings — IRIS produces actionable insights that help curriculum designers, faculty, and students understand where the programme needs to evolve.

---

## Problem Statement

Academic programmes are updated on multi-year cycles, while the job market evolves continuously. Without a systematic, data-driven comparison, programme committees rely on anecdotal evidence or infrequent industry surveys to decide which competencies to add, remove, or strengthen. IRIS automates this comparison at scale and makes the evidence reproducible.

---

## Data Sources

### Source 1 — Academic Programme

| Item | Description |
|------|-------------|
| Input | Programme specification documents, course syllabi, learning outcome statements |
| Granularity | Per academic year (e.g., Year 1 – Year 4) |
| Output | Skill & knowledge distribution per academic year |

The AI parses structured and unstructured programme documents to extract a weighted distribution of skills and knowledge areas gained by students who complete each year of the programme.

### Source 2 — Job Market Postings

| Item | Description |
|------|-------------|
| Input | Job postings scraped from selected target websites |
| Granularity | CS career path × Business sector × Timeline (default: yearly) |
| Output | Required skill & knowledge distribution per segment |

The AI collects and analyzes job descriptions to produce demand distributions broken down by:
- **CS Career Path** — e.g., Software Engineering, Data Science, Cybersecurity, AI/ML, DevOps
- **Business Sector** — e.g., Finance, Healthcare, Retail, Government, Tech
- **Timeline** — yearly aggregation by default; configurable to quarterly or monthly

---

## Core Capabilities

### 1. Curriculum Analysis Agent
- Ingests programme documents (PDF, DOCX, HTML, structured YAML/JSON)
- Extracts competency statements and maps them to a standardized skill taxonomy
- Produces a skill–knowledge distribution vector per academic year

### 2. Job Market Harvesting Agent
- Crawls and scrapes job postings from configured target websites
- Normalizes job descriptions using NLP (entity extraction, keyword clustering, embedding-based classification)
- Aggregates postings into demand distribution vectors segmented by career path, sector, and time period

### 3. Gap Analysis Engine
- Estimates goodness-of-fit between the academic supply distribution and the job market demand distribution
- Applies statistical methods (e.g., KL divergence, cosine similarity, chi-square test) to quantify alignment
- Generates ranked gap summaries: skills over-supplied, under-supplied, and absent from the curriculum

### 4. Reporting & Recommendation Agent
- Synthesizes gap analysis results into human-readable reports
- Provides prioritized curriculum recommendations
- Supports drill-down by career path, sector, or year of study
- Produces trend analysis comparing gap evolution across time periods

---

## System Architecture (High Level)

```
┌─────────────────────────────────────────────────────────┐
│                        IRIS Core                        │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────────────┐ │
│  │ Curriculum Agent │      │  Job Market Agent        │ │
│  │ ─────────────── │      │  ──────────────────────  │ │
│  │ • Document parse │      │  • Web scraping/crawling │ │
│  │ • Skill extract  │      │  • NLP extraction        │ │
│  │ • Distribution   │      │  • Segmented aggregation │ │
│  └────────┬─────────┘      └───────────┬──────────────┘ │
│           │                            │                 │
│           └──────────┬─────────────────┘                 │
│                      ▼                                   │
│           ┌──────────────────────┐                       │
│           │   Gap Analysis       │                       │
│           │   Engine             │                       │
│           │  • Goodness-of-fit   │                       │
│           │  • Divergence score  │                       │
│           │  • Ranked gap list   │                       │
│           └──────────┬───────────┘                       │
│                      ▼                                   │
│           ┌──────────────────────┐                       │
│           │  Reporting Agent     │                       │
│           │  • Narrative report  │                       │
│           │  • Recommendations   │                       │
│           │  • Trend dashboard   │                       │
│           └──────────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

---

## Skill Taxonomy

IRIS uses a two-level taxonomy to ensure consistency across both sources:

- **Level 1 — Domain** (e.g., Programming, Data Management, Systems, Security, Communication)
- **Level 2 — Competency** (e.g., Python, SQL, REST API Design, Statistical Modeling)

The taxonomy is configurable and can be extended or mapped to external frameworks (e.g., SFIA, ESCO, O*NET).

---

## Key Metrics

| Metric | Description |
|--------|-------------|
| Supply–Demand Alignment Score | Overall goodness-of-fit between programme output and job market demand |
| Top-N Skill Gaps | Skills most demanded by the market but least covered by the curriculum |
| Top-N Surpluses | Skills well-covered by the curriculum but rarely demanded |
| Trend Delta | Change in gap scores across consecutive time periods |
| Coverage Rate | Percentage of demanded competencies present in the curriculum |

---

## Target Users

- **Curriculum Committees** — Evidence-based input for programme review cycles
- **Faculty & Course Designers** — Understand which skills to emphasize or de-emphasize
- **Students & Advisors** — Identify supplementary learning priorities per career path
- **Institutional Researchers** — Longitudinal tracking of programme relevance

---

## Constraints & Assumptions

- Academic programme documents are available in digital format (PDF, DOCX, or structured data)
- Job postings are publicly accessible and scraping is permitted under applicable terms of service
- Skill taxonomy is defined and maintained separately; initial version is provided by domain experts
- Timeline aggregation defaults to yearly; finer granularity requires sufficient data volume
- Analysis is descriptive and advisory; final curriculum decisions remain with human committees

---

## Out of Scope (Initial Release)

- Automated curriculum modification or scheduling
- Real-time job posting monitoring (batch processing only)
- Assessment of teaching quality or pedagogy
- Integration with student grade or transcript data

---

## Roadmap

| Phase | Milestone |
|-------|-----------|
| Phase 1 | Define skill taxonomy; build curriculum ingestion pipeline |
| Phase 2 | Implement job market harvesting agent and NLP extraction |
| Phase 3 | Build gap analysis engine with statistical scoring |
| Phase 4 | Reporting agent and visualization dashboard |
| Phase 5 | Multi-programme and longitudinal comparison support |

---

## Repository Structure (Planned)

```
iris/
├── readme.md                  # This file
├── tech_stack.md              # Technology decisions
├── implementation_plan.md     # Detailed development plan
├── backend/                   # API and core processing services
├── agents/                    # Agentic AI components
│   ├── curriculum_agent/
│   ├── job_market_agent/
│   ├── gap_analysis_engine/
│   └── reporting_agent/
├── data/                      # Raw and processed data
│   ├── programme/
│   └── job_postings/
├── taxonomy/                  # Skill taxonomy definitions
├── frontend/                  # Dashboard and reporting UI
└── scripts/                   # Utility and automation scripts
```

---

*IRIS — Illuminating the gap between academic preparation and industry expectation.*
