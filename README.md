# Embeddable Widget & Lead-Capture Platform

A platform that lets a customer define a widget (signup form or CTA popover), hand visitors one `<script>` tag, and safely catch what the public internet sends back — validated, rate-limited, spam-filtered, geo-enriched, and dashboarded.

FlyRank AI Backend Engineering Internship — Capstone project.

> **Status:** All phases complete and verified. Widget delivery confirmed live in a real browser from a second origin; all 6 of the brief's acceptance probes verified fresh against the current code, including a deterministic geo-fallback proof and, for Probe 1, a real submission confirmed visible through the live dashboard API (see `EVIDENCE.md`); 24 tests passing. Also run end-to-end against the real target environment — `docker compose up --build` against Postgres 16 and a real Supabase account, not just SQLite and a test-only auth stand-in — which surfaced and fixed one genuine bug SQLite had been hiding (a `tenant_id` column truncation, see `EVIDENCE.md` and `BUILDLOG.md`).

## Architecture

See [`DESIGN.md`](./DESIGN.md) for the full data model, API surface, and layer sketch.

```
Widget Owner (authenticated, Supabase bearer token)
  → Widget Management API → Widget DB (tenant-isolated) → embed snippet

Customer Website (any origin)
  <script src="http://localhost:8000/widget.v1.js" data-widget-id="...">
  → GET /widgets/:id/config (public · cached 60s · CORS-open)
  → render widget

Website Visitor
  → POST /submissions (public · CORS-open)
    validate (schema, size) → rate limit (per-IP + per-widget) → honeypot check
    → geo enrichment (ip-api.com → ipapi.co → none) → store → notify (failure-tolerant)

Widget Owner (authenticated)
  → Dashboard API ← submissions + stats, tenant-scoped
```

```
app/
├── main.py             # entrypoint — CORS, router wiring, startup
├── schemas.py           # widget/submission request+response schemas
├── dashboard_schemas.py
├── seed.py               # demo data (fixed ids — see below)
├── routes/               # HTTP only — no SQL, no business logic
│   ├── widgets.py         # authenticated CRUD, tenant-scoped
│   ├── delivery.py        # public — versioned JS bundle, cached config
│   ├── submissions.py     # public — the hardened path
│   └── dashboard.py       # authenticated — submissions + stats
├── services/              # business logic, framework-agnostic
│   ├── widget_service.py
│   ├── submission_service.py    # orchestrates the full hardened path
│   ├── enrichment_service.py    # provider fallback chain
│   └── notification_service.py  # failure-tolerant side effect
├── repositories/          # the only layer that touches SQL
├── core/
│   ├── auth.py             # Supabase bearer-token verification
│   ├── rate_limit.py        # in-memory sliding-window limiter
│   └── spam.py               # honeypot check
├── db/
│   ├── models.py             # SQLAlchemy models (Widget, Submission)
│   └── session.py             # SQLite (dev) / Postgres (Docker) — env-driven
└── static/
    └── widget.v1.js            # the actual embed script, vanilla JS

customer-site/
└── index.html            # the "different origin" test page (Section 7's constraint)

tests/                    # 24 tests, all passing, zero real-network dependency
```

## Setup

### Option A — Docker (Postgres, the real target environment)

```bash
git clone https://github.com/fsafva13-coder/flyrank-capstone-widget-platform.git
cd flyrank-capstone-widget-platform
cp .env.example .env
# edit .env: fill in your real SUPABASE_URL and SUPABASE_KEY
docker compose up --build
```

Seed demo data (two widgets, fixed ids, safe to re-run):
```bash
docker compose exec app python -m app.seed
```

The API is now live at `http://localhost:8000`.

### Option B — Local dev (SQLite, no Docker)

```bash
pip install -r requirements.txt
cp .env.example .env
# edit .env: fill in your real SUPABASE_URL and SUPABASE_KEY
uvicorn app.main:app --reload
python -m app.seed
```

### Try the embed flow

Open `customer-site/index.html` directly in a browser (`file://` works, or serve it: `python -m http.server 5500` from that folder). It's pre-wired to the fixed demo widget id (`00000000-0000-0000-0000-000000000001`) that `python -m app.seed` creates — no editing needed on a clean clone.

### Run the tests

```bash
pytest tests/ -v
```

All 24 tests run with zero real network access — Supabase auth is swapped for a test-only bearer-token stand-in (see `tests/conftest.py`'s docstring for exactly what that does and doesn't prove), and the geo-enrichment fallback chain is tested against a fake HTTP transport, not the real providers.

## API documentation

See [`capstone.yaml`](./capstone.yaml) for the full endpoint list, and [`EVIDENCE.md`](./EVIDENCE.md) for real, run proof of every requirement in the brief.

## Politeness / hardening rules

| Rule | Value |
|---|---|
| Rate limit (per IP) | 10 requests / 60s |
| Rate limit (per widget) | 100 requests / 60s |
| Spam control | Hidden honeypot field (`_hp_website`), silently dropped if filled |
| Max submission payload | 10,000 bytes |
| Geo enrichment retry | 1 provider fallback (ip-api.com → ipapi.co → none), 3s timeout each |
| Widget bundle cache | `max-age=31536000, immutable` (versioned URL) |
| Widget config cache | `max-age=60` |

## Limitations

- The rate limiter is in-memory and single-process — it does not coordinate across multiple app instances/workers. Documented here rather than silently pretending otherwise; a real production deployment behind a load balancer would need a shared store (Redis) instead.
- CORS is deliberately wide open (`allow_origins=["*"]`) on the public endpoints. This is intentional, not an oversight — an embeddable widget must load from any customer's site, an origin this API can never know in advance. The security boundary here is validation/rate-limiting/spam control on what's sent, not which origins can call the endpoint.
- Geo enrichment's fallback chain is proven deterministically via `ENRICHMENT_MOCK=1` (see `EVIDENCE.md`) rather than against the real providers, per the brief's own guidance (Section 7: "mock the geo providers when you prove the fallback... real free APIs are for manual dev only"). Without that flag, both providers still call the real APIs as normal.

## Future Improvements

- Real-time dashboard (WebSockets/SSE)
- Redis-backed rate limiting for multi-instance deployments
- A minified, versioned production widget bundle
- Double opt-in + GDPR export/delete endpoints

## License

MIT