# ADR-002 — Document number allocation strategy

| Field | Value |
|---|---|
| Status | **Superseded by [ADR-009](./ADR-009-mysql-document-number-allocation.md)** (2026-10-25) |
| Date | 2026-10-25 |
| Superseded on | 2026-10-25 |
| Deciders | Business owner, technical lead |
| Related | ADR-009 (replacement), [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) |

> **Supersession note.** This record chose `UPDATE … RETURNING` on a counter table. **That statement does not execute on MySQL or MariaDB**, and the client has mandated one of those engines. ADR-009 rewrites the mechanism as `UPDATE … SET next_value = LAST_INSERT_ID(next_value + :step)` followed by `SELECT LAST_INSERT_ID()`.
>
> **Survives unchanged:** the counter-table approach, the transactional rollback property, one series per document type, allocation at issuance only, the database uniqueness constraint, immutable issued numbers, period derivation from the document date, and verbatim preservation of historical numbers. The body below is history; its reasoning about *why* a counter table beats a native sequence remains the basis of ADR-009.

---

## Context

**[Confirmed]** An invoice-numbering conflict has occurred at WJ-CCTV. The cause is **unconfirmed**.

Two distinct risks must not be conflated:

- **Risk A — internal duplication:** two documents in WJ-CCTV's own records share a number. Solvable here.
- **Risk B — external conflict:** a number is rejected by, or collides within, a client's system. **Not** solvable by an internal database alone.

This record addresses Risk A only. Risk B requires client confirmation (Q-NUM-01) and is handled by per-client format configuration, not by allocation mechanics. The mechanism must be correct regardless of the final format, because **format is configuration and allocation is correctness**.

---

## Decision (as originally recorded)

Allocate using a **counter table with a single atomic `UPDATE … RETURNING`**, executed inside the transaction that creates the document.

```sql
-- HISTORICAL. This syntax is not valid on MySQL or MariaDB.
INSERT INTO document_sequences (series_key, period_key, last_value)
VALUES ($1, $2, 0) ON CONFLICT (series_key, period_key) DO NOTHING;

UPDATE document_sequences SET last_value = last_value + 1
 WHERE series_key = $1 AND period_key = $2
RETURNING last_value;
```

| # | Supporting decision | Still current? |
|---|---|---|
| D1 | One series per document type — never shared | **Yes** |
| D2 | Series scope: company prefix + document type + period | **Yes**, now configurable per type |
| D3 | Number allocated at **issuance**, never at draft creation | **Yes** |
| D4 | `UNIQUE (series_key, document_number)` is mandatory and authoritative | **Yes** |
| D5 | Issued numbers are immutable and never reused, including after void | **Yes** |
| D6 | Period derived from the **document date**, not the server clock | **Yes** |
| D7 | Historical imported numbers preserved verbatim, flagged, excluded from counter advancement | **Yes** |
| D8 | Series counters initialised **above** the historical high-water mark | **Yes** |

---

## Consequences

### Positive — still true of the replacement

- **Atomic under concurrency.** The row lock serialises concurrent transactions; each receives a distinct value.
- **Transactional.** A rolled-back transaction returns the number; failed issuances do not burn numbers.
- **No gaps from failures.** Gaps arise only from voids — explainable business events.
- **Series configuration is data**, not schema. Adding a document type is a row, not a migration.
- **Inspectable.** The counter is a normal table row, visible and correctable with an audit trail.
- **Format-agnostic.** Works unchanged when the format changes for a client requirement.

### Negative

- **Row contention within a series.** Irrelevant at ~3 office staff.
- **Requires explicit transaction discipline.** Allocation must not be called outside the issuing transaction.
- **Manual counter correction** is possible and must itself be audited.
- **Discovered during the engine change:** the `UPDATE … RETURNING` and `ON CONFLICT` idioms were load-bearing in this design and are both unavailable. That is why a replacement record was required rather than an edit.

### Neutral

- Gaps after voids are expected and acceptable (DOCUMENT_NUMBERING §3, G8). Gap-free numbering is explicitly **not** a goal, because chasing it leads to number reuse.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Native database sequence per series | **Viable, rejected.** The decisive difference is rollback: with a non-transactional sequence, every failed issuance permanently burns a number, and unexplained gaps invite the "reuse the missing number" behaviour that plausibly caused the original conflict. **Unchanged in ADR-009** |
| `SELECT MAX(number) + 1` | **Rejected outright** — two concurrent transactions read the same maximum. The most likely original cause |
| Read then write in application code | **Rejected** — even with a constraint catching the collision, the user experience is a failed save under concurrency |
| UUID or timestamp-based numbers | **Rejected** — not human-verifiable, not sortable, unacceptable on a printed invoice; timestamp schemes collide under same-second issuance |
| Client-side generation with server validation | **Rejected** — directly contradicts FR-DOC-004 |
| Separate numbering service | **Rejected** — introduces a distributed boundary for a problem a single row lock solves |
| `AUTO_INCREMENT` on the document table | **Considered during the engine reconciliation and rejected** — cannot express per-series, per-period numbering with a configurable start and step, and gives no natural home for the high-water-mark rule |

---

## What this ADR does not solve

**Risk B remains open.** A company prefix makes external collisions unlikely, not impossible, and does not satisfy a client whose system rejects the format. Nine questions must be answered in writing before the invoice format is finalised (DOCUMENT_NUMBERING §2.4, Q-NUM-01).

**Explicit rejection of a common claim:** "we'll add a prefix and the problem goes away" solves Risk A while describing it as Risk B. Neither this record nor ADR-009 may be cited as evidence that external conflicts are resolved.

---

## Verification obligations (carried into ADR-009)

| # | Test | Proves |
|---|---|---|
| V1 | 50 concurrent issuances, one series | No duplicates |
| V2 | Rollback after allocation | Number not consumed |
| V3 | Direct duplicate insert | Constraint rejects |
| V4 | Client supplies a number | Ignored |
| V5 | Period rollover | New counter |
| V6 | Backdated issuance | Correct period |
| V7 | Migration high-water mark | No post-migration collision |

**V1 must run in parallel.** A sequential test of a numbering bug passes trivially.

---

## Cross-references

- Replacement: [`./ADR-009-mysql-document-number-allocation.md`](./ADR-009-mysql-document-number-allocation.md)
- Standard: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Entities: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §8
