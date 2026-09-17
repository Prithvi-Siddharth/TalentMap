# TalentMap — AI-Powered Resume-to-Job Matching Platform

TalentMap analyzes a user's resume with an LLM, then continuously scans real job postings and scores them against that resume using semantic (embedding-based) similarity — so instead of a user manually re-searching job boards, an agent searches on their behalf and surfaces (and emails) the best real fits as they appear.

---

## Architecture

The app is **two independent processes that only communicate through MongoDB** — neither ever calls the other directly:

```
┌────────────────────┐         ┌──────────────────────────┐
│   app/main.py        │         │       worker.py            │
│   (FastAPI web app)  │ ◄─────► │  (APScheduler worker)      │
│                       │ MongoDB │                             │
│ auth · upload/analyze │  only   │ fetch jobs (Adzuna +       │
│ dashboard · history   │         │ RemoteOK) → match every    │
│ matches · settings    │         │ user → send digest email   │
└────────────────────┘         └──────────────────────────┘
```

This split is deliberate: the scheduler used to run inside FastAPI's own lifespan, which meant every dev reload, redeploy, or crash of the API killed the background job scan along with it. Splitting them means the API can restart freely (that's normal/healthy for a web process) without ever interrupting an in-progress scan, and the heavy embedding/matching work never competes with live HTTP request handling. A Mongo-based leader lock (with a TTL-based auto-expiring claim) makes it safe to accidentally run more than one worker instance at once — only one actually does the work.

---

## How it actually works

1. **Upload** — resume PDF goes to AWS S3; a small metadata record (owner, filename, S3 key) goes to MongoDB. Only one resume is ever "active" per user.
2. **Analyze** — triggered automatically right after upload: `pdfplumber` extracts raw text, which is sent to **Groq** (running Llama 3.3 70B) to extract structured entities — skills, education, experience, projects, certifications — as JSON. Results save immediately; matching against the job pool kicks off as a background task so the response isn't blocked (a full pass against a large pool measured 382s before embedding caching was added).
3. **Job sourcing** (worker, on a fixed interval) — fetches postings from **Adzuna** (quota-limited, so search terms are derived from users' actual resume titles) and **RemoteOK** (free, unlimited), normalizes both into one schema, and runs each posting through a deterministic keyword/regex entity extractor (vocab lives in `config/*.json`).
4. **Matching** — resume and job postings are each broken into 4 components (skills/experience/education/certifications), embedded with a **BGE sentence-transformer** (`bge-base-en-v1.5`), and compared with weighted cosine similarity (skills 40%, experience 30%, education 15%, certifications 15%). A component missing on either side is excluded from the score entirely rather than penalized to 0. Job embeddings are cached on first score and reused across every user and cycle.
5. **Notifications** — per-user configurable (immediate / daily / weekly / never), sent through a fallback chain: Resend → Brevo → SMTP.
6. **Everything else** (dashboard, matches list, market trends, activity feed, agent status) is server-rendered as an empty shell and hydrated client-side from JSON APIs — the JWT lives in `localStorage`, not a cookie, so there's nothing to render server-side on a plain page load.

---

## Folder structure (as actually built)

```
TalentMap/
├── app/
│   ├── main.py                    # FastAPI entrypoint — registers routers, mounts /static
│   ├── config.py                  # Pydantic BaseSettings — one source of truth for env vars
│   │
│   ├── routers/                   # auth, pages, dashboard/settings/activity/market-trends APIs
│   ├── step1_api/                 # upload_resume, analyze, resume history endpoints
│   ├── step2_nlp/                 # pdf_parser, preprocess, Groq-based NER, BGE embeddings
│   ├── step4_agent/                # job fetchers (Adzuna/RemoteOK), matcher, scheduler, email
│   ├── services/                  # dashboard/market-trends data aggregation, settings, activity log
│   ├── schemas/                   # Pydantic request/response models
│   ├── core/                      # db.py (Mongo + indexes), s3_utils, security (JWT), templates
│   └── tasks/, database/          # earlier Celery/Redis-based scheduler attempt — superseded by
│                                   # worker.py + APScheduler; not wired into anything running
│
├── worker.py                      # AI Job Agent — separate long-running process, see Architecture
├── templates/                     # Jinja2 pages (client-hydrated shells) + static/{css,js}
├── config/*.json                  # skills / certifications / education / experience vocab
├── scripts/seed_data.py
├── tests/                         # test_api.py, test_embeddings.py, test_pdf_parser.py — see
│                                   # Known limitations below
├── .env.example
├── requirements.txt
├── Dockerfile, docker-compose.yml # sketched, not finished — see Known limitations
└── .github/workflows/ci.yml
```

---

## Tech stack

| Concern | Choice |
|---|---|
| API framework | FastAPI + Jinja2 (async, built-in background tasks, Pydantic validation) |
| Database | MongoDB Atlas (flexible schema for evolving entity/job shapes) |
| Resume entity extraction | Groq (Llama 3.3 70B) — switched off Gemini, whose free tier required billing to get any quota |
| Matching | BGE sentence-transformers + weighted cosine similarity, not TF-IDF/keyword matching |
| File storage | AWS S3 (presigned URLs generated fresh per download, never cached) |
| Auth | JWT (python-jose) + bcrypt, plus Google OAuth (stateless, single-use handoff token) |
| Scheduling | APScheduler in a separate process (`worker.py`), not Celery/Redis |
| Job sources | Adzuna (quota-limited) + RemoteOK (free) |
| Email | Resend → Brevo → SMTP fallback chain |

---

## Running the app

Two terminals, same `.env`:

```
# Terminal 1 — the API
uvicorn app.main:app --reload

# Terminal 2 — the scheduler
python worker.py
```

The web app works fine with the worker not running — you just won't get new job postings or matches until you start it. `agent_scheduler_enabled` must also be `true` in `.env`, or the worker exits immediately after logging that it has nothing to do.

### Production

Same two processes, just run somewhere with continuous uptime (a local terminal doesn't count — closing it kills the process along with any scheduled scan mid-countdown). Any host that can run two long-lived processes works — e.g. two services on Railway/Render/Fly.io, or two systemd units on a VPS:

```
web:    uvicorn app.main:app --host 0.0.0.0 --port 8000
worker: python worker.py
```

Both read the same `MONGODB_URI` — no other coordination needed between them.

### Environment variables

See `.env.example` for the full list. At minimum you need `MONGODB_URI`, `SECRET_KEY`, AWS credentials + `AWS_BUCKET_NAME`, and `GROQ_API_KEY`. Adzuna, Google OAuth, and email provider keys are optional — each feature degrades gracefully (clear 503s / skipped sends) rather than crashing when unconfigured.

---

## Known limitations

Kept here deliberately rather than glossed over:

- **Tests are scaffolding, not implemented.** Every function in `tests/test_api.py`, `test_embeddings.py`, and `test_pdf_parser.py` is a `pass` with comments describing the intended test — running pytest reports them all as "passing," which is misleading. None actually assert anything yet.
- **Containerization is sketched, not built.** `Dockerfile` and `docker-compose.yml` are comment-only outlines; the app currently runs as two plain local/host processes, not containers.
- **Dead code from an earlier design.** `app/core/celery_app.py`, `app/tasks/`, and `app/database/mongodb.py` are a superseded Celery+Redis scheduling attempt — `REDIS_URL` isn't even set in `.env`. The real scheduler is `worker.py` + APScheduler.
- **No OCR fallback.** A scanned/image-only PDF resume yields empty extracted text — there's no image-to-text fallback for non-text PDFs.
- **Worker matching is sequential, not parallelized** — it processes registered users one at a time per scan cycle, which would need batching/parallelization to hold up at a much larger user base.
