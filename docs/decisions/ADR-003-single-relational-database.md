# ADR-003 — Single relational database with immutable issued-document snapshots

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-25 |
| Deciders | Technical lead |
| Related | ADR-001, ADR-002, [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) |

---

## Context

The system must hold customers, projects, material requirements, an append-only movement ledger, procurement records, invoices, payments, and audit events. It must guarantee:

- Document-number uniqueness (FR-DOC-004).
- Reservation limits across projects (FR-MATL-006).
- Payment allocations never exceeding invoice totals (BR-FIN-003).
- Issued documents unaffected by later master-data edits (FR-FIN-003).
- No state change without its audit record (FR-AUD-003).

Scale is small: ~18 users, thousands of documents per year, not millions per day.

Two design questions are bundled here because they interact:

1. **One database or several?**
2. **How are historical documents protected from master-data drift?**

---

## Decision

### D1 — A single relational database (PostgreSQL)

All business data lives in one PostgreSQL database. No per-module databases, no document store for business records, no separate analytics database in MVP.

### D2 — Denormalised immutable snapshots for issued documents

When a document is issued, a frozen copy of all party and line data is written to a `snapshot` JSONB column on the document row (and to `issued_documents`), and the rendered PDF is stored with a SHA-256 hash.

Every render, retrieval, and reprint uses the snapshot. No issued document is ever rendered from live master data.

### D3 — Soft deactivation of referenced master data

Referenced master records are deactivated (`is_active = false`), never deleted. Foreign keys use `RESTRICT`. No `CASCADE` delete exists on any business table.

### D4 — Append-only ledger for quantities

Material quantities change only through `material_movements` rows. There is no writable balance column that a user or endpoint can edit.

---

## Consequences

### Positive

- **Every cross-domain guarantee is a single transaction.** Numbering, reservations, allocations, audit coupling — all native ACID.
- **Referential integrity is enforced by the database**, not by application hope.
- **Issued documents cannot drift.** The snapshot is the document; master data is only an input at issuance time.
- **One backup, one restore procedure.** Recovery is testable end to end (NFR-DATA-004).
- **Historical reconstruction is possible.** Ledger plus snapshot plus audit provides a complete, explainable history.
- **Reporting is a query**, not a data pipeline.

### Negative

- **Snapshot storage grows.** Every issued document duplicates its line data. At this volume this is negligible; it would matter at millions of documents.
- **Snapshot schema evolution is awkward.** Old snapshots retain old shapes. Mitigation: snapshot readers must tolerate missing fields and never assume the current schema.
- **Single database is a single failure domain.** Mitigated by backups, restore rehearsal, and managed hosting.
- **Reporting competes with transactional load.** At ~18 users this is not a practical concern; if it becomes one, a read replica or export is the answer, not a separate write store.

### Neutral

- JSONB for snapshots trades queryability for immutability. Reports over issued documents use the typed columns (totals, dates, status), not the snapshot, so query performance is unaffected.

---

## Alternatives considered

### Alternative A — Separate database per module

**Rejected.** Destroys the cross-module transactional guarantees that the two highest-risk requirements depend on. A reservation check that must consult another database cannot be atomic without distributed transactions or sagas.

### Alternative B — Document database (e.g. MongoDB) for business records

**Rejected.** The data is highly relational (customer → project → requirement → movement → receipt → payable). Uniqueness constraints, foreign keys, and multi-row transactions are exactly what this workload needs and exactly what document stores handle least naturally.

### Alternative C — Normalised rendering of issued documents (no snapshot)

**Rejected.** This is the failure mode FR-FIN-003 exists to prevent. Rendering from live data means a customer rename or price change silently alters a document the customer already holds. In a dispute, the business could not produce what it actually issued.

The argument for normalisation — avoiding update anomalies — applies to mutable data. An issued document is not mutable data; it is a historical fact. Storing the fact as it was is the correct representation, not a denormalisation compromise.

### Alternative D — Event sourcing as the system of record

**Rejected.** The audit log already provides attributable history. Event sourcing would require reconstructing every entity from events, complicate the numbering requirement, and add eventual-consistency reasoning for no business benefit at this scale.

### Alternative E — Separate analytics database / warehouse

**Rejected for MVP.** Premature. Reports run against the transactional database at this volume. Revisit only if reporting measurably degrades transactional performance.

### Alternative F — Hard delete with audit trail

**Rejected.** A deleted customer breaks every historical reference, and an audit record saying "customer deleted" does not let anyone reprint last year's invoice. Soft deactivation preserves the referential chain.

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Rename customer after issuance, re-download invoice | Byte-identical (hash match) — AC-FIN-003 |
| V2 | Change item price after issuance | Totals unchanged — AC-FIN-003 |
| V3 | Delete referenced customer | Refused by FK — AC-DATA-005 |
| V4 | Attempt quantity edit | No writable column — AC-MATL-012 |
| V5 | Restore backup, verify snapshots and chain | Recovery preserves history — AC-DATA-004, AC-AUD-003 |

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-DB-01 | Managed PostgreSQL or self-hosted? | Managed, to reduce operational burden |
| Q-DB-02 | Is a read replica needed for reporting? | No at MVP scale; revisit on measured degradation |
| Q-DB-03 | Snapshot retention: keep forever or archive? | Keep with the document; never delete |
