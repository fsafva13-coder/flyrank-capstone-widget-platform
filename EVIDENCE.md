# Evidence

One pasted proof per requirement in the brief's Section 6. Every entry below is filled in only once it's actually been run and verified — an unchecked box with no evidence is worth more than a checked one with none.

## Widget management
- [ ] Authenticated CRUD endpoints; unauthenticated requests rejected
- [ ] Multi-tenant isolation proven (tenant A cannot read/modify tenant B's data)

## Widget delivery
- [ ] Embed snippet generated per widget
- [ ] Public config endpoint, correct cache headers
- [ ] Widget JS served as a versioned bundle
- [ ] Widget renders on a page from a different origin than the API

## Public submission API
- [ ] Cross-origin submissions work (CORS + preflight)
- [ ] Malformed/oversized payloads rejected with clean 4xx JSON
- [ ] Valid submissions stored, linked to the right widget and tenant

## Abuse protection
- [ ] Rate limiting returns 429 under burst; legitimate traffic still served
- [ ] At least one spam control demonstrably blocks a spam submission

## Enrichment & safe side effects
- [ ] Fallback chain: provider A down → provider B answers
- [ ] Both providers down → submission still succeeds, no geo data
- [ ] Failing email/webhook does not block submission success

## Documentation
- [ ] README with architecture diagram, setup, API docs
