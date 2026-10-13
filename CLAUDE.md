# CLAUDE.md — bnh-backend

## Custodian rules

- Money is integer kobo only; never floats; never computed by an AI at runtime.
- DoA thresholds live in a database/config table and are enforced on the server. A role cannot approve above its band.
- Authority = a position held by a person for a period. Designations are data.
- The event log is append-only. No role, including admin, can edit or delete it. Corrections are new entries.
- Never commit .env files, keys, passwords or real BNH data.
- `bnh-ui` reads the backend URL from an environment variable, never hard-coded.
- For every task: `git pull` → make the change → run the app and all tests → fix until green → explain in plain words → commit → push → report the commit hash.
- Never force-push. If a push is rejected, pull, resolve, re-test, then push.
- When a change affects the API between the two repos, say so, and list what the other repo must change.

## This repo

Backend for **Custodian by Brendan**, Brendan Nicholas Holdings' (BNH) requisitions and Delegation of Authority (DoA) system — Module 1. Implementation is well underway (migration head `0020`). Inspect the code and recorded verification before assuming a capability is complete. The Custodian rules above override anything below.

### Known gaps against the Custodian rules (fix as dedicated tasks, not silently)

These are current code facts carried over from the previous agent's instructions, which conflict with the Custodian rules. Do not extend them; when you touch these areas, raise the conflict and plan a migration.

- **Money**: amounts are stored as `NUMERIC(17,2)` naira, computed with Python `Decimal`, and exchanged over the API as decimal strings (`app/authority/policy.py`, `app/requisitions/models.py`, migration `0005`). The rule requires integer kobo. Converting changes the API contract, so `bnh-ui` (`src/lib/money.ts`, generated `src/lib/api/schema.ts`, forms and displays) must change in step.
- **DoA thresholds**: the band limits (₦5m / ₦100m / ₦500m) and requester floors are hard-coded in `app/authority/policy.py`. The rule requires them in a database/config table enforced on the server.
- **Event log**: `custodian.audit_events` and signed evidence have append-only triggers (migration `0002`), but the app runs with a single schema-owner `DATABASE_URL` that could alter or drop those triggers. The rule says no role can edit or delete the log, so treat this as an open gap, not an accepted design.
- Authority as data is already in place: offices (`app/organisation/models.py` `Office`) link a person to a role, entity/department, `valid_from` / `valid_until` and an authorisation reference.

### Product references

- `docs/reference/ui_scrrens.md` (filename typo is real) — the authoritative release scope: fourteen screens.
- `docs/reference/implementation_pan.md` — detailed requirements, stable feature IDs, acceptance criteria and the canonical status tracker. Do not create a second tracker.
- `docs/reference/recap.md` — authority matrix and Board workflow. `docs/reference/features.md` — scope summary only.
- `docs/reference/ui_todos.md` (endpoint-consumption notes for the UI) and `docs/reference/task_done.md` (verified delivery notes). Older copies also sit at `docs/ui_todos.md` and `docs/task_done.md`.
- `docs/development.md` — full onboarding guide for both repos. `docs/render.md` — deployment and operations.
- These are snapshots of a shared `docs/` folder from the original workspace, which does not exist here. `scripts/sync_requirements.py` only works with that folder.

The user's latest explicit decisions take precedence over older requirements. Raise genuine business-policy conflicts instead of inventing resolutions.

### Release scope: fourteen screens

1. Sign In 2. Forgot Password 3. Set Password / Activate Account 4. Dashboard 5. Requisitions 6. Create / Edit Requisition 7. Requisition Details & Review 8. My Tasks / Approval Inbox 9. Board Resolution Workspace 10. Vendors 11. Staff & Access Management 12. Organisation & Authority 13. Audit Log 14. My Account & Security

Drawers/dialogs handle vendor details and edits, invitations, appointments, department edits and signing; a header drawer holds notifications; a viewer shows attachments and generated documents. Build backend capabilities only where they support these screens or necessary security, persistence and deployment. No extra apps, workflow builder, contracts, scoring, payments, or Modules 2–4. Approved means authorised, not paid.

### Stack

- Python 3.12 (`.python-version`), FastAPI, Pydantic v2, SQLAlchemy 2 (async, asyncpg), PostgreSQL 16, Alembic.
- uv for dependencies; commit `pyproject.toml`, `uv.lock`, `.python-version`. Install with `uv sync --locked`.
- Ruff (lint + format, line length 100), mypy (strict, pydantic plugin), pytest + pytest-asyncio (`asyncio_mode = auto`).
- ReportLab + pypdf for PDFs (bundled Liberation fonts in `app/assets/fonts/`, SIL OFL), Pillow, argon2-cffi, httpx.
- External services: Resend (email), Cloudinary (private documents, authenticated raw assets). Both optional for starting the app.

### Structure

- `app/main.py` — full assembly (`app.main:create_app`) and background workers. `auth_app.py`, `org_app.py`, `access_app.py`, `vendor_app.py` are partial factories kept for tests.
- `app/core/` — config, database sessions, errors, banking helpers.
- `app/<domain>/` — `access`, `audit`, `authority`, `board`, `dashboard`, `evidence`, `exports`, `history`, `identity`, `notifications`, `organisation`, `requisitions`, `transfers`, `vendors`. Files: `models.py`, `schemas.py`, `service.py`, `router.py` as needed.
- `migrations/versions/` — Alembic history, schema `custodian`. Forward migrations only.
- `tests/<domain>/` — unit and real-PostgreSQL integration/security tests.
- `scripts/` — `start.sh`, `provision_database`, `create_operator`, `dev_database` (isolated test cluster), `export_openapi`, `browser_fixture` (used by `bnh-ui` Playwright), `issue_recovery`, `notification_status`, `configure_audit_reviewer`, `sync_requirements`.

Conventions: build a modular monolith, not microservices. Routes stay thin; services own business transitions and transactions; query helpers never commit by themselves. No generic CrudUtil/Result/decorator frameworks.

### How it connects to bnh-ui

- `bnh-ui` (SvelteKit) is a sibling directory, `../bnh-ui`. The browser calls `/api/v1` on the UI origin; the UI's `src/hooks.server.ts` proxies it to this backend using `BACKEND_URL`.
- Session cookies are HttpOnly and CSRF-protected; the backend's `ALLOWED_ORIGINS` must list the UI origin exactly.
- API types: `uv run python -m scripts.export_openapi` writes `../bnh-ui/src/lib/api/openapi.json` and `schema.ts`. Commit the regenerated files in `bnh-ui` with the matching change; never hand-edit them.
- `bnh-ui`'s Playwright runner starts this repo with `.venv/bin/python -m scripts.browser_fixture` on port 8017, so keep the sibling layout.

### Install, run, test

```bash
uv sync --locked
cp .env.example .env            # then edit locally; never commit
uv run python -m scripts.provision_database   # fresh database only
uv run alembic upgrade head
uv run python -m scripts.create_operator      # first account only
uv run uvicorn app.main:create_app --factory --host 127.0.0.1 --port 8000 --reload
```

Health: `GET /api/v1/health/ready`. API docs: `/docs`. A 404 at `/` is expected. No default accounts.

Checks (all must pass before committing):

```bash
uv run python -m scripts.dev_database   # isolated test PostgreSQL on 127.0.0.1:55439, writes .env.test
uv run ruff check .
uv run ruff format --check .
uv run mypy app
uv run pytest
```

Tests need local PostgreSQL 16 binaries (`initdb`, `pg_ctl`, `psql`; set `PG_BIN` if not in `/usr/lib/postgresql/16/bin`). SQLite or mocks do not prove transactions, migrations, privileges, audit immutability or races. Skipped or unavailable integration tests are not a pass; report them as not run.

### Deployment

Render, Docker web service, using `Dockerfile` and `render.yaml` (`healthCheckPath: /api/v1/health/ready`). `scripts/start.sh` runs `alembic upgrade head` with `DATABASE_URL`, then Uvicorn; startup fails if migration fails. Production env: `ENVIRONMENT=production`, `DATABASE_URL`, `DATABASE_SSL=true`, `COOKIE_SECURE=true`, `ALLOWED_ORIGINS`, plus optional Resend/Cloudinary settings. Development and production use different databases. Deploy the backend before the matching frontend. Details in `docs/render.md`.

Branch: all work, pushes and Render deploys use `main` on `github.com/Sisa4real/BNH-Backend1`.

### Engineering conventions

- Work on one feature at a time, identified by its stable feature ID in `docs/reference/implementation_pan.md`, and finish it fully (backend, migrations, permissions, audit, config, errors) before starting the next. Put the feature ID in the commit message.
- Update the existing feature block in the tracker plus `ui_todos.md` / `task_done.md` with real behaviour, commands and results. Mark Implemented only after checks pass. Keep code, testing, deployment and staff acceptance separate.
- Preserve existing partial work and other people's changes; stage only the current feature.
- API under `/api/v1`, typed Pydantic models, extra fields rejected on business commands. One error shape: `code`, safe `message`, optional field errors, request ID. No tracebacks exposed.
- Scope every read/write/search/count/attachment/export/decision by `resource:action` permission plus record/entity scope. Apply scope before pagination and counts.
- Commit a business transition, its audit events and its delivery intent atomically; an audit failure rolls back the transition. Use expected versions, row locks and actor-scoped idempotency keys bound to the payload. Never duplicate a signed decision.
- Durable outbox jobs for email, exports and archives. Notification failure never fabricates an approval.
- UTC timestamps in storage; display Africa/Lagos. Meeting dates stay separate from recording/signing timestamps.
- Secrets only in `.env` (ignored) and Render; placeholders in `.env.example`. Redact bank data, signatures and credentials from logs. Never seed personal data from the sample forms.
- Three genuine staff-run requisitions are a human acceptance gate. Never fabricate records or sign for staff.

### Business rules (current policy data)

**Approval routing (policy `bnh-doa-v2`, NGN, inclusive upper limits):**

| Requester      | ₦1–₦5m                                   | >₦5m–₦100m     | >₦100m–₦500m | >₦500m |
| -------------- | ---------------------------------------- | -------------- | ------------ | ------ |
| Ordinary staff | Own department's HOD                     | Chief of Staff | MD           | Board  |
| HOD            | Self-approval via department sub-account | Chief of Staff | MD           | Board  |
| Chief of Staff | MD                                       | MD             | MD           | Board  |
| MD             | Board                                    | Board          | Board        | Board  |

- Route directly to the higher of the amount band and the requester's office floor. No sequential chain.
- Derive identity, department, membership and appointments from server records, never from browser input.
- Only the eligible assigned officeholder may act; seniority grants no substitute authority. Missing, inactive or ambiguous assignments block the action — never invent substitutes. Recheck privileges when the decision commits.
- Freeze the policy version and route explanation with each submitted revision. Historical v1 submissions keep their recorded route.
- **HOD self-approval**: a current, unique HOD of the requester's own department may approve their own ₦1–₦5m requisition with a separate fresh approval signature. Submission alone does not approve. No self-reject/return; the requester may retract before action. This is the only self-approval exception.
- **Board**: needs a real meeting resolution. The Company Secretary records outcome and evidence, signs and submits; a different person as Board Chairman (also not the requester) confirms and signs or returns it. Outcomes: approval, rejection, deferment, conditional approval (unresolved conditions are a hold). No online voting, quorum calculation or unilateral Board approval.

**Revisions:** freeze submitted revisions, vendor/bank snapshots, attachments, declarations, routing and signatures. Master-data edits never rewrite past requests. Returns create a successor revision with fresh signatures and recalculated routing. Use explicit lifecycle actions, not writable status fields. Quantities are positive (≤4 dp); line totals are recomputed server-side; the minimum submission is ₦1; tax/delivery are explicit cost lines.

**Auth and evidence:** invitation-only individual accounts; opaque server sessions in HttpOnly Secure cookies; CSRF and origin checks. Technical admins are not financial approvers; no self-grants of appointments. Signing needs fresh authentication, explicit intent, and a single-use, expiring challenge bound to actor and exact content/version; never auto-apply a saved signature. Files stay private with size/type/content validation; downloads are authorised before issuing short-lived URLs. Malware scanning is deferred by owner decision (2026-10-08): never claim files are scanned. Cloudinary is the chosen store (2026-10-08).

**Administrator visibility (2026-10-09):** a full administrator (active, activated, writable, holding `staff:manage`, `organisation:manage` and `office_assignment:manage`) may read requisitions and vendors across BNH, including drafts and inactive companies. This grants no office, membership, edit/submit/approve, vendor write, Audit Log, or document/PDF/Board-evidence access. Recheck on every request.

**Vendor bank visibility:** ordinary staff see full details for vendors they created; others need `vendor:read_sensitive`. Current HOD, Chief of Staff, MD, Company Secretary and Board Chairman appointments, and full administrators, may read bank details in scope. Read-only reviewers stay restricted. Summaries show `supplied`/`restricted`. New account numbers are exactly 10 ASCII digits, stored as strings with leading zeroes; bank fields are all-or-none.

**Signed bank transfer letters (2026-10-11):** the approval ceremony binds debit account (company account, or department sub-account for HOD self-approval — never fall back), beneficiary and amount into the challenge, with explicit transfer-signature consent. Use the actual approver's signature or the Chairman's for unconditional Board approval. No letter for conditional/rejected/deferred outcomes. Freeze bank data and signature; missing details are shown before signing and block a complete letter without reversing the approval. Requesters download their own letters; admin oversight grants no download.

**PDFs:** bundled Liberation Serif Regular (body) and Liberation Sans Bold (headings). New jobs use template `requisition-v2`; existing `requisition-v1` jobs keep their template. Do not print hashes, signature/export UUIDs or verification appendices; keep them internal. Payment letter: no revision suffix, no signing-timestamp line, no repeated footer.

**Draft deletion and retraction:** requesters delete only their own drafts (audit and evidence retained internally; hidden from normal reads). Retraction to Draft only before any approver decision or Board work; resubmission creates a fresh signed revision.

**Automatic uploads:** file selection uploads immediately, buffered privately per requester/company/creation key for 24 hours with per-file and per-person limits, cleaned hourly and at startup. Saving claims exact upload IDs into Cloudinary atomically with the draft and audit; reject expired, missing, foreign or changed sets.

**Department reference codes:** new references are `BNH-{3-letter dept code}-{shared sequence, ≥6 digits}`. Codes are three ASCII letters, uppercase, unique per company ignoring case; edits need the current version and audit. Never accept a caller-supplied reference. Existing references are never renumbered; no counter reset.
