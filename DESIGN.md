# Design — Embeddable Widget & Lead-Capture Platform

## Problem

A customer wants to collect leads on their own website — a signup form, a CTA popover — without building or hosting any of the backend themselves. They should be able to define a widget in an admin dashboard, paste one `<script>` tag into their site, and start receiving validated, spam-filtered, geo-enriched submissions in their own dashboard — while every other customer's widgets and submissions stay completely invisible to them.

The hard part isn't the CRUD. It's that the submission endpoint is public, cross-origin, and reachable by anyone on the internet — not just the customer's own visitors. The system has to validate, rate-limit, and degrade gracefully under conditions it cannot predict or control.

## Data model

**`tenants`** — one row per widget owner (maps 1:1 to a Supabase Auth user; `tenant_id` = Supabase `user.id`, no separate signup flow needed).

**`widgets`**
| Column | Type | Notes |
|---|---|---|
| `id` | UUID, PK | public-facing widget id, used in the embed URL |
| `tenant_id` | UUID, FK → tenant | every query filters on this — tenant isolation lives here |
| `type` | enum | `signup_form` \| `cta_popover` |
| `title` | text | |
| `description` | text, nullable | |
| `fields` | JSONB | field definitions: `[{"name": "email", "label": "Email", "type": "email", "required": true}, ...]` |
| `button_text` | text | |
| `display_options` | JSONB | type-specific: popover delay/position, form layout, etc. |
| `bundle_version` | text | which versioned `widget.vN.js` this widget was created against — lets old embeds keep working across script upgrades |
| `created_at`, `updated_at` | timestamptz | |

**`submissions`**
| Column | Type | Notes |
|---|---|---|
| `id` | UUID, PK | |
| `widget_id` | UUID, FK → widget | |
| `tenant_id` | UUID, FK → tenant | denormalized on purpose — every submission query filters by tenant without a join, and it's the field every access-control check tests against |
| `data` | JSONB | the submitted field values, exactly as validated against the widget's `fields` schema |
| `ip_address` | inet | |
| `geo_country`, `geo_city` | text, nullable | null if enrichment failed or was skipped — never fabricated |
| `geo_provider` | text, nullable | which provider answered (`ip-api`, `ipapi-co`, or `null` if both failed) — this is the fallback chain's own evidence trail |
| `spam_flagged` | boolean | true if the honeypot caught it; still stored, never silently dropped, for auditability |
| `created_at` | timestamptz | |

Indexes: `widgets(tenant_id)`, `submissions(tenant_id, created_at)`, `submissions(widget_id, created_at)` — every dashboard query is one of these two shapes.

## API surface

**Widget management** — authenticated (Supabase bearer token, same pattern as `auth-api`), tenant-scoped on every query:
```
POST   /widgets              create
GET    /widgets              list (this tenant's only)
GET    /widgets/{id}         read (404 if not this tenant's, not 403 — don't confirm existence)
PUT    /widgets/{id}         update
DELETE /widgets/{id}         delete
GET    /widgets/{id}/embed   returns the <script> snippet to paste
```

**Widget delivery** — public, cached, no auth:
```
GET /widget.v1.js              versioned bundle — cache long, immutable
GET /widgets/{id}/config       this widget's render config — cache short (e.g. 60s), CORS-open
```

**Public submission** — public, CORS, the hardened path:
```
POST /submissions              body: {widget_id, data: {...}}
```
Validation → rate limit → spam check → geo enrichment (fallback chain) → store → side effect (failure-tolerant) → response.

**Dashboard** — authenticated, tenant-scoped:
```
GET /dashboard/submissions           paginated, filterable by widget_id
GET /dashboard/stats                 counts over time, per-widget, geo breakdown
```

## Layer sketch

```
routes/           HTTP only — parse request, call service, shape response, set status code
  widgets.py
  delivery.py
  submissions.py
  dashboard.py

services/          business logic, framework-agnostic
  widget_service.py        CRUD + tenant-scoping enforcement
  submission_service.py    orchestrates: validate -> rate-limit -> spam-check -> enrich -> store -> notify
  enrichment_service.py    provider fallback chain (A -> B -> none)
  notification_service.py  side effect; catches its own failures, never propagates them

repositories/       the only layer that touches SQL
  widget_repo.py
  submission_repo.py

core/
  auth.py           Supabase token verification dependency (reused from auth-api)
  rate_limit.py
  spam.py           honeypot check
  cors.py

db/
  models.py         SQLAlchemy/schema definitions
  migrations/
```

Rule this enforces: a route handler never touches the database directly, and a service never knows it's running inside FastAPI. Storage is swappable (SQLite for local dev, Postgres via Docker for the real run) without touching `services/` — the same one-file-changes discipline every earlier assignment in this track has proven.

## Non-goal

**No visual widget builder.** Widget configuration happens through the management API as JSON (`fields`, `display_options`) — not a drag-and-drop admin UI. The brief is explicit that the grade lives in the backend, and building a form-builder frontend would be scope creep away from the actual thing being taught: public API hardening, CORS, rate limiting, and graceful degradation. The "dashboard" the brief asks for is API endpoints plus a minimal table, not a polished frontend product.
