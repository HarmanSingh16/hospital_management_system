# Hospital Management System (HMS)

A single-hospital management system where patients, doctors, receptionists and admins work against shared appointment and medical data — built as a backend-engineering portfolio project to demonstrate depth on concurrency, fine-grained authorization, and crash-resilient background work.

**Stack:** Go 1.27 (Gin + pgx + sqlc) · PostgreSQL · React 19 (TypeScript + Vite + Tailwind)

---

## Table of Contents

- [Overview](#overview)
- [The Three Problems This Project Solves](#the-three-problems-this-project-solves)
- [Tech Stack](#tech-stack)
- [Repository Layout](#repository-layout)
- [Architecture](#architecture)
- [Key Design Decisions](#key-design-decisions)
- [Roles and Permissions](#roles-and-permissions)
- [Appointment Lifecycle](#appointment-lifecycle)
- [API Surface](#api-surface)
- [Getting Started](#getting-started)
- [Seeded Accounts](#seeded-accounts)
- [Project Status](#project-status)
- [Roadmap](#roadmap)
- [Documentation](#documentation)

---

## Overview

The scope is deliberately narrow: a hospital that schedules 15-minute consultations, records what happened at each visit, and enforces that clinical data is visible only to the patient and the doctor treating them.

**Users:** patients · doctors · receptionists (who also act as nurses) · admins

**Covered:** appointment booking and cancellation, per-visit medical records, structured prescriptions, vitals, and email notifications.

The design goal is **depth over breadth**. Most of the surface is routine CRUD; the effort is concentrated into three problems that are hard to get right and easy to explain in an interview:

1. **Concurrent booking** — two staff members booking the same 15-minute slot at the same moment must produce exactly one booking.
2. **Authorization over sensitive data** — access to clinical data depends on the *relationship* between the user and the patient, not just the user's role.
3. **Reliable time-based background work** — the 24-hour reminder and the slot-removed email must survive restarts, retry on failure, and be cancelled when the appointment is cancelled.

Everything else — departments, patient intake, office hours, the front-desk queue — is scaffolding to make those three problems real.

---

## The Three Problems This Project Solves

### 1. Concurrent booking with zero double-booking

The naive approach (check availability, then insert) races. Two receptionists can both read "slot is free" and both insert. The fix here is to make the database the arbiter rather than the application.

A PostgreSQL **GiST exclusion constraint** makes overlapping bookings for the same doctor physically unrepresentable:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE appointments
ADD CONSTRAINT no_overlapping_doctor_appointments
EXCLUDE USING gist (
  doctor_id WITH =,
  tstzrange(starts_at, ends_at) WITH &&
) WHERE (status NOT IN ('cancelled'));
```

`btree_gist` supplies the scalar-equality operator class for `doctor_id`; `tstzrange` with `&&` supplies range overlap. The partial predicate means a cancelled appointment frees its slot for rebooking.

Concurrent attempts collide on the index, and the second one gets PostgreSQL error `23P01` (`exclusion_violation`). The service catches it and translates it into a domain error:

```go
// internal/service/appointment_service.go
if errors.As(err, &pgErr) && pgErr.Code == "23P01" {
    return domain.ErrSlotAlreadyBooked
}
```

which the handler maps to `409 Conflict` / `SLOT_CONFLICT`. Exactly one booking survives; the loser gets a clean, actionable error rather than a 500.

**Why not a lock?** A `SELECT ... FOR UPDATE` on the target slot needs a row to lock, but the slot row does not exist yet — it is projected, not stored (see [Dynamic slot projection](#dynamic-slot-projection-no-slot-bloat)). An advisory lock keyed on `(doctor_id, slot)` would work but requires the application to hold the lock correctly in every code path. A declarative constraint is correct by construction: it holds no matter which code path writes the row.

### 2. Authorization over sensitive data

A role check answers *"may a doctor read clinical data?"* — yes. It does not answer *"may **this** doctor read **this** patient's chart?"* That second question is the interesting one.

The rule the system enforces:

> Clinical data (diagnosis, notes, prescriptions) is visible only to the patient themselves and to a doctor who has **started a consultation** with that patient.

Three properties make this work:

- **Access is derived from an action, not a role.** Chart access is granted by starting a consultation, not by being the assigned doctor. Reassigning an appointment to another doctor grants that doctor *nothing* — it closes the backdoor where reassignment becomes a path to a chart.
- **Access is sticky.** Once granted, a doctor keeps read access to that patient's whole chart, including other doctors' records. The accepted consequence is that this never expires, so accountability moves to the audit trail.
- **Every access is logged.** Reads and writes of clinical data, account creation, schedule changes and cancellations are all recorded in an append-only audit log, and the patient can see who viewed their chart and when.

The admin role has **no path to clinical data at all**. The system is designed to defend against a malicious application admin, not against someone with direct database access — which is why onboarding is invite-based and roles are immutable:

| Control | What it blocks |
| --- | --- |
| Invite-based onboarding (admin never knows credentials) | Puppet doctor accounts controlled by an admin |
| One role per account, fixed at creation | An admin promoting themselves to doctor |
| Chart access starts only when the doctor starts a consultation | Reassigning an appointment as a backdoor to a chart |
| Append-only audit log | Hiding abuse after the fact |
| Patient-visible access log | Abuse that slips through, because the affected patient can see it |

### 3. Reliable time-based background work

In-memory timers (`time.AfterFunc`) are the obvious implementation and the wrong one: they die with the process, cannot be cancelled reliably, and have nowhere to record a retry.

The system uses a **transactional outbox**. Every email is a row written in the *same database transaction* as the state change that caused it:

```sql
-- Written inside the booking transaction, for bookings >= 24h ahead
INSERT INTO notifications (recipient_id, appointment_id, type, scheduled_for, status)
VALUES ($1, $2, 'appointment_reminder', $starts_at - INTERVAL '24 hours', 'pending');
```

Because the notification row and the appointment are committed atomically, the email can never disagree with the booking — no lost reminders, no ghost reminders for slots that were never booked.

Cancellation is the same mechanism in reverse, inside the cancellation transaction:

```sql
UPDATE notifications
SET status = 'cancelled'
WHERE appointment_id = $1 AND status = 'pending';
```

And the dispatcher polls the outbox with `FOR UPDATE SKIP LOCKED`, so multiple worker replicas can run without delivering an email twice:

```sql
SELECT id, recipient_id, type, scheduled_for
FROM notifications
WHERE status = 'pending' AND scheduled_for <= NOW()
ORDER BY scheduled_for ASC
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

The `notifications` table carries `retry_count` and `last_error` columns for exponential-backoff retries, and a `scheduled_for` column that lets a job survive a restart — the row simply stays `pending` until its time comes.

Email is sent in exactly three cases, and nowhere else: **account invite**, **appointment reminder** (24h before the slot), and **slot removed** (a doctor's schedule change cancelled the visit).

---

## Tech Stack

### Backend — `HMS-backend`

| Concern | Choice | Notes |
| --- | --- | --- |
| Language | Go 1.27.0 | |
| HTTP framework | Gin v1.12.0 | |
| Database driver | `pgx/v5` v5.11.0 | Native PostgreSQL driver, `pgxpool` |
| Query layer | `sqlc` | Type-safe Go from SQL, no ORM |
| Migrations | `goose` | Versioned SQL, up/down per migration |
| Auth | `golang-jwt/jwt/v5` v5.3.1 | HS256, stateless |
| Password hashing | `golang.org/x/crypto` | bcrypt |
| Config | `godotenv` | `.env`-driven |
| Database | PostgreSQL | Requires the `btree_gist` extension |

### Frontend — `hms_frontend`

| Concern | Choice | Notes |
| --- | --- | --- |
| Framework | React 19 | Function components + hooks only |
| Language | TypeScript 5.7 | `strict`, `noUnusedLocals`, `noUnusedParameters` |
| Build tool | Vite 6 | |
| Styling | Tailwind CSS 3.4 | Custom design tokens, no UI component library |
| Routing | react-router-dom 7 | `BrowserRouter`, role guards |
| HTTP client | axios 1.8 | Request/response interceptors |
| Dates | date-fns 4 | |
| Icons | lucide-react | |
| Server state | @tanstack/react-query 5 | `QueryClientProvider` mounted; queries not yet adopted |
| Deployment | Render | Static site + route rewrites to the API |

---

## Repository Layout

Two repositories, one system.

```
HMS_FINAL/                          # this README + product docs
├── Hospital Management System Project Requirements.md   # PRD
└── frontend_api_specification.md                        # API contract

HMS-backend/                         # Go API
├── cmd/
│   ├── server/        # HTTP API entrypoint (config, pool, router, graceful shutdown)
│   ├── worker/        # Background worker entrypoint
│   └── seed/          # Database seeding CLI
├── internal/
│   ├── api/
│   │   ├── handler/   # Gin handlers — translate JSON ↔ domain
│   │   ├── middleware/# JWT auth, RBAC
│   │   ├── response/  # Uniform JSON envelope + error mapping
│   │   └── router.go  # Route groups and middleware pipelines
│   ├── config/        # Environment configuration
│   ├── db/
│   │   ├── migrations/# Goose SQL migrations (schema + constraints + indexes)
│   │   ├── queries/   # sqlc SQL query definitions
│   │   └── *.sql.go   # sqlc-generated, type-safe queries & models
│   ├── domain/        # Models, enums, sentinel errors
│   ├── mailer/        # Email delivery abstraction
│   ├── service/       # Business logic orchestration
│   └── worker/        # Outbox dispatcher, reminder job, no-show sweep
├── plan/              # Technical implementation plan
└── tests/             # Concurrency + integration suites

hms_frontend/                        # React SPA
├── src/
│   ├── api/           # axios client + per-resource API modules
│   ├── components/
│   │   ├── common/    # Button, Card, Input, Modal, Badge, SyncIndicator
│   │   ├── layout/    # Navbar, Sidebar, RoleGuard, MainLayout
│   │   ├── queue/     # HistoryModal, CancelModal
│   │   ├── booking/   # 4-step BookingWizardModal
│   │   └── admin/     # InviteDoctorModal
│   ├── context/       # AuthContext (session + role)
│   ├── pages/         # auth/ receptionist/ doctor/ patient/ admin/
│   ├── types/         # Shared API contract types
│   └── utils/         # date, formatters, error extraction, demo data
├── server.js          # Zero-dependency Node SPA server + API reverse proxy
└── render.yaml        # Render deployment definition
```

---

## Architecture

### Backend layering

Requests flow strictly one way:

```
HTTP request
   ↓
middleware.Auth        # verify HS256 JWT → user_id, role, email into context
   ↓
middleware.RequireRoles# role gate per route group
   ↓
handler                # bind + validate JSON, shape the response envelope
   ↓
service                # business rules, invariants, transactions, audit + outbox writes
   ↓
db (sqlc + pgx)        # parameterized SQL, explicit transactions
   ↓
PostgreSQL
```

The **service** layer owns the invariants. Handlers never call the database directly, and the database is never asked to enforce a business rule that the schema cannot express.

### Frontend layering

```
page (useState + useEffect)
   ↓
src/api/<resource>.ts   # per-resource module, unwraps the {success, data, error} envelope
   ↓
src/api/client.ts       # axios instance: Bearer injection + 401 refresh interceptor
   ↓
backend
```

Every response uses one envelope, so the API modules unwrap once, in one place:

```json
{ "success": true,  "data": { ... } }
{ "success": false, "error": { "code": "SLOT_CONFLICT", "message": "..." } }
```

**Session handling.** The axios response interceptor detects `401 TOKEN_EXPIRED`, calls `POST /auth/refresh` with the stored refresh token, and replays the original request. A single-flight queue means concurrent 401s produce one refresh, not ten. On refresh failure the client clears storage and dispatches an `hms:auth-session-expired` event that `AuthContext` listens for, which drops the user and lets `RoleGuard` redirect to `/login`.

**Role guards.** Every protected route sits behind `<RoleGuard allowedRoles={[...]} />`. A user who reaches a route outside their role is redirected to their own home screen rather than shown an error.

---

## Key Design Decisions

### Database-enforced slot exclusivity

Covered above in [Concurrent booking](#1-concurrent-booking-with-zero-double-booking). The general principle: **when two concurrent requests must not both succeed, express that as a schema constraint, not as application logic.** Application logic can be bypassed by a new code path; a constraint cannot.

### Dynamic slot projection (no slot bloat)

The obvious design pre-generates a `slots` table — every 15 minutes of every doctor for the next year. That is a lot of rows, and every schedule change creates a synchronisation problem.

Instead, slots are **computed on demand**:

1. Load the doctor's recurring weekly office hours for the requested weekday.
2. Slice each shift window into discrete 15-minute intervals in memory.
3. Subtract intervals intersecting `doctor_time_off`.
4. Subtract intervals occupied by an appointment whose status is not `cancelled`.
5. Drop past intervals — except the one currently in progress, which is flagged `is_walk_in_eligible`.

The database stores schedules and bookings; it never stores empty slots.

```go
// internal/service/schedule_service.go — in-memory slicing, no persisted slot rows
for slotStart := shiftStart; slotStart.Add(15*time.Minute).Before(shiftEnd) ||
    slotStart.Add(15*time.Minute).Equal(shiftEnd); slotStart = slotStart.Add(15 * time.Minute) {
    slotEnd := slotStart.Add(15 * time.Minute)
    // ...
}
```

This also makes the two-phase schedule-change flow cheap: `PUT /doctors/:id/office-hours?preview=true` recomputes slot coverage against the proposed schedule and returns the affected bookings **without writing anything**, so the doctor can confirm before any patient is affected.

### Transactional outbox for notifications

Covered above in [Reliable time-based background work](#3-reliable-time-based-background-work).

### Append-only audit trail

Audit entries are inserts only — no update path, no delete path exists in the codebase. Actions recorded include `appointment_booked`, `appointment_cancelled`, `department_created`, `patient_registered`, `appointment_walk_in_booked` and `time_off_created`.

### Invite-based onboarding

Every account starts `invited`. Staff and patients receive a 48-hour token by email and set their own password. The raw token is never stored — only its SHA-256 hash — so a database read cannot be replayed into an account takeover. Two further consequences:

- The admin **never knows anyone's credentials**, so a hostile admin cannot obtain a doctor account to launder chart access through.
- A patient record can exist before a login does — the receptionist registers demographics and the patient claims the account later.

Deactivation, not deletion, is how access ends. A departing doctor loses all access but their authored entries stay in patient charts, and the audit trail stays intact.

### One role per account, fixed at creation

Changing a role means deactivating the account and creating a new one. There is no self-promotion path and no admin-privilege escalation path, because the role is written once at account creation and never updated.

### Appointment lifecycle state machine

The lifecycle is a strict state machine, and every transition is validated server-side and recorded in `appointment_status_events` so the timeline can show *who* moved the appointment and *why*.

```mermaid
stateDiagram-v2
    [*] --> Booked: Receptionist books slot (>= 24h: reminder enqueued)
    [*] --> CheckedIn: Walk-in booking (now inside ongoing 15m slot)

    Booked --> CheckedIn: Receptionist Check-In
    Booked --> Cancelled: Patient (>= 1h prior) or Staff (any time)
    Booked --> NoShow: Auto-marked by background worker (at slot end if not checked in)

    CheckedIn --> Completed: Doctor or Receptionist completes consultation
    CheckedIn --> Cancelled: Staff cancels visit

    NoShow --> CheckedIn: Staff Revert No-Show (override false alarm)

    Completed --> [*]
    Cancelled --> [*]
```

Invariants enforced in `internal/service/appointment_service.go`:

- Slot duration is exactly 15 minutes, aligned to `:00 / :15 / :30 / :45` with zero seconds.
- Booking window is 7 days ahead, with no minimum lead time.
- A slot stays bookable until it ends; a booking made during the current slot is a walk-in and is created already `checked_in`.
- Patients may cancel only 1 hour or more before the slot; staff may cancel at any time.
- Cancelled, completed and no-show appointments cannot be cancelled.

### Two-phase schedule change

A doctor editing their hours can hit `PUT /doctors/:id/office-hours?preview=true` first. The response lists the bookings that would be cancelled, with no writes performed. Only after confirmation does the commit path run, which — in one transaction — replaces the office hours, cancels the conflicting `booked` appointments with `cancellation_reason = 'doctor_schedule_change'`, and enqueues `slot_removed` notifications for the affected patients.

---

## Roles and Permissions

| Data | Patient | Doctor | Receptionist (also nurse) | Admin |
| --- | --- | --- | --- | --- |
| Identity and contact | Own | Patients on their appointments | Read, create (registers patients) | Manage accounts |
| Appointments | View own, cancel own | View own, cancel own | View all, book, cancel | Change appointments |
| Vitals | Read own | Read, for patients on their appointments | Read, write (editable until the consultation starts) | None |
| Clinical data | Read own | Read whole chart, append (after starting a consultation) | None | None |

Two rules that are easy to miss:

- **Patients cannot book.** Booking is performed by the receptionist, who also records vitals at the visit.
- **Admin and receptionist have no path to clinical data.** Their write access stops at demographics, scheduling and vitals.

---

## API Surface

All endpoints are mounted under `/api/v1` and return the uniform envelope. Full request/response contracts, error codes, and TypeScript types live in [`frontend_api_specification.md`](./frontend_api_specification.md).

### Authentication — `/auth`

| Method | Path | Access |
| --- | --- | --- |
| `POST` | `/auth/login` | Public |
| `POST` | `/auth/refresh` | Public |
| `GET` | `/auth/invite/verify?token=` | Public |
| `POST` | `/auth/invite/claim` | Public |
| `POST` | `/auth/logout` | Authenticated |
| `GET` | `/auth/me` | Authenticated |

### Departments — `/departments`

| Method | Path | Access |
| --- | --- | --- |
| `GET` | `/departments` | admin, receptionist, doctor |
| `POST` | `/departments` | admin |
| `PUT` | `/departments/:id` | admin |

### Patients — `/patients`, `/receptionist`

| Method | Path | Access |
| --- | --- | --- |
| `POST` | `/receptionist/patients` | receptionist |
| `GET` | `/patients?query=&page=&limit=` | receptionist, doctor, admin |
| `GET` | `/patients/:id` | admin, doctor, patient (self) |
| `PUT` | `/patients/:id` | receptionist, patient (self) |

### Doctors and schedules — `/doctors`

| Method | Path | Access |
| --- | --- | --- |
| `GET` | `/doctors?department_id=` | Any authenticated |
| `GET` | `/doctors/:id/schedule` | Any authenticated |
| `GET` | `/doctors/:id/available-slots?date=` | receptionist, doctor, patient, admin |
| `PUT` | `/doctors/:id/office-hours?preview=` | doctor (self), admin (any) |
| `POST` | `/doctors/:id/time-off` | doctor (self), admin (any) |

### Appointments — `/appointments`

| Method | Path | Access |
| --- | --- | --- |
| `POST` | `/appointments` | receptionist only |
| `GET` | `/appointments` | Authenticated (auto-scoped by role) |
| `GET` | `/appointments/:id` | Authenticated (scoped) |
| `POST` | `/appointments/:id/check-in` | receptionist |
| `POST` | `/appointments/:id/complete` | doctor (assigned), receptionist, admin |
| `POST` | `/appointments/:id/cancel` | patient (self, ≥ 1h prior), doctor, receptionist, admin |
| `POST` | `/appointments/:id/revert-no-show` | receptionist, doctor (assigned), admin |
| `GET` | `/appointments/:id/history` | Authenticated (scoped) |

Listing is scoped automatically: a doctor sees only their own appointments, a patient only their own, and staff see the whole hospital.

### Response conventions

- Timestamps: RFC3339 in UTC. Dates: `YYYY-MM-DD`. Times: `HH:MM` 24-hour.
- Weekdays: `0` = Sunday … `6` = Saturday (matches Go's `time.Weekday`).
- Errors carry a stable machine-readable `code` (`SLOT_CONFLICT`, `CANCELLATION_CUTOFF_EXCEEDED`, `INVALID_STATUS_TRANSITION`, …) alongside a human-readable message.

---

## Getting Started

### Prerequisites

- Go 1.27+
- PostgreSQL 14+ with the `btree_gist` extension available
- [goose](https://github.com/pressly/goose) for migrations
- [sqlc](https://sqlc.dev) for query generation (only needed if you change the SQL)
- Node.js 18+ and npm

### 1. Database

```bash
createdb hms
psql -d hms -c "CREATE EXTENSION IF NOT EXISTS btree_gist;"
```

### 2. Backend

```bash
git clone https://github.com/HarmanSingh16/HMS-backend.git
cd HMS-backend
```

Create `.env` in the repository root:

```bash
DATABASE_URL=postgres://postgres:postgres@localhost:5432/hms
PORT=8080
JWT_SECRET=change-me-in-production
ACCESS_TOKEN_TTL=15m
REFRESH_TOKEN_TTL=168h
INVITE_TOKEN_TTL=48h

GOOSE_DRIVER=postgres
GOOSE_DBSTRING=postgres://postgres:postgres@localhost:5432/hms
GOOSE_MIGRATION_DIR=internal/db/migrations
```

Run the migrations, then start the API:

```bash
goose -dir internal/db/migrations postgres "$DATABASE_URL" up
go run ./cmd/server
```

The API is now on `http://localhost:8080` — health check at `GET /health` returns `{"status": "ok"}`.

Seed demo data (idempotent; it resets non-admin data first):

```bash
go run ./cmd/seed
```

Useful `Makefile` targets:

```bash
make migrate-up        # apply migrations
make migrate-status    # show applied/pending migrations
make sqlc              # regenerate type-safe queries after editing SQL
make build             # go build -v ./...
make vet               # go vet ./...
```

### 3. Frontend

```bash
git clone https://github.com/HarmanSingh16/hms_frontend.git
cd hms_frontend
npm install
```

Create `.env`:

```bash
VITE_API_BASE_URL=/api/v1
```

Because the base URL is relative, the Vite dev server proxies `/api` and `/health` to the backend and no CORS configuration is needed during development. Point the proxy at your local backend in `vite.config.ts`:

```ts
const BACKEND_URL = 'http://localhost:8080';
```

Then run:

```bash
npm run dev        # Vite dev server on http://localhost:5173
npm run build      # type-check (tsc) + production build to dist/
npm run preview    # preview the production build
npm start          # serve dist/ with SPA fallback + API proxy (node server.js)
```

### 4. Walk through it

Open `http://localhost:5173/login`. The login page exposes one-click demo role buttons for the seeded accounts.

| Role | Lands on |
| --- | --- |
| Receptionist | Front Desk Queue — check in, revert no-show, cancel, register patients, book |
| Doctor | Clinical Waiting Room — complete consultations, manage office hours and leave |
| Patient | My Appointments — view, self-cancel (with the 1-hour cutoff enforced in the UI) |
| Admin | Departments — department CRUD, hospital-wide appointment monitor, doctor schedules |

---

## Seeded Accounts

The seed script populates the database with a realistic dataset: 12 departments, 12 doctors, 3 receptionists, 12 patients, invites, office hours, leave, appointments, status events, audit logs and outbox notifications.

All seeded accounts share the password **`Password123!`**.

| Role | Email | Details |
| --- | --- | --- |
| Admin | `admin@hospital.com` | Root administrator |
| Receptionist | `reception.clara@hospital.com` | Front desk (active) |
| Receptionist | `reception.mark@hospital.com` | Front desk (active) |
| Receptionist | `reception.elena@hospital.com` | Invited — claim flow demo |
| Doctor | `dr.sarah.connor@hospital.com` | Cardiology |
| Doctor | `dr.john.watson@hospital.com` | Neurology |
| Doctor | `dr.gregory.house@hospital.com` | Orthopedics |
| Doctor | `dr.meredith.grey@hospital.com` | Pediatrics |
| Patient | `alice.smith@example.com` | Active |
| Patient | `bob.jones@example.com` | Active |
| Patient | `charlie.brown@example.com` | Active |

Demo invite tokens for testing the claim flow:

- Receptionist: `receptionist_elena_invite_token_123`
- Patient: `patient_invite_token_sample_11_Kevin`

---

## Project Status

Honest accounting of what is built and what is not.

### Implemented

- **Auth** — bcrypt password hashing, JWT access/refresh pairs, invite verify + claim, role-scoped `GET /auth/me`.
- **Departments** — admin CRUD with unique-name conflict handling (`409 DEPARTMENT_NAME_TAKEN`).
- **Patients** — receptionist intake with invite generation, paginated search, self-service demographic updates.
- **Doctor schedules** — recurring weekly office hours with multi-interval support, time-off / leave, and the **dynamic 15-minute slot projection engine**.
- **Appointments** — concurrent-safe booking with the GiST exclusion constraint, role-scoped listing, check-in, complete, cancel (with the patient 1-hour cutoff), revert-no-show, and the transition timeline.
- **Transactional outbox writes** — reminders enqueued on booking, revoked on cancellation, `slot_removed` enqueued on schedule changes.
- **Audit trail writes** — booking, cancellation, department changes, patient registration, time-off.
- **Frontend** — all four role dashboards, the 4-step booking wizard, schedule management with the preview-conflict modal, the receptionist queue, the doctor waiting room, the patient portal with self-cancellation, invite claim, and the login flow.
- **Schema** — 4 Goose migrations, sqlc-generated queries, indexes on every hot path.

### Not yet implemented

| Area | State |
| --- | --- |
| Background worker daemon | `cmd/worker` and `internal/worker/*` are stubs — the outbox is written correctly but **nothing dispatches it yet** |
| SMTP / mailer delivery | `internal/mailer/*` are stubs — no email is actually sent |
| No-show auto-marking | Requires the worker sweep; the `revert-no-show` endpoint and the `no_show` state exist and work |
| Concurrency test | `tests/concurrency/booking_concurrency_test.go` is a stub — the constraint is in place and proven by the schema, but the 50-goroutine test is not written |
| Audit query endpoints | Audit rows are written; there is no read API for them yet |
| Patient-visible access log | Not implemented |
| Admin staff invites / account deactivation | Endpoints planned (`/admin/invites`, `/admin/accounts/{id}/deactivate`) but not mounted |
| Clinical records, vitals, prescriptions | Phase 2 — see below |
| CORS middleware | Not present; the frontend uses a same-origin dev/prod proxy instead |

The backend compiles clean (`go build ./...`, `go vet ./...`) and the frontend type-checks clean (`tsc` in `npm run build`).

---

## Roadmap

**Phase 2 — clinical data.** This is the largest remaining piece and the one that makes the authorization model meaningful:

- Append-only medical records, one per consultation, created when the doctor starts it
- Chart access granted by starting a consultation, with whole-chart read for that patient thereafter
- Amendments that reference the original entry; entries are never edited or deleted
- Structured prescriptions (drug, dosage, frequency, duration), downloadable as PDF
- Vitals (BP, pulse, temperature, weight, SpO₂) tied to an appointment, editable by reception until the consultation starts and locked afterwards
- Patient-facing visit timeline

**Also planned:**

- The background worker: outbox dispatcher with `SKIP LOCKED` polling, retry with exponential backoff, SMTP mailer, and the no-show sweep
- Concurrency test suite (`50` goroutines → exactly `1`× `201`, `49`× `409`) and lifecycle/outbox integration tests
- Admin governance endpoints: staff invites, account listing, deactivation
- Audit log read API and the patient-visible access log
- CORS middleware for direct cross-origin deployments

**Out of scope for this version** (by design): billing, insurance and payments; pharmacy and drug databases; lab results and imaging; inpatient beds and admissions; multiple hospitals or branches; patient self-booking; any email beyond the three listed.

---

## Documentation

| Document | Contents |
| --- | --- |
| [`Hospital Management System Project Requirements.md`](./Hospital%20Management%20System%20Project%20Requirements.md) | The PRD — goals, roles, permissions, lifecycle, notification rules, threat model, assumptions |
| [`frontend_api_specification.md`](./frontend_api_specification.md) | The API contract — every endpoint, request/response shape, error code, lifecycle diagram, TypeScript types, and seven annotated user journeys |
| [`plan/technical_implementation_plan.md`](https://github.com/HarmanSingh16/HMS-backend/blob/main/plan/technical_implementation_plan.md) | Architecture, folder structure, and the reasoning behind each engineering challenge |
| `docs/hms_schema_full_er_diagram_readable_headers.html` | Full ER diagram of the database schema |

---

## Repositories

| | |
| --- | --- |
| Backend | https://github.com/HarmanSingh16/HMS-backend |
| Frontend | https://github.com/HarmanSingh16/hms_frontend |

---

## License

Personal portfolio project. All rights reserved.
