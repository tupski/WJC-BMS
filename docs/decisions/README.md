# Architecture Decision Records

One decision per record. Each states its context, decision, consequences, alternatives considered, status, and verification obligations.

**All records are `Proposed`.** None has been ratified by the business, and none has been implemented. There is no code in this repository.

## Index

| ADR | Title | Decision in one line | Status |
|---|---|---|---|
| [ADR-001](./ADR-001-modular-monolith-and-stack.md) | Modular monolith architecture and technology stack | One deployable application, enforced module boundaries, TypeScript + PostgreSQL + private object storage | Proposed |
| [ADR-002](./ADR-002-document-number-allocation.md) | Document number allocation strategy | Atomic `UPDATE … RETURNING` on a counter table, inside the issuing transaction | Proposed |
| [ADR-003](./ADR-003-single-relational-database.md) | Single relational database with immutable snapshots | One PostgreSQL database; issued documents render from frozen snapshots; soft-deactivate master data | Proposed |
| [ADR-004](./ADR-004-audit-logging-approach.md) | Audit logging approach | Same-transaction append-only audit, privilege-enforced, hash-chained | Proposed |
| [ADR-005](./ADR-005-authorisation-model.md) | Authorisation model | Action-and-resource permissions, assignment-scoped technicians, logged overrides for small-team conflicts | Proposed |
| [ADR-006](./ADR-006-private-file-storage.md) | Private object storage | Private bucket, short-lived signed URLs after authorisation, immutable issued PDFs | Proposed |
| [ADR-007](./ADR-007-spreadsheet-migration.md) | Spreadsheet migration approach | Migrate master data and open items; preserve historical numbers verbatim; freeze spreadsheets at cutover | Proposed |

## Reading order

For a new contributor: **ADR-001 → ADR-003 → ADR-002 → ADR-004 → ADR-005 → ADR-006 → ADR-007**.

ADR-002 is placed early despite its narrow subject because document numbering is the highest-severity known defect and constrains several later decisions.

## How these records relate to other documents

| ADR | Elaborated by |
|---|---|
| ADR-001 | [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) |
| ADR-002 | [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) |
| ADR-003 | [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) |
| ADR-004 | [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md) |
| ADR-005 | [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) |
| ADR-006 | [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §7 |
| ADR-007 | [`../PRD.md`](../PRD.md) §6.8 |

An ADR states the decision and its rationale. The linked document states the operational detail. Where they appear to conflict, the ADR governs the decision and the document governs the procedure — a conflict should be raised rather than silently resolved.

## Status values

| Status | Meaning |
|---|---|
| Proposed | Recommended, awaiting ratification. Must not be cited as settled |
| Accepted | Ratified by the business owner and technical lead |
| Superseded | Replaced by a later ADR, which is named |
| Rejected | Considered and declined, with the reason recorded |

## Adding a decision

A new ADR is warranted when a choice is **architecturally significant** — it constrains future work, is expensive to reverse, or affects a cross-cutting property such as security, data integrity, or auditability.

Not warranted for: naming conventions, minor library choices, or anything reversible in an afternoon.

Each ADR must state alternatives considered and why they were rejected. A decision record that lists no alternatives is a decision that was not examined.
