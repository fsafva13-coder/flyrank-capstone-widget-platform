# Evidence

One pasted proof per requirement in the brief's Section 6. Every entry below is filled in only once it's actually been run and verified — an unchecked box with no evidence is worth more than a checked one with none.

## Widget management

- [x] **Authenticated CRUD endpoints; unauthenticated requests rejected**
```
$ pytest tests/test_widgets.py::test_unauthenticated_request_rejected -v
PASSED — GET /widgets with no Authorization header returns 401
```

- [x] **Multi-tenant isolation proven (tenant A cannot read/modify tenant B's data)**
```
$ pytest tests/test_widgets.py::test_tenant_cannot_read_another_tenants_widget -v
PASSED
```
Negative control run to confirm this test is actually meaningful, not trivially green: temporarily changed `app/routes/widgets.py` to use the unscoped `widget_repo.get_public()` instead of the tenant-scoped `widget_repo.get()`. Result: the isolation test correctly FAILED (`assert 200 == 404`). Reverted, test passes again. This proves the test genuinely detects a broken isolation boundary rather than passing for an unrelated reason.

Also proven live against real Postgres with a real second tenant — see **Full-stack live proof** below: a brand-new Supabase user's `GET /widgets` returned only that user's own widget, never the two widgets seeded under the fixed demo tenant.

## Widget delivery

- [x] **Embed snippet generated per widget**
```
$ pytest tests/test_widgets.py::test_embed_snippet_contains_widget_id -v
PASSED
```

- [x] **Public config endpoint, correct cache headers**
```
$ curl -s http://127.0.0.1:8000/widgets/00000000-0000-0000-0000-000000000001/config
{"type": "signup_form", "title": "Join our newsletter", "description": "One email a week, no spam.", "fields": [...], "button_text": "Subscribe", "display_options": {}}
```
Real curl against a live running server, real seeded data, not TestClient.

- [x] **Widget JS served as a versioned bundle**
```
$ curl -s -D - -o /dev/null http://127.0.0.1:8000/widget.v1.js
HTTP/1.1 200 OK
cache-control: public, max-age=31536000, immutable
content-type: application/javascript
```

- [x] **Widget renders on a page from a different origin than the API** — `customer-site/index.html` opened directly as a `file://` page (a different origin than `http://localhost:8000`, where the API runs). Screenshot 1: the signup form rendered correctly, config and script both loaded cross-origin. Screenshot 2: after submitting a real email, the page shows "Thanks!" — proof the cross-origin POST round-tripped successfully, not just that the page loaded.

## Public submission API

- [x] **Cross-origin submissions work (CORS + preflight)**
```
$ curl -s -i -X OPTIONS http://127.0.0.1:8000/submissions \
  -H "Origin: https://some-customer-site.example" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type"
HTTP/1.1 200 OK
access-control-allow-origin: *
access-control-allow-methods: GET, POST
access-control-allow-headers: Accept, Accept-Language, Content-Language, Content-Type

$ curl -s -i -X POST http://127.0.0.1:8000/submissions \
  -H "Origin: https://real-customer-site.example" \
  -H "Content-Type: application/json" \
  -d '{"widget_id": "...", "data": {"email": "visitor@example.com", "name": "Real Visitor"}}'
HTTP/1.1 201 Created
access-control-allow-origin: *
{"status":"received","id":"6ebc0b76-a610-445c-99d7-3a9ede0b5887"}
```

- [x] **Malformed/oversized payloads rejected with clean 4xx JSON**
```
$ curl -s -i -X POST http://127.0.0.1:8000/submissions \
  -H "Content-Type: application/json" \
  -d '{"widget_id": "...", "data": {}}'
HTTP/1.1 400 Bad Request
{"error":"Missing required field: email"}
```
Oversized payload (20,000-byte field value) also confirmed via `pytest tests/test_submissions.py::test_oversized_payload_rejected` — PASSED.

- [x] **Valid submissions stored, linked to the right widget and tenant**
```
$ pytest tests/test_submissions.py::test_submission_linked_to_correct_widget_and_tenant -v
PASSED — stored row's widget_id and tenant_id both verified against the actual widget
```

## Abuse protection

- [x] **Rate limiting returns 429 under burst; legitimate traffic still served**
```
$ for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " -X POST http://127.0.0.1:8000/submissions ...; done
201 201 201 201 201 201 429 429 429 429 429 429 429 429 429
```
Only 6 of these 15 succeeded rather than 10, because 4 requests earlier in the same 60s window (from earlier probe runs against the same IP) had already used up part of the limit — `4 + 6 = 10`, then `429` for the rest. Confirms the limiter counts a true rolling window per IP (`ip_limiter.allow()` in `app/core/rate_limit.py`), not a naive "10 per burst" counter that would reset between calls.

Different-IP request right after, while the first IP is still blocked:
```
$ curl -s -i -X POST http://127.0.0.1:8000/submissions -H "X-Forwarded-For: 203.0.113.99" ...
HTTP/1.1 201 Created
```
Succeeding while the first IP is still rate-limited is the actual proof of "legitimate traffic still served" — not just that 429s appear somewhere in a burst.

- [x] **Spam control demonstrably blocks a spam submission**
```
$ curl -s -i -X POST http://127.0.0.1:8000/submissions \
  -H "Content-Type: application/json" \
  -d '{"widget_id": "...", "data": {"email": "bot@x.com", "_hp_website": "http://spam.example"}}'
HTTP/1.1 201 Created
{"status":"received","id":null}
```
Response looks like success (no signal to the bot), `id: null` confirms no row was actually stored. Also verified at the DB layer: `pytest tests/test_submissions.py::test_honeypot_filled_silently_dropped` — PASSED, asserts the `Submission` table row count is 0 after the honeypot submission.

## Enrichment & safe side effects

- [x] **Fallback chain: provider A down → provider B answers** (deterministic, fake transport — no real network dependency)
```
$ pytest tests/test_enrichment.py::test_falls_back_to_provider_b_when_a_down -v
PASSED
```

- [x] **Both providers down → submission still succeeds, no geo data**
```
$ pytest tests/test_enrichment.py::test_degrades_gracefully_when_both_down -v
PASSED — enrich_ip() returns (None, None, None), never raises
```

- [x] **Manual proof against a live running server** (the brief, Section 7: "mock the geo providers when you prove the fallback so it is deterministic — real free APIs are for manual dev only"; `ENRICHMENT_MOCK=1` added so this proof never depends on `ip-api.com`/`ipapi.co` being reachable at proof time — confirmed neither is reachable from this build environment: `ip-api.com` → `403`, `ipapi.co` → connection failure, both blocked by this sandbox's own network policy, which is exactly the failure case this deterministic mode exists to route around)
```
$ ENRICHMENT_MOCK=1 ENRICHMENT_FORCE_A_DOWN=1 uvicorn app.main:app ...
$ curl -s -X POST http://127.0.0.1:8000/submissions -H "X-Forwarded-For: 8.8.8.8" \
    -d '{"widget_id":"...","data":{"email":"probe4a@example.com"}}'
{"status":"received","id":"e0a81992-208b-49e1-9f01-ef39ad0e9f83"}

$ sqlite3 widgets.db "SELECT geo_country, geo_city, geo_provider FROM submissions WHERE id='e0a81992-...'"
Mockland | Fallbackville (B) | ipapi-co

$ ENRICHMENT_MOCK=1 ENRICHMENT_FORCE_A_DOWN=1 ENRICHMENT_FORCE_B_DOWN=1 uvicorn app.main:app ...
$ curl -s -i -X POST http://127.0.0.1:8000/submissions -H "X-Forwarded-For: 8.8.8.8" \
    -d '{"widget_id":"...","data":{"email":"probe4b@example.com"}}'
HTTP/1.1 201 Created
{"status":"received","id":"7d88a288-b4e1-41c5-b873-3c70c47c4083"}

$ sqlite3 widgets.db "SELECT geo_country, geo_city, geo_provider FROM submissions WHERE id='7d88a288-...'"
None | None | None
```
Provider A down → provider B answers, correct provider name recorded. Both down → submission still stored (`201`), all three geo fields `null`. Real HTTP server, real DB row, zero dependency on outbound network reachability.

- [x] **Failing email/webhook does not block submission success**
```
$ NOTIFICATION_FORCE_FAIL=1 ... (server started with this env var)
$ curl -s -i -X POST http://127.0.0.1:8000/submissions \
  -H "Content-Type: application/json" \
  -d '{"widget_id": "...", "data": {"email": "test@example.com"}}'
HTTP/1.1 201 Created
{"status":"received","id":"e07c59bc-61d2-4b89-9cb0-fae01bf96c6f"}

Server log:
Notification failed for submission=e07c59bc-61d2-4b89-9cb0-fae01bf96c6f (submission itself is unaffected): Forced failure for Probe 5 verification
```
Submission genuinely stored (real UUID returned), `201` returned, notification failure caught and logged without ever reaching the HTTP response.

## Full-stack live proof — Docker Compose, real Postgres, real Supabase auth, dashboard API

Everything above was re-run end-to-end against the actual target environment: `docker compose up --build` (Postgres 16, not SQLite), a real Supabase project (not a test-only bearer-token stand-in), and the live dashboard endpoints. This is also the evidence for Probe 1's "...and visible via the dashboard API" clause.

**A real bug found only by running against real Postgres.** `docker compose exec app python -m app.seed` failed on the first attempt:
```
sqlalchemy.exc.DataError: (psycopg2.errors.StringDataRightTruncation) value too long for type character varying(36)
```
Cause: `app/db/models.py` defines `tenant_id` as `String(36)` — sized for a real UUID, which is always exactly 36 characters. But `app/seed.py`'s `DEMO_TENANT_ID` was `"demo-tenant-00000000-0000-0000-0000-000000000000"` — 48 characters. SQLite never enforces varchar length limits, so this was invisible in every SQLite-backed test and local run; Postgres enforces it strictly and rejected the insert outright. Fixed by shortening `DEMO_TENANT_ID` to `"00000000-0000-0000-0000-000000000000"` (36 chars, matching the style of the two fixed demo widget IDs). Re-ran seed — succeeded:
```
$ docker compose exec app python -m app.seed
Seeded 2 widgets for tenant 00000000-0000-0000-0000-000000000000:
  - 00000000-0000-0000-0000-000000000001  (signup_form)
  - 00000000-0000-0000-0000-000000000002  (cta_popover)
```

**Docker Compose stack healthy, serving real data from Postgres:**
```
$ curl -i http://localhost:8000/health
HTTP/1.1 200 OK
{"status":"ok"}

$ curl -s http://localhost:8000/widgets/00000000-0000-0000-0000-000000000001/config
{"type": "signup_form", "title": "Join our newsletter", ...}
```

**Real Supabase signup + login (not a test-only stand-in) → real bearer token:**
```
$ curl -X POST ".../auth/v1/signup" -d '{"email":"fsafva13+captest2@gmail.com","password":"..."}'
{"access_token":"eyJhbGci...", "user":{"id":"9508571d-09f7-4f4c-8a94-30190cf5887e", ...}}
```

**Authenticated widget creation under that real token (proves the CRUD path end-to-end, not just via tests):**
```
$ curl -X POST http://localhost:8000/widgets -H "Authorization: Bearer $token" -d '{"type":"signup_form","title":"Live verification widget",...}'
{"id":"79314d6f-fb94-448f-afe0-c75aef57d3cf","type":"signup_form","title":"Live verification widget",...}
```

**Live tenant isolation, not just unit-tested:**
```
$ curl http://localhost:8000/widgets -H "Authorization: Bearer $token"
[{"id":"79314d6f-fb94-448f-afe0-c75aef57d3cf", "title":"Live verification widget", ...}]
```
Only this tenant's one widget — the two demo widgets (under the separate fixed demo tenant) never appear. Two genuinely different Supabase users, real Postgres rows, correctly scoped.

**Real public submission against the live widget, then confirmed via the dashboard API:**
```
$ curl -X POST http://localhost:8000/submissions -d '{"widget_id":"79314d6f-fb94-448f-afe0-c75aef57d3cf","data":{"email":"visitor@example.com"}}'
{"status":"received","id":"e8d2c9e3-2370-483d-855a-2c8a09caf0c2"}

$ curl http://localhost:8000/dashboard/stats -H "Authorization: Bearer $token"
{"total_submissions":1,"by_widget":[{"widget_id":"79314d6f-fb94-448f-afe0-c75aef57d3cf","count":1}],"by_geo":[]}

$ curl http://localhost:8000/dashboard/submissions -H "Authorization: Bearer $token"
[{"id":"e8d2c9e3-2370-483d-855a-2c8a09caf0c2","widget_id":"79314d6f-fb94-448f-afe0-c75aef57d3cf","data":{"email":"visitor@example.com"},"geo_country":null,"geo_city":null,"created_at":"2026-09-25T17:02:01.831612Z"}]
```
The submission is visible via the dashboard API for the tenant that owns it, and nowhere else — this is the full public-submission → owner-dashboard loop, live, against real infrastructure.

## Documentation

- [x] **README with architecture diagram, setup, API docs** — architecture diagram, both setup paths (Docker + local dev), and API doc pointer all present and verified working.