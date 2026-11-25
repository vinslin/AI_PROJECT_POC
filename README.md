# AB Consultants — AI-Powered RFQ & Drawing Analysis · Proposal Workflow

A proof-of-concept pipeline that ingests a civil engineering RFQ package (DOCX + PDF drawings), runs AI analysis across multiple stages, compares the project against historical portfolio data, and assembles a draft proposal — with human gate decisions at key checkpoints.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Environment Setup](#environment-setup)
5. [Database Setup](#database-setup)
6. [Backend Setup](#backend-setup)
7. [Frontend Setup](#frontend-setup)
8. [Running the Application](#running-the-application)
9. [Pipeline Steps](#pipeline-steps)
10. [Database Tables](#database-tables)

---

## Architecture Overview

```
Frontend (React + Vite)          Backend (FastAPI + Python)
  localhost:5173            ←→     localhost:8001
  
  - Horizontal timeline UI          - SSE streaming pipeline
  - Clickable step panels           - OpenAI GPT-4o (text + vision)
  - Gate approve / decline          - PostgreSQL via SQLAlchemy
  - Live progress via SSE           - DOCX export via python-docx
```

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Python | 3.10 + | 3.12 recommended |
| Node.js | 18 + | For React frontend |
| PostgreSQL | 14 + | Local or remote instance |
| OpenAI API Key | — | GPT-4o access required |

---

## Project Structure

```
AI_PROJECT_POC/
├── backend/
│   ├── main.py               # FastAPI app + pipeline runner
│   ├── pipeline.py           # All AI pipeline step functions
│   ├── models.py             # SQLAlchemy ORM models
│   ├── db.py                 # DB connection + session factory
│   ├── seed_past_projects.py # Seeds 10 mock past projects
│   ├── documents/            # Drop RFQ (.docx) + Drawing (.pdf) here
│   └── exports/              # Generated DOCX proposals saved here
├── frontend/
│   ├── src/
│   │   ├── pages/PipelinePage.jsx   # Main pipeline UI
│   │   └── components/              # Header, ResultCard, etc.
│   └── package.json
├── example.env               # Template — copy to .env
├── requirements.txt          # Python dependencies
└── README.md
```

---

## Environment Setup

1. Copy the example env file and fill in your values:

```bash
cp example.env .env
```

2. Edit `.env`:

```env
# OpenAI
OPENAI_API_KEY=sk-proj-your-key-here
OPENAI_TEXT_MODEL=gpt-6-luna
OPENAI_VISION_MODEL=gpt-6-luna

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=rfq_poc
DB_USER=postgres
DB_PASSWORD=your-db-password
```

> **Never commit `.env`** — it is listed in `.gitignore`.

---

## Database Setup

### 1. Create the PostgreSQL database

Connect to PostgreSQL and run:

```sql
CREATE DATABASE rfq_poc;
```

If your user is not `postgres`, also run:

```sql
CREATE USER your_user WITH PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE rfq_poc TO your_user;
```

### 2. Create all tables

Tables are created automatically on first backend startup via SQLAlchemy's `create_all`. No manual migration needed.

The following tables are created:

| Table | Purpose |
|-------|---------|
| `analysis_runs` | Tracks each pipeline execution |
| `rfq_requirements` | AI-extracted RFQ fields |
| `drawing_register` | AI-extracted drawing sheet data |
| `discrepancies` | Cross-document gaps / RFI register |
| `proposal_sections` | AI-generated proposal draft sections |
| `approval_log` | Audit trail of all gate decisions |
| `past_projects` | Historical project portfolio for bid viability |

### 3. Seed past projects (required for Bid Viability step)

```bash
cd backend
python seed_past_projects.py
```

This inserts 10 mock historical projects. It is safe to re-run — skips if data already exists.

---

## Backend Setup

```bash
# From the project root
cd backend

# Create and activate virtual environment
python -m venv venv

# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

---

## Frontend Setup

```bash
cd frontend
npm install
```

---

## Running the Application

### Start the backend

```bash
cd backend
venv\Scripts\activate      # Windows
source venv/bin/activate   # macOS / Linux

uvicorn main:app --port 8001 --reload
```

Backend runs at: `http://localhost:8001`

### Start the frontend

```bash
cd frontend
npm run dev
```

Frontend runs at: `http://localhost:5173`

### Add documents

Place your documents in `backend/documents/`:

- One `.docx` file — the RFQ package
- One `.pdf` file — the engineering drawings

The app auto-detects them by file extension on startup.

---

## Pipeline Steps

| # | Step | Type | Description |
|---|------|------|-------------|
| 1 | Intake Acceptance | Auto | Captures file manifest and metadata |
| 2 | Package Quality Gate | Auto | Checks file readability and page counts |
| 3 | RFQ Document Analysis | AI | Extracts requirements, scope, deadlines via GPT-4o |
| 4 | Drawing Register | AI Vision | Classifies sheets and extracts labels via GPT-4o Vision |
| 5 | Cross-Document Analysis | AI | Flags gaps between RFQ and drawings, generates RFI list |
| 6 | Bid Viability Assessment | AI + Portfolio | Compares project against 10 historical projects; produces Go/No-Go recommendation |
| 7 | Bid Go / No-Go Decision | Human Gate | Manager approves to proceed or declines the bid |
| 8 | Engineering Validation | Human Gate | Discipline lead validates drawing findings |
| 9 | Proposal Assembly | AI + Template | Generates 8-section proposal draft |
| 10 | Commercial Review | Human Gate | Proposal manager reviews coverage and fees |
| 11 | Final QA & Release | Human Gate | Authorised AB approver checks mandatory sections |
| 12 | Export & Archive | Auto | Generates approved DOCX + full audit trail |

---

## Database Tables

### `analysis_runs`
Tracks each pipeline run.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | Auto-increment |
| status | String | started / completed |
| current_step | String | Last active step ID |
| created_at | DateTime | — |
| updated_at | DateTime | Auto-updated |

### `rfq_requirements`
AI-extracted fields from the RFQ document.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | — |
| run_id | Integer | FK → analysis_runs.id |
| field | String | e.g. project_name, disciplines |
| value | Text | Extracted value |
| source_file | String | Filename |

### `drawing_register`
Sheet-by-sheet data extracted from engineering drawings.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | — |
| run_id | Integer | FK → analysis_runs.id |
| sheet_id | String | e.g. NCMC-RD-001 |
| title | Text | Sheet title |
| discipline | String | Civil / Structural / etc. |
| revision | String | e.g. P1, A |
| scale | String | e.g. 1:200, NTS |
| source_file | String | PDF filename |
| findings | JSON | Observations from AI |

### `discrepancies`
Cross-document gaps and generated RFI items.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | — |
| run_id | Integer | FK → analysis_runs.id |
| rfi_id | String | e.g. RFI-01 |
| issue | Text | Description of the gap |
| evidence | Text | Source reference |
| priority | String | High / Medium / Low |
| suggested_action | Text | Recommended resolution |

### `proposal_sections`
AI-generated proposal draft content.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | — |
| run_id | Integer | FK → analysis_runs.id |
| section_number | Integer | 1–8 |
| section_title | String | e.g. Executive Summary |
| content | Text | AI-drafted body text |

### `approval_log`
Full audit trail of every gate decision.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | — |
| run_id | Integer | FK → analysis_runs.id |
| gate | String | G0 / G1 / G_BID / G2 / G3 / G4 |
| action | String | approve / return |
| role | String | Reviewer role |
| comment | Text | Optional reason |
| timestamp | DateTime | Decision time |
| version | Integer | Revision number |

### `past_projects`
Historical project portfolio used for Bid Viability comparison.

| Column | Type | Notes |
|--------|------|-------|
| id | Integer PK | — |
| project_name | String | Full project name |
| year | Integer | Tender year |
| client | String | Client organisation |
| project_type | String | road_rehabilitation / stormwater / bridge / building |
| disciplines | JSON | List of discipline strings |
| corridor_length_km | String | e.g. "2.1" or "N/A" |
| drawing_completeness | String | complete / partial / illustrative |
| rfi_count_at_tender | Integer | Number of RFIs raised at tender |
| outcome | String | won / lost / declined |
| profit_margin_pct | String | e.g. "18" or "N/A" |
| complexity | String | low / medium / high |
| active_road | String | yes / no |
| notes | Text | Free-text notes |

> Seed data: run `python backend/seed_past_projects.py` to populate 10 mock records.
