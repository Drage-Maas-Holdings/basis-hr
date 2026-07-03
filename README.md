# Basis HR

A lean, self-hosted HR module for small businesses, built on the same stack as Basis CRM. Ships as its own service with its own database, sharing only identity and authentication with CRM via a standalone Better Auth service.

---

## What it is

Basis HR is a single-tenant HR system covering the essentials: employee records, attendance, leave, payroll basics, expense claims, performance tracking, recruitment, and compliance reporting. It is not a SaaS product. It runs on infrastructure you control, stores data in its own SQLite file, and exposes a REST API for integration with other internal tools.

Core capabilities (MVP scope, build order below):

1. Employee Records — profiles, contracts, documents
2. Attendance & Leave — check-in/out, leave requests, holiday calendars
3. Payroll Basics — payslip generation, regional tax compliance
4. Expense Claims — lightweight reimbursement workflow
5. Performance Tracking — goal-setting, periodic reviews
6. Recruitment Lite — job postings, candidate tracking
7. Compliance & Reports — statutory reports, audit-ready exports

Deferred: complex shift scheduling, biometric integrations, advanced performance management, deep ERP integrations, mobile app.

---

## Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 22 |
| Framework | Hono |
| Auth | Better Auth (standalone service, shared with CRM) |
| ORM | Drizzle |
| Database | SQLite (better-sqlite3), separate file from CRM |
| Containerization | Docker Compose |

This matches the CRM stack exactly, so both services share engineering patterns, migration tooling, and API conventions. The only shared infrastructure is authentication.

---

## Authentication architecture

Better Auth runs as its own service, not embedded in either CRM or HR. Both CRM and HR authenticate against it over the network and maintain no local `users` table of their own beyond a foreign reference (e.g. `owner_id`) to the identity service's user ID.

This is a deliberate deviation from the CRM's current implementation, where Better Auth is embedded directly in the CRM API. See the corresponding entry in the CRM's `improvements.md` for the migration rationale.

Reasoning:

- One login across CRM and HR, without merging product data
- Session/token validation logic exists once, not duplicated per service
- Either service can be deployed, scaled, or taken down independently without affecting login for the other
- New modules built later authenticate against the same identity service without re-implementing auth

This does introduce a dependency: if the auth service is down, both CRM and HR are unable to authenticate new sessions. This tradeoff is accepted because the alternative (duplicated auth logic per service) is worse for an enterprise-facing product, and the auth service is the smallest, most stable piece of the system to keep highly available.

---

## Database ownership

HR uses its own SQLite database, entirely separate from the CRM's. Neither service queries the other's tables directly. Cross-module data (e.g. showing HR status on a CRM contact) is retrieved via API calls between services, not shared schema.

This is intentional, not a placeholder for a future merge:

- Independent migration lifecycles — HR schema changes cannot break CRM and vice versa
- Independent backup and restore — restoring one service's data does not touch the other
- No hidden coupling from convenient direct queries across module boundaries
- SQLite is single-writer; separate files avoid lock contention between unrelated services

---

## Build order

Build order matters here because later modules depend on schemas established earlier. Building out of order risks migrating live data mid-project.

1. **Auth integration first** — connect to the standalone Better Auth service before any HR-specific schema work. Every other collection references a user ID from this service.
2. **Employee Records** — the foundational entity. Attendance, Payroll, Expenses, Performance, and Recruitment all link back to an employee record.
3. **Attendance & Leave** — required before Payroll, since payroll calculations consume attendance/leave data as input.
4. **Payroll Basics** — depends on Attendance & Leave. Keep this and Expense Claims on consistent financial data models before either is built out further.
5. **Expense Claims** — second financial workflow; reuse patterns established by Payroll rather than diverging.
6. **Performance Tracking** and **Recruitment Lite** — both depend only on Employee Records, can be built in either order once step 2 is stable.
7. **Compliance & Reports** — last. Reads from all prior collections, so it should be built once those schemas are stable, not before.

Do not begin Recruitment Lite or Performance Tracking before Employee Records is finalized. Both depend on the same employee schema, and reworking that schema after either module exists means a live data migration.

---

## Monorepo structure

```
hr/
  api/        # Hono REST API
  web/        # Web GUI
  docker-compose.yml
```

Mirrors the CRM's structure for consistency across services.

---

## Getting started

Prerequisites: Docker and Docker Compose, Node.js 22+, and a running instance of the standalone Better Auth service.

```bash
git clone <hr-repo-url>
cd hr
cp api/.env.example api/.env
docker compose up
```

`api/.env` must point `BETTER_AUTH_URL` at the standalone auth service, not a local instance.

---

## Status

This is a full restart of the module previously scoped as an Employee Management System on MERN with a planned Web3/Solidity component. That direction has been dropped. This module is scoped from a cofounder proposal for a lean HR system and built to match the CRM's actual stack (Hono/Drizzle/SQLite/Docker), not the earlier PocketBase description. No code has been ported from the prior version.
