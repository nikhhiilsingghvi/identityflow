# IdentityFlow — Identity Lifecycle Automation

Automatically provisions, updates and revokes employee and contractor access across downstream
applications in response to HR events (joiner / mover / leaver), with approvals, retries, idempotency,
failure recovery, RBAC and a tamper-evident audit trail.

```
HR system ──signed JSON──▶ IdentityFlow API ──▶ PostgreSQL (events, workflows, task queue, audit)
(hr-sim :9100)             (FastAPI :8000)            ▲
                                  │                   │ leases tasks (SKIP LOCKED)
                          React console :8080     Worker(s) ──SCIM 2.0──▶ Collab Hub  :9001
                                                                    └───▶ ServiceDesk :9002
```

## Quick start

```bash
cp .env.example .env            # change secrets for anything beyond a demo
docker compose up --build -d
open http://localhost:8080      # console   (admin/admin-pass, approver/approver-pass,
                                #            operator/operator-pass, auditor/auditor-pass)
open http://localhost:8000/docs # OpenAPI / Swagger
pip install httpx && python scripts/demo.py   # scripted run of all six scenarios
```

Tests (no Docker needed; uses in-memory SQLite and in-process simulated apps):

```bash
cd backend && pip install -r requirements.txt && python -m pytest -v
```

## The six demonstration cases

| # | Scenario | Where it is proven |
|---|----------|--------------------|
| 1 | Contractor joins → access to two apps | `test_1_…`, demo step 1 |
| 2 | Department change → permissions updated in place | `test_2_…`, demo step 2 |
| 3 | Contractor leaves → revoked in both apps | `test_3_…`, demo step 3 |
| 4 | One app down during offboarding → retries, failure recorded, **never reported as success** | `test_4_…`, demo step 4 |
| 5 | Replayed event → no duplicate accounts | `test_5_…`, `test_5b_…`, demo step 5 |
| 6 | Unauthorized user cannot approve/execute privileged changes | `test_6_…`, demo step 6 |

## How it works

**Event ingestion** (`POST /api/v1/events/hr`). The HR system signs `"<timestamp>.<raw body>"` with
HMAC-SHA256 (`X-HR-Signature`, `X-HR-Timestamp`, ±5 min tolerance). Each event carries an
`idempotency_key`; a unique constraint guarantees one workflow per key, so redelivery returns the
original workflow (`duplicate: true`). Events that are valid but impossible in the current state
(mover for an unknown worker, joiner for an active worker) return 409 and are not stored, so HR can
redeliver after fixing its data.

**Planning.** Access rules (`department × worker_type → app role`, optionally *privileged*) produce a
desired state. Joiner → `CREATE` tasks. Mover → diff against confirmed accounts: `CREATE` / `UPDATE` /
`DEACTIVATE`. Leaver → `DEACTIVATE` everywhere (including anything half-created by an in-flight
workflow, which is cancelled). Leavers never wait for approval.

**Approvals.** If a joiner or mover would *newly* receive a privileged role, the workflow is
`PENDING_APPROVAL` and its tasks are `BLOCKED` — nothing is sent downstream. Only the `approver` role may
approve/reject; requesters cannot approve their own requests; denied attempts are audited.

**Execution.** Tasks are a durable queue in PostgreSQL. Workers lease due tasks with
`FOR UPDATE SKIP LOCKED` (leases expire, so a crashed worker's task is picked up again). The SCIM
client upserts by `externalId` (lookup → PUT, else POST, 409 → re-lookup), so repeating a task can't
create duplicates even if local state is lost. Transport errors, 408/425/429 and 5xx are retried with
exponential backoff + jitter up to `MAX_ATTEMPTS`; other 4xx fail immediately. Exhausted tasks become
`FAILED` and the workflow becomes `FAILED`.

**Honest state.** `Account` rows are written only after the downstream confirms. An employee becomes
`ACTIVE` / `TERMINATED` only when *every* task succeeded; otherwise a leaver stays `OFFBOARDING` with
a failed workflow on the dashboard. Operators re-queue with `POST /sync-tasks/{id}/retry` (or
`/sync-tasks/retry-failed`).

**Audit.** Every state change appends an `audit_events` row whose SHA-256 covers its content and the
previous row's hash. Updates/deletes are blocked by an ORM guard and (on PostgreSQL) a trigger that
also blocks `TRUNCATE`. `GET /audit/verify` recomputes the chain; `GET /audit/export` produces CSV or
JSON including hashes so an auditor can re-verify offline.

**Field mapping.** HR field → SCIM path (`name/givenName`, `emails/0/value`, `enterprise/department`)
with `none|lower|upper|title` transforms, editable per application in the console, validated
(`userName` required, reserved attributes protected), with a payload preview.

## Roles

| Role | Can |
|------|-----|
| viewer (`auditor`) | read dashboards, audit export |
| operator | viewer + retry failed tasks, run worker |
| approver | viewer + approve/reject privileged workflows |
| admin | viewer + operator + edit mappings/rules + control simulators (**cannot approve**) |

## Project layout

```
backend/app/        config, db, models, engine (JML + approvals), executor (queue, retries),
                    scim_client, mapping, audit, security, routers/
backend/sim_apps/   simulated SCIM 2.0 apps with outage / failure injection
backend/hr_sim/     simulated HR system (signs and sends events, can replay them)
backend/tests/      pytest scenarios
frontend/           React (Vite) console, served by nginx in Docker
scripts/demo.py     scripted demonstration against the running stack
```

## Deliberate scope decisions

* **No Celery/Redis.** The Postgres-backed queue gives durable, multi-worker, crash-safe execution with one
  less moving part. Swap in Celery by having `executor.process_task` called from a Celery task.
* **Tables via `create_all`, not Alembic.** Fine for a demo; add Alembic migrations before evolving a live schema.
* **Simulated apps hold users in memory** (reset on restart).

## Before a real deployment

Set `ENV=production` and strong `SECRET_KEY`, `HR_WEBHOOK_SECRET`, DB and app tokens; put TLS in front;
add login rate-limiting/lockout, SSO for operators, secret storage (Vault/KMS) for downstream tokens,
Alembic migrations, metrics/alerting on `FAILED` tasks, and DB role separation so the app role cannot
disable the audit trigger.
