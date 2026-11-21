# AI-Assisted Clinical EMR & Diagnostic Management System

A cloud-native Electronic Medical Record (EMR) and pathology intelligence system designed for clinical diagnostic centers. The platform ingests multimodal lab reports (native digital PDFs, flatbed scanner TIFFs, and smartphone captures), extracts structured biomarkers using a two-tier vision LLM pipeline, normalizes test results against demographic physiological baselines, and renders longitudinal patient trajectories.

---

## System Overview

Manual transcription of laboratory reports into medical records is error-prone, labor-intensive, and lacks longitudinal context. This system eliminates manual data entry by combining computer vision, Large Language Model spatial reasoning, and clinical reference databases into a streamlined clinical decision support workflow.

```
[Diagnostic Slip: PDF / Scan / Image]
                  │
                  ▼
   ┌─────────────────────────────┐
   │    FastAPI Gateway (ASGI)   │
   │  - Stateless JWT Auth Guard │
   │  - Brute-Force Rate Limiter │
   │  - Pydantic v2 Validations  │
   └──────────────┬──────────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
  [Cloudflare R2]    [Two-Tier AI Engine]
  S3-Compatible      - Tier 1: Claude 3.5 Haiku (Fast Text, < 2.0s)
  Raw Document Blob  - Tier 2: Claude 3.5 Sonnet (200 DPI Vision)
                            │
                            ▼
              [Clinical Normalization Engine]
              - Age & Sex-Calibrated Envelopes
              - Critical Alert Threshold Checks
              - Multi-Tier Status Classification
                            │
                            ▼
               [Relational Database: DBWrapper]
               - PostgreSQL (Supabase Production)
               - SQLite (Local Dev with WAL Mode)
               - Immutable Audit Trail Logging
```

---

## Core Capabilities

- **Multimodal Document Ingestion:** Processes native vector PDFs, flatbed scanner slips, and smartphone photographs (`.pdf`, `.png`, `.jpg`, `.jpeg`, `.webp`, `.tiff`).
- **Two-Tier AI Extraction:**
  - *Fast Path:* In-memory text extraction via `pdfplumber` routed to Claude 3.5 Haiku with sub-2.0s token resolution.
  - *Vision Escalation:* Scanned artifacts and stacked layouts rendered at 200 DPI via `pypdfium2` and resolved using Claude 3.5 Sonnet Vision.
- **Stacked-Cell Disambiguation:** Spatial prompt grounding correctly splits adjacent or vertically stacked multi-test table cells without cross-mixing results.
- **Demographic Reference Normalization:** Validates extracted values against sex- and age-specific physiological boundaries (Normal, High, Low, Critical Alert, Needs Review).
- **Human-in-the-Loop Verification:** Interactive pre-commit review table allows clinicians to inspect and correct AI-parsed parameters prior to persistence.
- **Longitudinal Trend Visualizations:** Interactive SVG trajectory charts (Recharts) displaying patient biomarker progression over multiple visits with shaded normal envelopes.
- **Verified Document Provenance:** Pairs generated A4 discharge/summary documentation with direct, immutable links to original raw scanner files stored in object storage.
- **Role-Based Access Control (RBAC):** Strict operational segregation between Administrator and Clinician roles.
- **Real-Time Duplicate Prevention:** Real-time database indexing on laboratory receipt accession IDs prevents accidental duplicate uploads.

---

## Technology Stack & Libraries

### 1. Frontend Client Tier
- **Framework:** React 18 (Single Page Application)
- **Tooling & Bundler:** React Scripts / Webpack with ES6+ modular architecture
- **State Management:** React Context API with persistent browser storage synchronization
- **Data Visualization:** Recharts (SVG-based reactive time-series charts and flowsheet matrices)
- **Styling:** Modular CSS3 with clinical design tokens and responsive mobile navigation

### 2. Backend Application Tier
- **Runtime:** Python 3.12
- **Web Framework:** FastAPI (Asynchronous ASGI REST architecture)
- **Server:** Uvicorn (High-performance ASGI server)
- **Data Validation:** Pydantic v2 (Strict request/response contracts and phone/age regex validators)
- **Document Processing:**
  - `pdfplumber`: Deterministic text extraction from vector PDFs
  - `pypdfium2`: High-speed PDF rendering to image buffers at 200 DPI
  - `Pillow` (PIL): Image manipulation, resizing, and format conversion

### 3. AI & Language Models
- **SDK:** Anthropic Python SDK (`anthropic`)
- **Primary Fast Model:** Claude 3.5 Haiku (`claude-haiku-4-5`)
- **Vision Escalation Model:** Claude 3.5 Sonnet (`claude-sonnet-4-5`)

### 4. Security & Cryptography
- **Token Management:** `PyJWT` (Stateless HMAC-SHA256 signed Bearer tokens)
- **Password Security:** `bcrypt` (12 salt rounds with automatic legacy hash migration)
- **Rate Limiting:** Custom thread-safe sliding-window rate limiter protecting authentication endpoints
- **Governance:** Immutable relational audit log capturing timestamps, IP addresses, and operational actions

### 5. Persistence & Cloud Infrastructure
- **Database Engine:** Dual-engine architecture via custom `DBWrapper`:
  - *Production:* PostgreSQL on Supabase (`psycopg2-binary`)
  - *Development:* SQLite 3 with Write-Ahead Logging (`WAL` mode)
- **Object Storage:** Cloudflare R2 via AWS S3-compatible SDK (`boto3`) with Supabase Storage fallback
- **Hosting Environments:**
  - *Backend:* Containerized Linux Web Service (Render, Singapore Region)
  - *Frontend:* Edge-cached static distribution (Cloudflare Pages)

---

## Repository Structure

```
.
├── ai-emr/
│   ├── emr-backend/                 # FastAPI Application Service
│   │   ├── main.py                  # ASGI entry point, middleware & CORS configuration
│   │   ├── nixpacks.toml            # Deployment environment & dependency build definitions
│   │   ├── requirements.txt         # Pinned Python package dependencies
│   │   ├── database/
│   │   │   └── database.py          # Unified DBWrapper, schema definitions & migrations
│   │   ├── routes/
│   │   │   ├── auth.py              # Authentication, password management & audit endpoints
│   │   │   ├── patients.py          # Patient registry, search & validation routes
│   │   │   └── reports.py           # Ingestion, duplicate checking & report mutation routes
│   │   └── utils/
│   │       ├── extractor.py         # Two-tier Claude AI multimodal extraction engine
│   │       ├── reference_ranges.py  # Demographic clinical reference range database
│   │       ├── security.py          # JWT lifecycle, Bcrypt hashing & rate limiting
│   │       └── storage.py           # S3 / Cloudflare R2 & Supabase storage interface
│   └── frontend/                    # React 18 Client Application
│       ├── package.json             # NPM dependencies & build scripts
│       ├── public/                  # Static assets & HTML root
│       └── src/
│           ├── App.js               # Shell router, navigation state & layout
│           ├── components/          # Reusable UI widgets (Sidebar, UploadModal, etc.)
│           ├── context/             # Global AppContext state provider
│           ├── pages/               # Views: Dashboard, Patients, LabReports, Settings
│           └── utils/               # Axios/Fetch API client & data normalizers
└── README.md
```

---

## Environment Configuration

### Backend Environment Variables (`.env`)

| Key | Type | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Required | API key for Anthropic Claude models |
| `JWT_SECRET_KEY` | Required | Cryptographic secret for signing JWT access tokens (min. 32 characters) |
| `DATABASE_URL` | Optional | PostgreSQL connection URI (if omitted, falls back to local SQLite) |
| `CORS_ORIGINS` | Required | Comma-separated list of allowed frontend origins (e.g. `http://localhost:3000`) |
| `DEFAULT_LAB_NAME` | Optional | Institutional facility name rendered on official documentation headers |
| `MASTER_CODE` | Optional | Emergency administrative break-glass credential reset code |
| `CLAUDE_MODEL` | Optional | Model identifier for deep vision inference (default: `claude-sonnet-4-5`) |
| `CLAUDE_HAIKU_MODEL`| Optional | Model identifier for fast-tier text extraction (default: `claude-haiku-4-5`)|
| `HAIKU_MIN_VALUES` | Optional | Minimum parameter threshold triggering vision escalation (default: `1`) |
| `R2_ACCOUNT_ID` | Optional | Cloudflare R2 Account ID for S3-compatible document storage |
| `R2_ACCESS_KEY_ID` | Optional | Cloudflare R2 Access Key ID |
| `R2_SECRET_ACCESS_KEY` | Optional | Cloudflare R2 Secret Access Key |
| `R2_BUCKET_NAME` | Optional | Cloudflare R2 storage bucket identifier |

### Frontend Environment Variables (`.env`)

| Key | Description |
|---|---|
| `REACT_APP_API_URL` | Base HTTP endpoint of the backend API (e.g. `https://api.yourdomain.com`) |

---

## Local Development Runbook

### Prerequisites
- Python 3.12+
- Node.js 18+ and `npm`

### 1. Backend Service Setup
```bash
cd ai-emr/emr-backend

# Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Populate ANTHROPIC_API_KEY and JWT_SECRET_KEY in .env

# Start ASGI development server
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```
Interactive OpenAPI documentation will be accessible at `http://localhost:8000/docs`.

### 2. Frontend Client Setup
```bash
cd ai-emr/frontend

# Install dependencies
npm install

# Start local development server
npm start
```
The application will launch at `http://localhost:3000`.
