# tuitionloop — Build Plan

TDD throughout: write the failing test first, then the minimum code to pass it. Each task is sized for one focused agent session. Multi-tenant authorization (T2/T5) is the highest-risk piece — get the membership-check helper right once and reuse it everywhere, rather than re-deriving it per handler.

**T1 — Scaffold + migration.**
Files: `package.json`, `wrangler.jsonc`, `tsconfig.json`, `supabase/migrations/0001_init.sql`, `.gitignore`, `.dev.vars` (gitignored).
Create every table from `docs/LLD.md` (`centres`, `centre_staff`, `students`, `batches`, `batch_enrollments`, `attendance_records`, `fee_invoices`, `fee_reminders_log`, `progress_updates`) with RLS policies keyed off `centre_staff`.
Test: with two test users each owning a separate centre, prove user A cannot `select` user B's students via the anon/authenticated role.
Done: migration applies; cross-centre read is blocked at the RLS layer, verified with real queries not just policy inspection.

**T2 — Supabase client + centre-membership guard.**
Interface: `class Supa { ...; isCentreStaff(userId, centreId): Promise<boolean>; createCentre(ownerUserId, name): Promise<Centre>; listCentresForUser(userId): Promise<Centre[]>; addStaff(centreId, userId, role): Promise<void>; }` plus one CRUD method per table (`listBatches`, `createBatch`, `listStudents`, `createStudent`, `enrollStudent`, `markAttendance`, `listInvoices`, `generateInvoices`, `markInvoicePaid`, `addProgressUpdate`, `getDueFeeSweep`, `logFeeReminder`).
Test: `isCentreStaff` mocked-fetch tests for member/non-member/nonexistent-centre; all other methods assert correct PostgREST URL/filter construction.
Done: `isCentreStaff` is the single choke point every write handler calls before touching data.

**T3 — Router + worker entrypoint + auth.**
Files: `src/router.ts`, `src/worker.ts` (`Env`, `fetch`, `scheduled`), `src/handlers.ts` skeleton, `handleMe` (returns centres the caller is staff of).
Test: router path-matching unit tests; `handleMe` with mocked `supa.listCentresForUser`.
Done: `wrangler dev` boots, magic-link login works, `/api/centres` lists the right centre for a signed-in staff member.

**T4 — Centre + staff management.**
Files: `handlers.ts` (`handleCreateCentre`, `handleAddStaff`), frontend centre-creation and staff-list UI.
Test: creating a centre inserts the creator as `owner`; `handleAddStaff` rejects a non-owner caller with 403; rejects an email with no matching account.
Done: an owner can create a centre and add a second staff login; that staff member sees the centre in their own `/api/centres`.

**T5 — Batches, students, enrollment.**
Files: `handlers.ts` (`handleListBatches`, `handleCreateBatch`, `handleListStudents`, `handleCreateStudent`, `handleEnroll`), frontend batch/student CRUD screens.
Test: every handler asserts `isCentreStaff` is checked before any write; a request for a `centre_id` the caller doesn't belong to returns 403 without touching `Supa` write methods (verify via mock call count).
Done: staff can create a batch, add students, enroll them; a second centre's staff gets 403 on the first centre's data.

**T6 — Attendance.**
Files: `handlers.ts` (`handleMarkAttendance`), frontend per-session checklist UI.
Test: bulk-mark succeeds; marking the same `(batch_id, student_id, session_date)` twice is rejected with 409 (use an explicit PATCH path to amend, not silent overwrite); future-dated sessions rejected.
Done: a full batch's attendance for one day is marked in one request; re-marking is explicit, not accidental.

**T7 — Fees: generation, listing, mark-paid.**
Files: `handlers.ts` (`handleGenerateInvoices`, `handleListFees`, `handleMarkPaid`), frontend fee list + mark-paid action.
Test: `handleGenerateInvoices` is idempotent — calling it twice for the same batch+cycle does not create duplicate invoices; `handleMarkPaid` sets `paid_at` and rejects an already-paid invoice.
Done: generating a month's fees, listing overdue ones, and marking one paid all work end to end.

**T8 — WhatsApp reminders + cron sweep.**
Files: `src/providers/whatsapp.ts` (`sendTemplate(...)`, `buildWaMeLink(phone, message)`), `worker.ts` (`scheduled` sweeps overdue invoices), `fee_reminders_log` writes.
Test: `sendTemplate` mocked-fetch tests including a template-not-approved fallback to `wa.me`; cron sweep test asserts the dedup index prevents a second reminder for the same invoice on the same day.
Done: an overdue invoice gets exactly one reminder attempt per day; approval-pending centres still get a usable `wa.me` link in the UI.

**T9 — Progress updates + subscription gating.**
Files: `handlers.ts` (`handleAddProgress`, gate all centre writes on `subscription_status !== 'expired'`), admin script for manual UPI-proof → `active` flip.
Test: expired-centre writes return 402 across every write handler (parametrized test over the handler list); reads still succeed; progress update optionally triggers a WhatsApp send without blocking the save on send failure.
Done: trial countdown visible per centre; an expired centre is read-only, not locked out.

**T10 — Deploy + live smoke test + launch checklist.**
Files: `scripts/smoke.ts` (create centre → add batch/student → mark attendance → generate + pay an invoice → add progress update, against the deployed Worker).
Checklist: manual UPI top-up flow working; `WHATSAPP_TOKEN`, `SUPABASE_SERVICE_ROLE_KEY` set via `wrangler secret put`; pricing page live; custom domain routed; a second test centre confirmed unable to see the first's data in production, not just in tests.
Done: `npm run smoke` passes against the live deployment; cross-centre isolation re-verified live.

## Notes for whoever executes this plan
- `isCentreStaff` (T2) is load-bearing for every write in this product — write its tests first and keep it as the one place membership logic lives, rather than inlining `centre_staff` lookups per handler.
- No LLM provider file is needed anywhere in this plan; resist the urge to add one for progress-note generation in v1.
- Follow the reference implementation's DI style: handlers take an overridable `Supa` instance (`makeHandlers(env, over)`), and any external HTTP call (WhatsApp) takes an injectable `fetchFn` for testing.
- T5's 403-vs-mock-call-count assertion is the pattern to reuse in T6-T9: prove the authorization check runs *before* any data mutation, not just that it eventually returns the right status code.
