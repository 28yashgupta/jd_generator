# Deloitte JD Generator

A RAG-powered Job Description generator built with **FastAPI + React + Gemini + ChromaDB**.

It uses your existing 12,000+ JD PDFs as a knowledge base. When a user fills in fields like *title, location, skills, description*, the system retrieves the most semantically similar JDs and feeds them as context to Gemini to produce a polished, on-brand Deloitte JD. Final JD can be downloaded as **PDF or DOCX**.

---

## Features

- **Login / Register** (JWT auth, bcrypt-hashed passwords)
- **JD Generation** with RAG (ChromaDB + sentence-transformers + Gemini 1.5 Flash)
- **JD History** per user with view / download / delete
- **PDF + DOCX export** with full Deloitte branding (black + green `#86BC25`)
- **Deloitte-themed UI** (Source Serif Pro + Open Sans, branded colors)

---

## Architecture

```
┌─────────────────────┐         ┌──────────────────────────┐
│   React Frontend    │ ──────► │   FastAPI Backend        │
│   (Vite + Router)   │         │                          │
└─────────────────────┘         │   ┌──────────────────┐   │
                                │   │   Auth (JWT)     │   │
                                │   ├──────────────────┤   │
                                │   │   RAG Service    │   │
                                │   │   (ChromaDB +    │   │
                                │   │   sentence-trans)│   │
                                │   ├──────────────────┤   │
                                │   │   Gemini Service │   │
                                │   │   (1.5 Flash)    │   │
                                │   ├──────────────────┤   │
                                │   │   PDF + DOCX     │   │
                                │   │   Export         │   │
                                │   └──────────────────┘   │
                                │                          │
                                │   SQLite (users + JDs)   │
                                └──────────────────────────┘
```

---

## Project structure

```
jd-generator/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app
│   │   ├── config.py            # Settings (env)
│   │   ├── database.py          # SQLAlchemy setup
│   │   ├── models/
│   │   │   ├── db_models.py     # User, JDHistory tables
│   │   │   └── schemas.py       # Pydantic schemas
│   │   ├── routes/
│   │   │   ├── auth_routes.py   # /api/auth/*
│   │   │   ├── jd_routes.py     # /api/jd/*
│   │   │   └── admin_routes.py  # /api/admin/*
│   │   └── services/
│   │       ├── auth.py          # JWT + bcrypt
│   │       ├── rag_service.py   # ChromaDB + embeddings
│   │       ├── gemini_service.py# Gemini API wrapper
│   │       └── document_service.py # PDF + DOCX export
│   ├── data/
│   │   ├── jds_pdfs/            # ⬅ DROP YOUR 12k JD PDFs HERE
│   │   ├── chroma_db/           # vector index (auto-created)
│   │   └── generated/           # downloadable PDFs/DOCX
│   ├── ingest.py                # one-time ingestion script
│   ├── requirements.txt
│   └── .env.example
└── frontend/
    ├── src/
    │   ├── pages/               # Login, Register, Generator, History
    │   ├── components/          # Navbar
    │   ├── services/api.js      # axios + auth helpers
    │   └── styles/global.css    # Deloitte theme
    ├── package.json
    └── vite.config.js
```

---

## Setup

### Prerequisites
- Python 3.10+
- Node.js 18+
- A Gemini API key (https://aistudio.google.com/apikey)

### 1. Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# Edit .env and set GEMINI_API_KEY and SECRET_KEY
```

### 2. Drop your 12k JD PDFs

Copy all your existing PDFs into `backend/data/jds_pdfs/`.

### 3. Ingest the corpus (one-time, ~30-60 min for 12k PDFs on CPU)

```bash
cd backend
python ingest.py
```

This will:
- Read every PDF in `data/jds_pdfs/`
- Extract text with pypdf
- Embed each JD with `all-MiniLM-L6-v2` (downloads on first run, ~80 MB)
- Store vectors in ChromaDB at `data/chroma_db/`

Re-running is safe — already-indexed PDFs are skipped. Use `python ingest.py --force` to wipe and reindex.

### 4. Run the backend

```bash
cd backend
uvicorn app.main:app --reload --port 8000
```

Backend will be at http://localhost:8000. Interactive docs at http://localhost:8000/docs.

### 5. Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend at http://localhost:5173.

---

## Usage

1. Go to http://localhost:5173
2. Click **Create one** → register a username + password
3. You're auto-logged in → fill the JD form (title is mandatory; rest is optional but helpful)
4. Click **Generate JD** → wait ~5-10 sec for Gemini to respond
5. Download as **PDF** or **DOCX** from the right panel
6. View past JDs in the **History** tab

---

## API endpoints

| Method | Endpoint                          | Description                 |
|--------|-----------------------------------|-----------------------------|
| POST   | `/api/auth/register`              | Create account              |
| POST   | `/api/auth/login`                 | OAuth2 password flow → JWT  |
| GET    | `/api/auth/me`                    | Current user info           |
| POST   | `/api/jd/generate`                | Generate a new JD           |
| GET    | `/api/jd/history`                 | List user's JDs             |
| GET    | `/api/jd/{id}`                    | Get a specific JD           |
| GET    | `/api/jd/{id}/download/{pdf\|docx}` | Download generated file     |
| DELETE | `/api/jd/{id}`                    | Delete a JD                 |
| POST   | `/api/admin/ingest`               | Trigger PDF ingestion       |
| GET    | `/api/admin/stats`                | Vector DB stats             |

All `/api/jd/*` and `/api/admin/*` endpoints require `Authorization: Bearer <token>`.

---

## Tunables (in `.env`)

| Variable                  | Default              | Description                           |
|---------------------------|----------------------|---------------------------------------|
| `TOP_K_RETRIEVAL`         | `5`                  | How many similar JDs to retrieve      |
| `EMBEDDING_MODEL`         | `all-MiniLM-L6-v2`   | Any sentence-transformers model       |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `1440` (24h)     | JWT lifetime                          |

To switch Gemini model, edit `app/services/gemini_service.py` (`gemini-1.5-flash` → `gemini-1.5-pro` for higher quality, slower).

---

## Notes on scaling to 12k PDFs

- **Embedding** all 12k JDs takes ~30-60 min on CPU, a few minutes on GPU. Run once.
- ChromaDB stores everything on disk in `data/chroma_db/` (~200-400 MB for 12k JDs).
- Retrieval is sub-second per query.
- If memory is an issue during ingestion, lower `batch_size` in `rag_service.py`.

---

## Production checklist

- [ ] Set a strong `SECRET_KEY` (generate with `openssl rand -hex 32`)
- [ ] Use Postgres instead of SQLite (`DATABASE_URL=postgresql://...`)
- [ ] Restrict CORS origins in `app/main.py`
- [ ] Add rate limiting (e.g., `slowapi`)
- [ ] Restrict `/api/admin/*` to admin role only
- [ ] Use HTTPS + secure cookie storage instead of localStorage for tokens
- [ ] Add input sanitization on Gemini prompts
# jd_generator
