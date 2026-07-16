# tuitionloop — Low-Level Design

## Architecture

```
        ┌──────────────────────┐
        │  Browser (staff app)  │
        │  centre selector +    │
        │  dashboard             │
        └───────────┬────────────┘
                     │ fetch (JWT bearer)
                     ▼
        ┌──────────────────────┐
        │  Cloudflare Worker    │
        │  (TS, ESM, strict)    │
        │  Workers Assets +     │
        │  /api/*                │
        └──┬──────────────┬─────┘
           │ plain fetch  │ fetch
           │ (PostgREST + │
           │  GoTrue)     ▼
           │        ┌──────────────────┐
           ▼        │ WhatsApp Cloud API │
  ┌──────────────────┐  (fee reminders,   │
  │ Supabase           │  progress nudges)  │
  │ - GoTrue auth      └──────────────────┘
  │ - Postgres + RLS         ▲
  │   (centre_id scoped      │ cron sweep
  │   on every table)        │ (overdue fees,
  └──────────────────┘        due nudges)
```

Request flow: staff logs in (magic link) → frontend fetches `/api/centres` to find which centre(s) the JWT's user is staff of → all subsequent requests carry `centre_id` and the Worker re-validates centre membership against `centre_staff` before reading or writing (the service-role key bypasses RLS, so this check is not optional). A scheduled Worker sweeps fee invoices past `due_date` still `pending`, flips them to `overdue`, and sends a WhatsApp reminder (deduped per invoice per day); a separate sweep can nudge parents with a batch's latest progress update.

## Data model

Every table below carries `centre_id`. RLS policies check membership via `centre_staff`, not `auth.uid() = <owner column>` — this is the deliberate deviation from this challenge's single-owner reference.

```sql
create table public.centres (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  owner_user_id uuid not null references auth.users(id) on delete cascade,
  subscription_status text not null default 'trial' check (subscription_status in ('trial','active','expired')),
  trial_ends_at timestamptz not null default (now() + interval '14 days'),
  created_at timestamptz not null default now()
);
alter table public.centres enable row level security;

create table public.centre_staff (
  centre_id uuid not null references public.centres(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role text not null default 'teacher' check (role in ('owner','teacher','front_desk')),
  created_at timestamptz not null default now(),
  primary key (centre_id, user_id)
);
alter table public.centre_staff enable row level security;
create policy "read own memberships" on public.centre_staff for select using (auth.uid() = user_id);
create policy "read centre if staff" on public.centres for select using (
  exists (select 1 from centre_staff cs where cs.centre_id = centres.id and cs.user_id = auth.uid()));

create table public.students (
  id uuid primary key default gen_random_uuid(),
  centre_id uuid not null references public.centres(id) on delete cascade,
  name text not null,
  parent_phone text, -- E.164, for WhatsApp nudges
  created_at timestamptz not null default now()
);

create table public.batches (
  id uuid primary key default gen_random_uuid(),
  centre_id uuid not null references public.centres(id) on delete cascade,
  name text not null,
  subject text,
  fee_amount_paise bigint not null check (fee_amount_paise > 0),
  fee_cycle text not null default 'monthly' check (fee_cycle in ('monthly','one_time')),
  created_at timestamptz not null default now()
);

create table public.batch_enrollments (
  batch_id uuid not null references public.batches(id) on delete cascade,
  student_id uuid not null references public.students(id) on delete cascade,
  centre_id uuid not null references public.centres(id) on delete cascade, -- denormalized for RLS/index
  status text not null default 'active' check (status in ('active','inactive')),
  enrolled_at timestamptz not null default now(),
  primary key (batch_id, student_id)
);

create table public.attendance_records (
  id uuid primary key default gen_random_uuid(),
  centre_id uuid not null references public.centres(id) on delete cascade,
  batch_id uuid not null references public.batches(id) on delete cascade,
  student_id uuid not null references public.students(id) on delete cascade,
  session_date date not null,
  status text not null check (status in ('present','absent','late')),
  marked_by uuid not null references auth.users(id),
  created_at timestamptz not null default now(),
  unique (batch_id, student_id, session_date)
);

-- state machine: pending -> paid | pending -> overdue (cron, due_date passed) -> paid
create table public.fee_invoices (
  id uuid primary key default gen_random_uuid(),
  centre_id uuid not null references public.centres(id) on delete cascade,
  student_id uuid not null references public.students(id) on delete cascade,
  batch_id uuid not null references public.batches(id) on delete cascade,
  amount_paise bigint not null check (amount_paise > 0),
  due_date date not null,
  status text not null default 'pending' check (status in ('pending','paid','overdue')),
  paid_at timestamptz,
  created_at timestamptz not null default now()
);

create table public.fee_reminders_log (
  id uuid primary key default gen_random_uuid(),
  centre_id uuid not null references public.centres(id) on delete cascade,
  invoice_id uuid not null references public.fee_invoices(id) on delete cascade,
  sent_at timestamptz not null default now(),
  whatsapp_message_id text,
  status text not null check (status in ('sent','failed'))
);
create unique index fee_reminder_dedup on public.fee_reminders_log (invoice_id, (sent_at::date));

create table public.progress_updates (
  id uuid primary key default gen_random_uuid(),
  centre_id uuid not null references public.centres(id) on delete cascade,
  student_id uuid not null references public.students(id) on delete cascade,
  batch_id uuid not null references public.batches(id) on delete cascade,
  note text,
  test_score int,
  max_score int,
  sent_to_parent boolean not null default false,
  created_at timestamptz not null default now()
);

-- repeat this policy shape for every centre-scoped table above:
create policy "centre staff full access" on public.students for all using (
  exists (select 1 from centre_staff cs where cs.centre_id = students.centre_id and cs.user_id = auth.uid()));
```

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/centres` | POST | JWT | Create a centre, caller becomes `owner` in `centre_staff` | 400 missing name |
| `/api/centres` | GET | JWT | List centres the caller is staff of | 401 |
| `/api/centres/:id/staff` | POST | JWT (owner only) | Add an existing user (by email) as staff | 403 not owner; 404 email has no account |
| `/api/batches` | GET, POST | JWT + centre membership | List/create batches for a centre | 403 not staff of `centre_id` |
| `/api/students` | GET, POST | JWT + centre membership | List/create students; enroll into a batch | 403; 400 duplicate enrollment |
| `/api/attendance` | POST | JWT + centre membership | Bulk-mark a batch's attendance for one date | 403; 409 date already marked (use PATCH to amend) |
| `/api/fees` | GET | JWT + centre membership | List invoices, filter by status/batch/student | 403 |
| `/api/fees/generate` | POST | JWT + centre membership | Generate this cycle's invoices for a batch (idempotent per batch+cycle) | 403; 409 already generated for this cycle |
| `/api/fees/:id/mark-paid` | POST | JWT + centre membership | Flip invoice to `paid` | 403; 404 not found |
| `/api/progress` | POST | JWT + centre membership | Add a progress update, optional immediate WhatsApp send | 403; 502 WhatsApp send failure (update still saved) |

Cron (`scheduled()`, daily): sweep `fee_invoices` where `due_date < today` and `status = 'pending'` → set `overdue`, send WhatsApp reminder, log to `fee_reminders_log` (dedup index makes re-runs safe).

## LLM strategy
**Not applicable to v1.** Attendance, fees, and progress notes are structured data staff enter directly — there's no generation or extraction step. A future version could draft a natural-language parent summary from a student's attendance/score history, but v1 sends templated messages built from the raw fields (no cost, no hallucination risk, no async concerns).

## Frontend pages
Login (magic link) · Centre picker (if staff of more than one) · Dashboard (batches, today's attendance status, overdue fee count) · Batch detail (roster, fee cycle) · Attendance marking (per-session checklist, bulk save) · Fees (invoice list, mark-paid, overdue flags) · Student progress (score history, add update) · Settings (staff list, subscription/billing, trial countdown).

## Error handling
Every write handler re-checks `centre_staff` membership before touching data (service-role key bypasses RLS, so this is the real gate). Attendance and fee-generation endpoints are idempotent on their natural keys (`batch_id, student_id, session_date` / `batch_id` + cycle) so a retried request can't double-write. WhatsApp send failures are logged and surfaced in the UI but never roll back the underlying data change (a progress note or paid invoice is still true even if the notification failed).

## Integrations and launch gates
**WhatsApp Cloud API — BLOCKS automated fee reminders and progress nudges**, same gate as elsewhere in this challenge: Meta Business verification + pre-approved message templates, asynchronous approval. v1 ships a manual "notify via WhatsApp" button that opens a `wa.me` deep link (no approval needed) alongside in-app visibility of overdue fees; automated cron-driven sends are a fast-follow once Cloud API is approved.

## Security notes
RLS via `centre_staff` membership on every table (sketch above); because the Worker uses the service-role key for all writes, the *actual* security boundary is the application-level membership check in each handler, not RLS alone — treat RLS as defense-in-depth for the (rare) direct-client-read path, not the primary control. Secrets (`WHATSAPP_TOKEN`, `SUPABASE_SERVICE_ROLE_KEY`) via `wrangler secret put`. Input validation: attendance `session_date` cannot be in the future, fee amounts must be positive paise, phone numbers validated to E.164 before any WhatsApp send.

## Out of scope for v1
Parent-facing login/portal (parents only receive WhatsApp/email nudges, no account); Razorpay recurring billing (manual UPI + admin activation, matching this challenge's day-1 approach); full invite-by-email token flow for staff (v1: owner adds an already-registered user's email directly); multi-branch centre hierarchies; LLM-generated progress summaries; SMS fallback for nudges.
