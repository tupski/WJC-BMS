# Architecture — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed — no code exists |
| Version | 0.1 |
| Scope owner | This document owns system structure, module boundaries, and technology recommendation |
| Requirement source | [`../PRD.md`](../PRD.md) |
| Decision records | [`../decisions/`](../decisions/) |

---

## 1. Architectural basis and honesty statement

The workspace was verified empty before authoring: zero files including hidden entries, no `.git`, no existing `AGENTS.md`, no `docs/`. There is **no established technology stack, no codebase, no build or test commands, and no existing conventions.**

Everything below is therefore **[Proposed]**. Nothing here has been installed, scaffolded, benchmarked, or verified. The recommendation is a planning position intended to be ratified by an ADR before Phase 1 begins.

**[Confirmed]** constraints that shape the design:

- ~3 office employees, ~15 technicians — a small user population.
- Operations span administration, purchasing, suppliers, project execution, field technicians, invoicing, and financial administration.
- Administration currently runs on spreadsheets, documents, and WhatsApp; there is no centralised inventory.
- Projects span multiple days; teams range from one technician to several.
- An invoice-numbering conflict has occurred; the cause is unconfirmed.

These constraints argue strongly against distributed systems. A team of this size cannot operate microservices, a message broker, or an event-sourced platform without dedicating a disproportionate share of its capacity to infrastructure.

---

## 2. Architectural style: modular monolith

**[Proposed]** A single deployable application with **enforced module boundaries**, one relational database, and one private object store.

### 2.1 Why

| Consideration | Modular monolith | Microservices |
|---|---|---|
| Operational burden for ~3 office staff and no dedicated ops engineer | One deploy, one database, one log stream | Multiple deploys, service discovery, distributed tracing, network failure modes |
| Transactional integrity for numbering, reservations, approvals | Native ACID across all modules | Requires sagas/outbox; correctness becomes a distributed problem |
| Team size | Suitable | Requires more engineers than exist |
| Evolution path | Modules can be extracted later if a boundary proves genuinely independent | Extraction backwards is expensive |

The decisive argument is transactional integrity. The two highest-risk requirements — atomic document-number allocation (FR-DOC-004) and reservation correctness across projects (FR-MATL-006) — are trivially correct inside one database transaction and materially harder across service boundaries. Choosing distribution here would create the exact class of bug the business is trying to eliminate.

**Consequence:** module boundaries must be real, not nominal. Enforced by (a) module-private data access, (b) a rule that cross-module reads go through a module's public interface, and (c) an automated dependency check in CI that fails on illegal imports. Without enforcement, a modular monolith degrades into a monolith with folders.

### 2.2 Layers

```
┌──────────────────────────────────────────────────────────────┐
│ Presentation      Server-rendered pages / API handlers        │
│                   Indonesian UI, mobile-first responsive      │
├──────────────────────────────────────────────────────────────┤
│ Application       Use cases: commands and queries             │
│                   Transaction boundary · authorisation check  │
│                   Audit write · domain event dispatch         │
├──────────────────────────────────────────────────────────────┤
│ Domain            Entities, value objects, state machines     │
│                   Business rules (BR-*) — no I/O here         │
├──────────────────────────────────────────────────────────────┤
│ Infrastructure    Repositories, ORM, object storage, PDF,     │
│                   mail/SMS (none in MVP), clock, ID gen       │
└──────────────────────────────────────────────────────────────┘
                    │                          │
        ┌───────────▼──────────┐   ┌───────────▼────────────┐
        │ PostgreSQL           │   │ S3-compatible storage  │
        │ single database      │   │ private bucket         │
        └──────────────────────┘   └────────────────────────┘
```

**Rules:**

1. Dependencies point downward only. Domain never imports infrastructure.
2. A use case owns exactly one transaction. Multi-step state changes either commit fully or not at all (NFR-DATA-001).
3. Authorisation is checked in the application layer before any write, and again at the data-access boundary for row-scoped entities (defence in depth). Hiding a button is not authorisation (NFR-SEC-002).
4. Audit records are written inside the same transaction as the state change they describe (FR-AUD-003). This is the single most important structural rule in the system.
5. Domain logic does not read the clock or generate IDs directly — both are injected, so time-dependent rules (overdue, number year/month) are testable.

### 2.3 Module boundaries

| Module | Owns | Public interface (summary) | Depends on |
|---|---|---|---|
| `identity` | Users, roles, sessions, permission checks | `can(user, action, resource)`, `currentUser()` | — |
| `audit` | Audit events, hash chain, audit queries | `record(event)`, `query(filter)` | `identity` (actor ref only) |
| `numbering` | Document series, counters, allocations | `allocate(seriesKey, ctx)` | — |
| `customer` | Customers, contacts, service categories | `getCustomer(id)`, `listFor(user)` | `identity` |
| `catalog` | Item/material catalogue, units | `getItem(id)`, `search()` | — |
| `supplier` | Suppliers, terms | `getSupplier(id)` | `identity` |
| `project` | Projects, assignments, schedules, project state machine | `createProject`, `transition`, `closeProject` | `customer`, `identity` |
| `workorder` | Work orders, daily reports, evidence, verification | `assign`, `submitReport`, `verify` | `project`, `identity` |
| `material` | Requirements, stock status, reservations, movement ledger, issues/returns | `reserve`, `issue`, `consume`, `return`, `reconcile` | `project`, `catalog` |
| `procurement` | Purchase requests, approvals, POs, goods receipts | `submitRequest`, `decide`, `issuePo`, `receive` | `material`, `supplier`, `numbering` |
| `finance` | Quotations, invoices, payments, payables, expenses, allocations | `issueInvoice`, `recordPayment`, `confirm` | `project`, `customer`, `numbering` |
| `document` | File storage, attachments, PDF generation, document registry | `store`, `attach`, `render`, `download` | `identity` |
| `reporting` | Read models and exports | `dailyProgress`, `projectDashboard`, `ageing` | read-only across modules |

**Boundary rules:**

- `material` never mutates `procurement` records and vice versa; they exchange explicit intents. A goods receipt in `procurement` publishes a receipt fact that `material` consumes to create an inbound movement. Both happen in one transaction.
- `finance` never reads `material` tables directly. It reads project cost through the `project` module's cost interface, which aggregates recognised cost sources (BR-FIN-005).
- `reporting` is read-only and may query views. It must never be the source of a business decision.
- `numbering` has no dependencies. It is the most isolated module and must stay that way.

---

## 3. Technology recommendation

**[Proposed]** No stack exists. Three options were compared briefly; the recommendation is recorded as [`../decisions/ADR-001-modular-monolith-and-stack.md`](../decisions/ADR-001-modular-monolith-and-stack.md).

| Criterion | Option A: TypeScript full-stack monolith | Option B: Laravel (PHP) monolith | Option C: Django (Python) monolith |
|---|---|---|---|
| Fit to requirements | Strong: one language across server and UI, typed end-to-end, mature ORM with migration tooling | Strong: batteries-included, fast CRUD, familiar in Indonesian SME context | Strong: excellent admin, mature ORM |
| Transactional control for numbering | Excellent: explicit transaction control, `UPDATE … RETURNING` available via raw SQL when needed | Good | Good |
| Typed contracts between modules | Strong (shared types, compile-time boundary checks) | Moderate | Moderate (type hints, not enforced) |
| Server-rendered mobile-first UI | Good | Very good (Blade) | Good (templates) |
| PDF generation | Good | Very good | Good |
| Long-term maintainability by a small team | Good if boundaries enforced | Good | Good |
| Hiring availability locally | High | High | Moderate |
| Risk | Ecosystem churn; requires discipline to avoid over-abstraction | Framework version upgrades; weaker static typing | Weaker static typing |

**Recommendation:** Option A — TypeScript full-stack modular monolith with **PostgreSQL**, a typed ORM with explicit migration control, and **S3-compatible private object storage**. Server-rendered pages with progressive enhancement for the technician flows, rather than a heavy client-side SPA, because the technician experience must work on modest phones over weak mobile data (NFR-OPS-003, NFR-OPS-004).

**Why PostgreSQL specifically:** it provides the exact primitives this design depends on — `UPDATE … RETURNING` for atomic sequence allocation, real `SERIALIZABLE` isolation, `CHECK` constraints, partial unique indexes, row-level locking, and transactional DDL for safe migrations.

**This is a recommendation, not an installation.** No `package.json`, lockfile, config, or source directory exists or will be created in this documentation phase.

**Explicitly rejected for MVP:** microservices, event sourcing as the system of record, CQRS with separate read/write stores, message brokers, Kubernetes, GraphQL, client-side document-number generation.

---

## 4. Authentication and authorisation

Detail in [`./SECURITY_AND_ACCESS_CONTROL.md`](./SECURITY_AND_ACCESS_CONTROL.md). Architectural summary:

- **Authentication:** email/username + password with a slow salted hash (Argon2id or bcrypt with a documented cost factor), server-side sessions with idle expiry, session invalidation on password change. MFA is **[Out of Scope]** for MVP but the session model must not preclude it.
- **Authorisation:** a central policy function evaluated per `(user, action, resource)`. Permissions are data (roles → permissions), not `if role === 'owner'` scattered through code.
- **Row scoping:** technician access is filtered at the query layer by assignment, not by post-filtering results in the UI. A query that could return another technician's rows must be structurally incapable of doing so.
- **Enforcement points:** application layer (authorisation), data layer (scoped repository), and database (constraints, and optional row-level security as a second line for the most sensitive tables).

---

## 5. Data integrity and transaction boundaries

| Concern | Mechanism |
|---|---|
| Document number uniqueness | Single-row `UPDATE … RETURNING` on a series counter + `UNIQUE` constraint on `(series_key, allocated_number)` (ADR-002) |
| Reservation correctness | `SERIALIZABLE` isolation or explicit `SELECT … FOR UPDATE` on the stock balance row for the item; reservation insert and balance update in one transaction |
| Approval → PO | Approval row and PO creation in one transaction; PO cannot reference an unapproved request (constraint + application check) |
| Goods receipt → stock | Receipt lines and inbound movements written atomically; a receipt without movements is impossible |
| Invoice issuance | Number allocation, snapshot write, PDF record, and audit row in one transaction |
| Payment allocation | Allocation sum ≤ invoice total enforced by a database constraint on the aggregate (checked within the transaction under lock) |
| Audit coupling | Audit insert in the same transaction; a trigger or equivalent guarantees no state transition commits without its audit row (ADR-004) |
| Issued document immutability | Snapshot tables are insert-only; update/delete revoked at the database privilege level |

**Concurrency policy:** the system is small but concurrent — three office staff issuing invoices simultaneously is exactly the scenario that produced the historical conflict. All allocation and balance-mutating operations must be proven under parallel load (AC-DATA-002, AC-MATL-006), not merely tested sequentially.

---

## 6. Key relationship shapes

Full entity detail is in [`./DATA_MODEL.md`](./DATA_MODEL.md). The structurally significant shapes:

```
Customer 1──* Contact
Customer 1──* Project
Project  1──* MaterialRequirementLine ──1 Item
Project  1──* WorkOrder ──1 Technician
WorkOrder 1──* DailyReport ──* Attachment
Project  1──* Document (quotation, invoice, …)

MaterialRequirementLine 1──* MaterialMovement   (append-only ledger)
MaterialRequirementLine 0..1 Reservation
MaterialRequirementLine 0..* PurchaseRequestLine
PurchaseRequestLine *──1 PurchaseRequest ──0..1 ApprovalDecision
PurchaseRequest 0..1 PurchaseOrder ──* GoodsReceipt
PurchaseOrder 1──* Payable ──* PaymentAllocation ──1 Payment

Invoice 1──* PaymentAllocation
Invoice 1──1 IssuedDocumentSnapshot   (immutable)
```

Two shapes carry most of the business correctness:

1. **`MaterialMovement` is the only way quantity changes.** There is no writable `quantity` column that a user can edit. Balance is derived, with a periodically reconciled materialised value for performance (FR-MATL-012, FR-MATL-013).
2. **`IssuedDocumentSnapshot` is written once and never updated.** Every issued document renders from its snapshot, not from live master data (FR-FIN-003, BR-DOC-001).

---

## 7. Attachments and private file storage

- Files go to a **private** bucket. No object is publicly readable. There is no permanent public URL anywhere in the system (FR-DOC-001).
- Access is via short-lived signed URLs (minutes, not hours) issued only after a permission check on the owning record. A signed URL is a capability, not an authorisation bypass — the check happens before signing.
- Object keys are opaque and non-guessable; they do not encode customer names, invoice numbers, or any business meaning.
- Every attachment records: owning entity type and ID, uploader, upload timestamp, content type, byte size, and a content hash.
- Uploads are validated server-side (content type sniffing, size limit, extension allow-list). Client-side checks are convenience only.
- Deletion of a file referenced by an issued document is prohibited. Soft-delete and retention apply (BR-DATA-002).
- **File recovery:** bucket versioning enabled, plus inclusion in the backup regime (NFR-DATA-004).

---

## 8. Audit architecture

Detail in [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md).

- Append-only table; `UPDATE` and `DELETE` privileges revoked from the application role.
- Written in the same transaction as the state change (FR-AUD-003). There is no asynchronous "audit later" path, because a crash between commit and audit write would create an unattributable change.
- Each record captures actor, action, entity type/ID, before/after values for changed fields, timestamp, correlation ID, and source IP where available.
- Hash chaining over issued-document and approval events provides tamper evidence (NFR-AUD-003).
- Audit writes must never contain secrets, password hashes, or full file contents.

---

## 9. Deployment and environments

**[Proposed]**

| Environment | Purpose | Data |
|---|---|---|
| Development | Local work | Synthetic fixtures only |
| Staging | Pre-release verification, UAT, migration rehearsal | Anonymised copy or synthetic; never live personal data |
| Production | Live business operations | Real data, access-restricted |

- Single-region deployment near Indonesia to keep technician latency acceptable (NFR-OPS-004).
- Production database not reachable from the public internet; application connects over a private network.
- Migrations run as an explicit release step, forward-only, rehearsed on staging (NFR-DATA-006). No automatic destructive migration on deploy.
- Configuration via environment variables; secrets in a managed secret store (NFR-SEC-007). No secret in the repository, ever.
- Health endpoint for liveness; structured logs shipped to a retained log store (NFR-OPS-002).
- Rollback strategy: application rollback by redeploying the previous artefact; database changes must be backward-compatible for at least one release (expand → migrate → contract).

**[Open Question] Q-OPS-01** (from PRD §12) — hosting provider, backup location, retention period, and access policy remain unconfirmed. Defaults proposed in the PRD keep planning unblocked.

---

## 10. Data migration and operational maintenance

- Migration is a **product feature**, not a one-off script (FR-DATA-001…006): validation dry-run, explicit commit, per-row rejection report, verbatim preservation of historical numbers.
- Imported historical numbers are stored in a reserved historical range so they cannot collide with newly allocated numbers (FR-DATA-004).
- Operational maintenance tasks: daily reconciliation report (FR-MATL-013), audit chain verification, backup verification, overdue ageing refresh, orphaned-file scan.
- All maintenance tasks must be idempotent and safe to re-run.

---

## 11. What this architecture deliberately does not do

| Not done | Reason |
|---|---|
| Microservices | No operational capacity; would break transactional guarantees that the highest-risk requirements depend on |
| Event sourcing as system of record | Audit log already provides history; event sourcing would add reconstruction complexity for no business gain |
| Full warehouse model (bins, lots, serials) | Project-scoped control suffices (Q-INV-01 defines the trigger to expand) |
| General ledger | Requires accountant input; L3 is out of MVP scope (Q-FIN-01) |
| Native mobile app | Mobile-first responsive web covers the need (Q-UX-01) |
| Real-time sync / offline-first | Not justified by observed need; would dominate the build |
| Client-side document numbering | Directly contradicts FR-DOC-004 and the confirmed failure mode |

---

## 12. Cross-references

- Requirements and status vocabulary: [`../PRD.md`](../PRD.md)
- Entities and constraints: [`./DATA_MODEL.md`](./DATA_MODEL.md)
- Permissions and overrides: [`./SECURITY_AND_ACCESS_CONTROL.md`](./SECURITY_AND_ACCESS_CONTROL.md)
- Numbering mechanism: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Audit model: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
- Phasing: [`../implementation/IMPLEMENTATION_PLAN.md`](../implementation/IMPLEMENTATION_PLAN.md)
