# Build Log — AI usage

Honest record of where AI (Claude) helped, where it was wrong, and what I changed. The brief is explicit that "the AI wrote it" isn't an answer for any 2–3 lines an evaluator picks — this log exists so I can point to exactly what I need to be able to explain, and keep myself honest about actually understanding it rather than just shipping it.

## Phase 1 — Design

**What AI did:** Drafted the full `DESIGN.md` (data model, API surface, layer sketch, non-goal) from the capstone brief in one pass, based on decisions I made first (widget types: signup form + CTA popover; auth: reuse the Supabase pattern from `auth-api` rather than API keys, for consistency with existing work).

**What I need to be able to explain:** why `tenant_id` is denormalized onto `submissions` instead of only living on `widgets` (query performance — every dashboard/access-control check needs it without a join), and why the non-goal is a widget-builder UI specifically rather than something else (the brief grades the backend, not a frontend product).

**What I changed / would push back on:** — *(fill in as the design gets stress-tested during Phase 2 — a design doc that survives contact with real code unchanged usually means it wasn't examined closely enough)*
