# Embeddable Widget & Lead-Capture Platform

A platform that lets a customer define a widget (signup form or CTA popover), hand visitors one `<script>` tag, and safely catch what the public internet sends back — validated, rate-limited, spam-filtered, geo-enriched, and dashboarded.

FlyRank AI Backend Engineering Internship — Capstone project.

> **Status:** Phase 1 (design) complete. Build in progress.

## Architecture

See [`DESIGN.md`](./DESIGN.md) for the full data model, API surface, and layer sketch.

```
Widget Owner (authenticated)
  → Widget Management API → Widget DB (tenant-isolated) → embed snippet

Customer Website (any origin)
  <script src="widget.v1.js?id=123">
  → GET /widgets/:id/config (public · cached · CORS)
  → render widget

Website Visitor
  → POST /submissions (public · CORS)
    validation → rate limit → spam check → geo enrichment (fallback chain) → store → side effect
```

## Setup

*(filled in as the build progresses — Phase 1 has no runnable code yet)*

## API documentation

See [`capstone.yaml`](./capstone.yaml) for the full endpoint list.

## Limitations

*(honest list, filled in as the project develops)*

## License

MIT
