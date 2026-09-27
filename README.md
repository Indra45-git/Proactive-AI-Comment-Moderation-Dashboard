# Proactive AI Comment Moderation Dashboard

AI-powered dashboard that helps social-media moderators keep comment sections healthy by automatically classifying incoming comments (Toxic, Spam, Question, Neutral, Positive) and surfacing the most harmful content first in a severity-sorted moderation queue.

> Final-year engineering thesis project · IEEE 830-compliant SRS · MVP build (1 month)

## Overview

Manually reviewing every comment on a social post is slow, exhausting, and keyword filters are easy to bypass. This project connects to a simulated social-media account, ingests comments, and uses an LLM to classify each one with a label, confidence score, and rationale. Moderators review flagged items in a prioritized queue and take action (Approve / Hide / Delete), with every decision recorded in an immutable audit trail.

## Features

- 🔐 JWT-based auth with role-based access control (Admin / Moderator)
- 🔌 Mock social-media account connection (settings page + "Test Connection")
- 📥 Manual or scheduled comment ingestion, with idempotent deduplication
- 🤖 LLM-based classification via a swappable Strategy pattern (OpenAI / Anthropic / local model / rule-based fallback)
- 🧵 Multi-turn context awareness — classification is grounded in the parent comment/thread, not just the isolated text, so sarcasm and reply-chain harassment are caught correctly
- 🌐 Multi-language detection — each comment's language is detected first (including code-mixed Hindi-English) and classified accordingly, instead of assuming English
- 📋 Severity-sorted moderation queue with filters, search, and pagination
- ✅ Approve / Hide / Delete actions with an immutable audit log
- 🔄 Real-time UI updates via WebSocket (Observer pattern)
- 📊 Dashboard metrics: volume, % toxic, actions taken, avg. latency

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | SvelteKit / React + DaisyUI |
| Backend | Python (FastAPI), HTTPX, Pydantic |
| Database | SQLite |
| LLM | OpenAI / Anthropic Claude API (pluggable) |
| Auth | JWT + CSRF protection |
| Deployment | Docker Compose |
| CI | GitHub Actions (ruff, eslint, pytest) |

## Architecture

Three-tier architecture:

1. **Presentation** — SvelteKit/React dashboard (queue, history, settings, metrics)
2. **Application** — FastAPI backend handling auth, ingestion, classification, and moderation logic
3. **Data** — SQLite with normalized tables (`users`, `mock_credentials`, `comments`, `classifications`, `moderation_actions`)

Two design patterns are core to the system:
- **Strategy Pattern** — interchangeable classification prompts/providers
- **Observer Pattern** — WebSocket-driven UI refresh after new comments are classified

See [`docs/SRS.pdf`](docs/) for the full component diagram, sequence diagram, and ER diagram (Mermaid source included).

### Context-Aware & Multi-Language Classification

- **Thread context**: `comments` gains a nullable `parent_comment_id` (self-referencing FK). On ingest, if a comment is a reply, its parent's text (and the parent's parent, up to a configurable depth) is resolved and passed into the prompt as `context`, so the Strategy sees the conversation, not an isolated line.
- **Language pipeline**: before classification, a lightweight language-ID step (e.g. `langdetect`/`fasttext`, or the LLM itself in a cheap first pass) tags the comment's `detected_language` (supports code-mixed strings like Hindi-English). This tag is passed into the prompt so the LLM classifies in-language instead of silently assuming English, and is stored on the `classifications` row for filtering/analytics.
- Both are implemented inside the existing `ClassificationStrategy.classify()` interface — no new pattern is introduced, just a richer `context` argument and one extra field in `ClassificationResult`.

## Getting Started

### Prerequisites

- Docker & Docker Compose
- An API key for your chosen LLM provider (OpenAI or Anthropic)

### Setup

```bash
git clone https://github.com/Indra45-git/comment-moderation-dashboard.git
cd comment-moderation-dashboard
cp .env.example .env   # fill in LLM_API_KEY, JWT_SECRET, etc.
docker compose up --build
```

- Backend: `http://localhost:8000`
- Frontend: `http://localhost:3000`
- API docs (Swagger): `http://localhost:8000/docs`

### Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | SQLite connection string |
| `LLM_API_KEY` | API key for the classification provider |
| `JWT_SECRET` | Secret used to sign auth tokens |
| `MOCK_SOCIAL_BASE_URL` | Base URL of the mock social-media API |
| `RATE_LIMIT_PER_MINUTE` | Requests/min allowed on ingest & classify endpoints (default: 30) |

## API Reference

Base URL: `/api/v1`

| Method | Path | Description | Role |
|---|---|---|---|
| POST | `/auth/login` | Get JWT | — |
| POST | `/auth/register` | Create user | Admin |
| GET | `/comments/queue` | Severity-sorted pending queue | Moderator |
| POST | `/comments/ingest` | Trigger fetch + classify | Moderator |
| POST | `/comments/{id}/action` | Approve / Hide / Delete | Moderator |
| GET | `/history` | Filterable audit history | Moderator |
| GET/POST | `/settings/credentials` | Manage mock credentials | Admin |
| GET | `/metrics/summary` | Dashboard metrics | Moderator |
| WS | `/ws/updates` | Real-time queue updates | — |

**Schema additions for context + language:**
- `comments.parent_comment_id` — nullable self-referencing FK, resolved into thread context at classification time
- `classifications.detected_language` — ISO language code (or `mixed` for code-mixed text) set by the language-ID step before classification

## Testing

```bash
# Backend unit + integration tests
cd backend && pytest --cov

# Frontend end-to-end tests
cd frontend && npx playwright test
```

Target: ≥80% unit-test coverage on core business logic (classification strategy, validation, RBAC).

## Project Status

MVP scope — see the [SRS](docs/) for what's explicitly out of scope (real OAuth integrations, multi-tenancy, streaming ingestion, custom model fine-tuning, mobile apps).

## License

Academic project — © 2026. All rights reserved for educational use.
