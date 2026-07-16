# tuitionloop

Coaching-centre manager: batches, attendance, fee-collection nudges, and parent progress updates — one dashboard for the front desk instead of a WhatsApp group, a fee register notebook, and a stack of attendance sheets.

**Status: planned — not yet built (50-SaaS challenge #48)**

## The problem
Indian tuition and coaching centres run on a patchwork: attendance on paper, fees tracked in a diary or a spreadsheet, and parent updates sent (if at all) as one-off WhatsApp messages. Nothing talks to anything else, so overdue fees and no-show students both slip through.

## Target buyer
Indian tuition/coaching-centre owners running one or more batches (subject/class groupings) with a handful of staff — the kind of centre where the owner is also the person chasing fee payments.

## Pricing hypothesis
Rs499/centre/month flat subscription (multi-tenant: each centre is its own billing unit, can have multiple staff logins).

## Stack
Cloudflare Worker (TypeScript, Workers Assets + `/api/*`) serving the staff dashboard. Supabase for magic-link auth and Postgres — every table carries `centre_id` and RLS enforces staff-of-that-centre access, not the single-owner model used elsewhere in this challenge. Fee and progress nudges go out via WhatsApp Cloud API on a cron sweep.

## How to continue this build
Read `docs/LLD.md` for the multi-tenant data model and API design, then `docs/PLAN.md` for the ordered, TDD-sized task list. `CLAUDE.md` points an agent session at both. Nothing is implemented yet — these three files are the source of truth.

## Risks / constraints (verified — do not soften)
- **Multi-tenant RLS is the core architectural challenge here** — unlike this challenge's single-owner reference, every table needs `centre_id` scoping and RLS policies keyed off a `centre_staff` membership table, not a bare `auth.uid() = user_id` check. Get this wrong and one centre can read another's students.
- **The Worker writes with the service-role key, which bypasses RLS entirely** — so centre-membership authorization must be re-checked in application code on every write handler, not assumed from the database layer.
- **WhatsApp Cloud API is a launch gate**: business-initiated fee reminders and progress nudges need Meta Business verification and pre-approved templates, an asynchronous process that can take days. v1 must not block on it.
- **No LLM in v1** — attendance, fees, and progress notes are structured data entered by staff; there is no generation step to get wrong or pay for. (A natural-language parent-update summary is a plausible v2, not v1.)
- Domain knowledge here draws on direct exposure to Indian coaching-centre workflows (built alongside a UGC NET exam-prep product) — validate the batch/fee-cycle assumptions below against a real centre before treating them as settled.
