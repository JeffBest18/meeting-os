# Meeting OS

**Ask a venture firm's own memory a question and get a cited answer.**

> *"What has the team been looking at this month?"*
> *"Where does [company] stand, and does the pipeline tracker agree with what the founder said on the call?"*
> *"Give me a two-week firm digest."*

Meeting OS indexes a small firm's calls, email, documents, chat, calendar, pipeline sheets and CRM into one searchable layer, then lets an agent answer questions across all of it, with every claim cited back to its source.

I designed and built it solo for the seed-stage fund where I work, a team of six. It is read-only by design and restricted to the firm's own domain.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full design.

---

## Why I built it

A small fund's real knowledge doesn't sit in its CRM. It sits in call transcripts, email threads and half-updated spreadsheets, and those sources often disagree. A founder says one thing on a call; the tracker says another; the partner remembers a third. Nobody has time to reconcile them.

The goal was a system that could answer the questions partners actually ask, and that would **show its sources and flag disagreements** rather than smooth them over. An answer the team can't verify is worse than no answer.

---

## Design decisions

**Citations are validated, not just generated.** The agent must ground each claim in a retrieved chunk, and citations are checked against the retrieval set before an answer is returned. Unsupported claims are dropped rather than shown.

**Hybrid retrieval.** Dense vector search (pgvector) and BM25 keyword search run in parallel and are merged with reciprocal rank fusion. Vector search handles paraphrase; keyword search catches company names, tickers and jargon that embeddings blur.

**Entity resolution before retrieval.** The same company appears as a domain in email, a name in a transcript and a row in a sheet. CRM companies seed a canonical entity layer, and later mentions resolve by domain first, with a blocklist for shared domains (gmail.com, calendar invites, etc.) that would otherwise merge unrelated companies.

**Read-only by default.** The system can read seven sources but writes to none of them. The blast radius of a bad agent action is zero.

**Tiered answers.** Cheap retrieval-only answers for simple lookups; the full agent loop, with tool use and enrichment, only when a question needs it.

**Degrades gracefully.** The API boots with any subset of connector credentials. A missing token disables that connector instead of breaking the app.

---

## Status

| Component | State |
|---|---|
| Database schema (entities, documents, chunks, hybrid search, row-level security) | Built |
| Ingest pipeline (chunk → embed → resolve → index) | Built |
| Entity resolution + domain blocklist | Built, 19 tests passing |
| Call-transcript connector (webhook + poll + backfill) | Built |
| Retrieval (hybrid RRF search, timeline, recent activity) | Built |
| Agent loop (streaming, tools, citation validation, tiering) | Built |
| Chat API (SSE), auth, webhook endpoint | Built |
| Background worker (sync, digest, retention) | Built |
| Email / Drive / Sheets / Calendar / Slack / CRM connectors | Built |
| Frontend | Built |

---

## Stack

| Layer | Technology |
|---|---|
| API | Python 3.11, FastAPI, server-sent events |
| Agent | Claude (tool use, streaming) |
| Database | Supabase Postgres + pgvector, row-level security |
| Workers | Celery + Redis |
| Sources | Fathom, Gmail, Google Drive, Google Sheets, Google Calendar, Slack, HubSpot |
| Deploy | Docker on Railway (API + worker), Vercel (frontend) |

---

## Local setup

```bash
# Python 3.11
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.11
uv venv --python 3.11 backend/.venv
uv pip install --python backend/.venv/bin/python -r backend/requirements.txt

# Configure (annotated list of variables in .env.example)
cp backend/.env.example backend/.env

# Apply the schema: run database/schema.sql in your Supabase SQL editor

# Run
cd backend
./.venv/bin/uvicorn main:app --reload                                  # API on :8000
./.venv/bin/celery -A workers.celery_app worker -B --loglevel=info     # worker

# Tests
./.venv/bin/python -m pytest tests -q
```

`GET /health` should return `{"status": "healthy"}`. Startup logs which connectors are live.

---

## Backfill

Order matters. CRM companies load first so they seed the canonical entity layer, and later mentions resolve by domain rather than guesswork:

```python
from workers.tasks import backfill_source
for source in ["hubspot", "fathom", "gcal", "gsheets", "gmail", "gdrive", "slack"]:
    backfill_source.delay(source, days=84)
```

Slack goes last: non-Marketplace Slack apps are heavily rate-limited on history requests, so a one-time backfill runs over hours. Steady-state sync is only a handful of messages per cycle.

---

*Built by Jeff Best · [LinkedIn](https://www.linkedin.com/in/jeffrey-w-best/)*
