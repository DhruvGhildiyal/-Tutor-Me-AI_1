# -Tutor-Me-AI_1
# Tutor Me AI — Web App (hardened build)

An AI-powered study dashboard: Ask AI, Assignment Solver, MCQ Generator,
Question Paper Generator, Content Explainer, PDF/Notes Summarizer, Career
Roadmap, Book Recommendations, personal Notes, and per-user History — all
behind a real login system.

This is a rebuild of an earlier single-file prototype, restructured to fix
every security, robustness, and architecture gap that prototype had. See
"What changed and why" below if you want the reasoning, not just the code.

## Features

- **Auth**: register/login/logout, passwords hashed with `werkzeug.security`,
  never stored or logged in plaintext.
- **CSRF protection** on every form and every `/api/*` POST/DELETE (Flask-WTF).
- **Rate limiting** per-route (Flask-Limiter) so one user can't burn your
  entire Gemini quota by scripting requests.
- **Structured AI output**: MCQ Generator and Question Paper Generator ask
  Gemini for JSON matching an explicit schema and validate it, instead of
  regex-parsing free text.
- **Sanitised AI output**: all free-text AI responses are converted from
  markdown to HTML and passed through `bleach` before ever reaching a
  browser — AI output is treated as untrusted input, same as any user input.
- **File upload safety**: extension allow-list, 10 MB hard cap
  (`MAX_CONTENT_LENGTH`), unique per-request temp files (no shared temp-file
  race between concurrent users).
- **Database migrations** via Flask-Migrate/Alembic instead of `db.create_all()`.
- **Blueprint architecture**: `auth`, `dashboard`, `tools` — not one giant `app.py`.
- **Structured logging** to `instance/logs/tutor_me.log`, with AI failures,
  auth events, and unhandled exceptions all logged server-side without
  leaking internals to the client.
- **Test suite** (pytest) covering auth flows, API auth boundaries, file
  validation, notes ownership isolation, and XSS sanitisation — 21 tests.
- **Production-safe config**: `ProductionConfig` refuses to start if
  `SECRET_KEY`, `DATABASE_URL`, or `GEMINI_API_KEY` aren't set as real
  environment variables — no silent fallback to a dev secret in production.

## Project layout

```
tutor_me_ai/
├── app/
│   ├── __init__.py          # create_app() factory, logging, error handlers
│   ├── config.py            # Development / Testing / Production configs
│   ├── extensions.py        # db, migrate, login_manager, csrf, limiter
│   ├── models.py            # User, Note, HistoryItem
│   ├── forms.py             # WTForms (CSRF + validation) for register/login
│   ├── prompts.py           # All prompt templates + JSON schemas, in one place
│   ├── ai.py                # Gemini wrapper: timeouts, retries, structured output
│   ├── utils.py             # Safe markdown rendering, file extraction, history
│   ├── blueprints/
│   │   ├── auth.py          # /register /login /logout
│   │   ├── dashboard.py     # / /dashboard
│   │   └── tools.py         # /api/* — every AI tool endpoint
│   ├── templates/
│   └── static/
│       ├── css/style.css
│       └── js/dashboard.js
├── migrations/               # Alembic migration history (flask db ...)
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   └── test_tools.py
├── run.py                     # `python run.py` — local dev server
├── wsgi.py                    # `gunicorn wsgi:app` — production
├── requirements.txt
├── .env.example
└── .gitignore
```

## 1. Install

```bash
cd tutor_me_ai
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## 2. Configure

```bash
cp .env.example .env
```

Edit `.env`:

```
GEMINI_API_KEY=your_real_key_here
SECRET_KEY=any_long_random_string
```

`.env` is git-ignored. Never commit it.

## 3. Set up the database

```bash
export FLASK_APP=run.py        # Windows: set FLASK_APP=run.py
flask db upgrade
```

This creates `instance/tutor_me_dev.db` with all tables via the checked-in
migration history (no more "drop the db and start over" when the schema
changes — that's the whole point of migrations).

## 4. Run it

```bash
python run.py
```

Visit **http://127.0.0.1:5000**.

## 5. Run the tests

```bash
pytest -v
```

All AI calls are mocked in tests (`unittest.mock.patch` on `ask_text` /
`ask_structured`), so the suite runs with no API key and no network access,
and won't burn your Gemini quota.

## Making schema changes later

```bash
# after editing app/models.py
flask db migrate -m "describe your change"
flask db upgrade
```

## Deploying to production

1. Set real environment variables on your host (not a `.env` file):
   `SECRET_KEY`, `DATABASE_URL` (e.g. a real Postgres URL), `GEMINI_API_KEY`.
   The app **will refuse to start** without all three — that's intentional.
2. Run migrations against the production DB: `flask db upgrade`.
3. Serve with a real WSGI server:
   ```bash
   gunicorn -w 4 -b 0.0.0.0:8000 wsgi:app
   ```
4. Put it behind a reverse proxy (nginx/Caddy) with HTTPS.
5. Consider `RATELIMIT_STORAGE_URI=redis://...` instead of the in-memory
   limiter if you run more than one gunicorn worker — the in-memory limiter
   doesn't share state across processes.

## What changed from the original prototype, and why

| Area | Before | After |
|---|---|---|
| CSRF | None | Flask-WTF CSRF on all forms + all `/api/*` mutating requests |
| Rate limiting | None | Per-route limits via Flask-Limiter |
| AI output rendering | Hand-rolled regex → raw HTML | `markdown` → `bleach`-sanitised HTML (XSS-safe) |
| MCQ / question paper | Free-text parsed loosely | Gemini `response_schema` + JSON validation |
| File uploads | No size cap, shared fixed temp filename | 10 MB cap, per-request unique temp files |
| Secrets | `SECRET_KEY` silently defaults to a dev value | Production config refuses to start without it |
| Errors | Generic `except Exception` everywhere, no logs | Typed `AIError`, distinguished status codes, rotating file logs |
| DB schema changes | Manual `db.create_all()` / delete the file | Flask-Migrate / Alembic |
| Code structure | Single 300-line `app.py` | Application factory + blueprints + config classes |
| Tests | None | 21 pytest tests: auth, API auth boundary, uploads, isolation, XSS |
| Duplicate signup race | Unhandled `IntegrityError` → 500 | Caught, friendly message |
| API auth failures | Redirect to `/login` (useless for `fetch()`) | Clean JSON 401 for `/api/*` |

## Known trade-offs (worth naming, not hiding)

- No email verification on signup — anyone can register with any email string.
- No service layer between routes and models — fine at this size, would be
  worth adding if this grew past a handful of tools.
- The in-memory rate limiter doesn't share state across multiple worker
  processes — swap to Redis (`RATELIMIT_STORAGE_URI`) before scaling out.
- No per-user AI usage budget/cost cap — Gemini calls cost money; a real
  deployment should track and cap usage per user, not just rate-limit requests.
