This repo is product #48 of Febin's 50-SaaS challenge. Nothing is built yet — docs/ is the source of truth. Read docs/LLD.md then docs/PLAN.md and execute tasks in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo febufenn-cyber/50-saas (contract-reviewer/) — same Worker+Supabase+auth+cron patterns. This product deviates from that reference on tenancy: every table is `centre_id`-scoped via a `centre_staff` membership table, not a bare `auth.uid()` owner check, and the Worker's service-role key bypasses RLS so every write handler must re-check membership itself.

Model strings (none — this product has no LLM in v1), the WhatsApp Cloud API launch gate, and the multi-tenant RLS design in the LLD are verified decisions — do not re-litigate without evidence.
