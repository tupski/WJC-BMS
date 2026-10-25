# ADR-002 — Document number allocation strategy

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-25 |
| Deciders | Business owner, technical lead |
| Related | [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md), ADR-003 |

---

## Context

**[Confirmed]** An invoice-numbering conflict has occurred at WJ-CCTV. The cause is **unconfirmed**.

Two distinct risks exist and must not be conflated:

- **Risk A — internal duplication:** two documents in WJ-CCTV's own records share a number. Solvable here.
- **Risk B — external conflict:** a number is rejected by, or collides within, a client's system. **Not** solvable by an internal database alone.

This ADR addresses Risk A only. Risk B requires client confirmation (Q-NUM-01) and is handled by per-client format configuration, not by allocation mechanics.

The allocation mechanism must be correct regardless of the final format, because format is configuration and allocation is correctness.

---

## Decision

Allocate document numbers using a **counter table with a single atomic `UPDATE … RETURNING`**, executed inside the transaction that creates the document.

```sql
INSERT INTO document_sequences (series_key, period_key, last_value)
VALUES ($1, $2, 0)
ON CONFLICT (series_key, period_key) DO NOTHING;

UPDATE document_sequences
   SET last_value = last_value + 1
 WHERE series_key = $1 AND period_key = $2
RETURNING last_value;
```

Supporting decisions:

| # | Decision |
|---|---|
| D1 | One series per document type (`INV`, `PO`, `QUO`, `CN`, `RCP`, `WO`) — never shared |
| D2 | Series scope: company prefix + document type + period (year by default) |
| D3 | Number allocated at **issuance**, never at draft creation |
| D4 | `UNIQUE (series_key, document_number)` constraint is mandatory and authoritative |
| D5 | Issued numbers are immutable and never reused, including after void |
| D6 | Period derived from the **document date**, not the server clock, so backdated documents get period-correct numbers |
| D7 | Historical imported numbers preserved verbatim, flagged, and excluded from counter advancement |
| D8 | Series counters initialised **above** the historical high-water mark during migration |

---

## Consequences

### Positive

- **Atomic under concurrency.** The row lock is taken by the `UPDATE`; concurrent transactions serialise and each receives a distinct value.
- **Transactional.** A rolled-back transaction returns the number. Failed issuances do not burn numbers.
- **No gaps from failures.** Gaps arise only from voids, which are explainable business events.
- **Series configuration is data**, not schema. Adding a document type is a row, not a migration.
- **Inspectable.** The counter is a normal table row, visible and correctable with an audit trail.
- **Format-agnostic.** The mechanism works unchanged when the format changes for a client requirement.

### Negative

- **Row contention within a series.** Concurrent issuance in one series serialises on one row. At ~3 office staff this is irrelevant; at thousands of issuances per second it would matter.
- **Requires explicit transaction discipline.** The allocation must not be called outside the issuing transaction.
- **Manual counter correction** is possible and must itself be audited.

### Neutral

- Gaps after voids are expected and acceptable (DOCUMENT_NUMBERING §3, G8). Gap-free numbering is explicitly **not** a goal, because chasing it leads to number reuse.

---

## Alternatives considered

### Alternative A — PostgreSQL native `SEQUENCE` per series

**Viable, rejected.**

| | Counter table | Native sequence |
|---|---|---|
| Atomic | Yes | Yes |
| Rollback returns the number | **Yes** | **No** — `nextval` is non-transactional |
| Gaps on failed issuance | None | One per failure |
| Configuration | Data | DDL per series |

The decisive difference is rollback behaviour. With a native sequence, every failed issuance (PDF error, validation failure, network drop) permanently burns a number. Over time the sequence fills with unexplained gaps — and unexplained gaps invite exactly the "reuse the missing number" behaviour that plausibly caused the original conflict.

### Alternative B — `SELECT MAX(number) + 1`

**Rejected outright.** Two concurrent transactions read the same maximum and both write the same value. This is the most likely original cause and would reproduce the defect in the replacement system.

### Alternative C — Read then write in application code

**Rejected.** Even with a uniqueness constraint catching the collision, the user experience is a failed save under concurrency. The constraint would prevent duplicates but the design would be wrong.

### Alternative D — UUID or timestamp-based numbers

**Rejected.** Not human-verifiable, not sortable, not acceptable on a printed invoice, and timestamp schemes collide under same-second issuance.

### Alternative E — Client-side generation with server validation

**Rejected.** Directly contradicts FR-DOC-004. Client state is not authoritative; two tabs, a refresh, or a double-submit all break it. Frontend validation is never a control.

### Alternative F — Separate numbering service

**Rejected.** Introduces a distributed boundary for a problem that a single row lock solves. It would make the numbering requirement harder while appearing more rigorous.

---

## Verification obligations

The decision is only as good as its tests. Required:

| # | Test | Proves |
|---|---|---|
| V1 | 50 concurrent issuances, one series | No duplicates |
| V2 | Rollback after allocation | Number not consumed |
| V3 | Direct duplicate insert | Constraint rejects |
| V4 | Client supplies a number | Ignored |
| V5 | Year rollover | New counter |
| V6 | Backdated issuance | Correct period |
| V7 | Migration high-water mark | No post-migration collision |

**V1 must run in parallel.** A sequential test of a numbering bug passes trivially and proves nothing.

Acceptance: AC-DATA-002, AC-DOC-004, AC-FIN-002, AC-FIN-011.

---

## What this ADR does not solve

**Risk B remains open.** A company prefix makes external collisions unlikely; it does not make them impossible, and it does not satisfy a client whose system rejects the format. Nine specific questions must be answered in writing before the invoice format is finalised (DOCUMENT_NUMBERING §2.4, Q-NUM-01).

**Explicit rejection of a common claim:** "we'll add a prefix and the problem goes away" solves Risk A while describing it as Risk B. This ADR must not be cited as evidence that external conflicts are resolved.
