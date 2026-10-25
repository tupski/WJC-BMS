# ADR-003 — Single relational database with immutable issued-document snapshots

| Field | Value |
|---|---|
| Status | **Accepted (style) — amended in place 2026-10-25 for MySQL/MariaDB** |
| Date | 2026-10-25 |
| Revision | Rev 2 — engine specifics replaced; decision unchanged |
| Deciders | Technical lead |
| Related | ADR-008 (platform), ADR-009 (allocation), [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) |

> **Revision note.** The decision — **one relational database with immutable snapshots and an append-only ledger** — is unchanged. What changed is the **engine**: the client mandates MySQL or MariaDB. Rev 2 replaces PostgreSQL-specific mechanisms with real equivalents and states plainly where the engine offers a weaker guarantee. No decision was reversed; several mechanisms were rewritten.

---

## Context

The system must hold customers, projects, material requirements, an append-only movement ledger, procurement records, work sessions, invoices, payments, and audit events, and must guarantee:

- Document-number uniqueness (FR-DOC-004).
- Reservation limits across projects (FR-MATL-006).
- Payment allocations never exceeding invoice totals (BR-FIN-003).
- Issued documents unaffected by later master-data **or company-profile** edits (FR-FIN-003, BR-SYS-001).
- No state change without its audit record (FR-AUD-003).
- No writable quantity column anywhere (BR-MATL-003).

Scale is small: ~18 users, thousands of documents per year. Two questions are bundled because they interact: **one database or several?** and **how are historical documents protected from master-data drift?**

---

## Decision

### D1 — A single relational database on MySQL or MariaDB

All business data in one InnoDB database. No per-module databases, no document store for business records, no separate analytics database in MVP.

**Engine floor: MySQL 8.0.16+ or MariaDB 10.2.1+** (NFR-DATA-009). Not arbitrary: earlier versions parse `CHECK` constraints and **silently ignore them**, making several integrity rules decorative.

### D2 — Denormalised immutable snapshots

On issuance, a frozen copy of party, line, **company, and bank** data is written to a `JSON` `snapshot` column, and the PDF is stored with a SHA-256 hash. Every render and reprint uses the snapshot.

**Engine note:** `JSON` replaces `JSONB`. MySQL's `JSON` has no GIN indexing, so anything filtered or aggregated must be a real relational column; `JSON` is reserved for snapshots and flexible metadata. Reports use the typed columns, never the snapshot payload.

### D3 — Soft deactivation of referenced master data

Referenced records are deactivated, never deleted. FKs use `ON DELETE RESTRICT` by default. No `CASCADE` on any business table.

**Engine note:** InnoDB enforces FKs but has **no deferrable constraints**. Inserts must be ordered correctly; where a genuine cycle exists, a nullable FK plus application-level sequencing is used.

### D4 — Append-only ledger for quantities

Quantities change only through `material_movements` rows. No writable balance column exists.

### D5 — Conditional uniqueness via generated columns

No partial unique indexes. Rules like "at most one active team leader per project" use a generated or nullable discriminator that is `NULL` when the rule should not apply — unique indexes ignore `NULL`. Documented once in DATA_MODEL §1.3.

### D6 — Case-insensitive uniqueness via normalised columns

No `citext`. A `*_normalized` column (application-maintained or stored generated) with a unique index. `_ci` collation is available but is not the primary control, because the rule should be visible in the schema rather than implied by collation behaviour.

### D7 — UTC storage, Jakarta rendering

`TIMESTAMP`/`DATETIME` store no offset and `timestamptz` does not exist. All timestamps are stored UTC in `DATETIME(3)`, converted at the boundary, rendered in `Asia/Jakarta`. Relying on the database server's local timezone is forbidden (NFR-OPS-009).

### D8 — Isolation, locking, deadlock policy

MySQL defaults to `REPEATABLE READ` with gap locks. Allocation and reservation paths run at `READ COMMITTED` or — preferably — use a single-row conditional `UPDATE` that is atomic without an explicit lock. Explicit `SELECT … FOR UPDATE` is used only where a check-then-write cannot be collapsed, always by primary key to keep the locked range narrow. Deadlocks are retried under a bounded policy for idempotent transactions only.

### D9 — Non-transactional DDL

DDL causes implicit commits, so a failed migration cannot be rolled back atomically. Migrations follow expand → migrate → contract (NFR-DATA-010).

---

## Consequences

### Positive

- **Every cross-domain guarantee is a single transaction.** Numbering, reservations, allocations, audit coupling — all native ACID within InnoDB.
- **Referential integrity is enforced by the database**, not by application hope.
- **Issued documents cannot drift.** The snapshot is the document; master data is only an input at issuance.
- **One backup, one restore procedure**, now explicitly including the files directory.
- **Historical reconstruction is possible.** Ledger plus snapshot plus audit.
- **Reporting is a query**, not a data pipeline.

### Negative

- **Snapshot storage grows.** Negligible at this volume.
- **Snapshot schema evolution is awkward.** Snapshot readers must tolerate missing fields and never assume the current schema.
- **Single database is a single failure domain.** Mitigated by backups, rehearsal, managed hosting.
- **Reporting competes with transactional load.** Not a practical concern at ~18 users.
- **Engine limitations are real and must be worked around, not assumed away:** no `JSONB`, no GIN, no `citext`, no partial unique indexes, no `RETURNING` on `UPDATE`, no deferrable FKs, no transactional DDL, and `CHECK` enforcement only from the stated versions. Each has a documented replacement (DATA_MODEL §1).
- **`CHECK` constraints are version-dependent.** On MySQL 5.7 they are parsed and ignored. The boot-time version assertion (AC-DATA-009) is a control, not a formality.

### Neutral

- `JSON` for snapshots trades queryability for immutability. That is the intended trade.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Separate database per module | **Rejected** — destroys the cross-module transactional guarantees the two highest-risk requirements depend on |
| Document database for business records | **Rejected** — the data is highly relational; uniqueness constraints, FKs, and multi-row transactions are exactly what it needs |
| Normalised rendering of issued documents (no snapshot) | **Rejected** — the failure mode FR-FIN-003 exists to prevent. The anti-anomaly argument applies to mutable data; an issued document is a historical fact |
| Event sourcing as the system of record | **Rejected** — the audit log already provides attributable history |
| Separate analytics database / warehouse | **Rejected for MVP** — premature at this volume |
| Hard delete with audit trail | **Rejected** — breaks every historical reference; an audit record saying "customer deleted" does not let anyone reprint last year's invoice |
| Snapshots as normalised child tables instead of `JSON` | **Considered during the engine reconciliation and rejected** — multiplies tables per document type and makes immutability harder to state, because a child row could be edited. A write-once `JSON` column is a simpler, stronger statement |

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Rename customer after issuance, re-download invoice | Byte-identical — AC-FIN-003 |
| V2 | Change item price after issuance | Totals unchanged — AC-FIN-003 |
| V3 | **Change company bank account after issuance** | Byte-identical; old account printed — AC-FIN-003, AC-SYS-003 |
| V4 | Delete referenced customer | Refused by FK — AC-DATA-005 |
| V5 | Attempt quantity edit | No writable column — AC-MATL-012 |
| V6 | Boot against an older engine | Refused — AC-DATA-009 |
| V7 | Insert a row violating a `CHECK` | Rejected, not silently accepted — AC-DATA-009 |
| V8 | Conditional unique rule (one active leader) | Second active leader rejected — AC-PROJ-007 |
| V9 | Restore backup, verify snapshots and chain | Recovery preserves history — AC-DATA-004, AC-AUD-003 |

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-DB-01 | Managed MySQL/MariaDB or self-hosted? | Managed if available in-region; self-hosted acceptable at this scale |
| Q-DB-02 | Read replica for reporting? | No at MVP scale; revisit on measured degradation |
| Q-DB-03 | Snapshot retention: keep forever or archive? | Keep with the document; never delete |
| Q-DB-04 | **MySQL or MariaDB specifically?** | Choose at implementation planning; both satisfy the floor. Prefer what the host supports best |

---

## Cross-references

- Platform: [`./ADR-008-laravel-platform-baseline.md`](./ADR-008-laravel-platform-baseline.md)
- Allocation: [`./ADR-009-mysql-document-number-allocation.md`](./ADR-009-mysql-document-number-allocation.md)
- Data model: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md)
- Architecture: [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §5
