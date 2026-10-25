# ADR-009 — Atomic document-number allocation on MySQL/MariaDB

| Field | Value |
|---|---|
| Status | **Accepted (design) — not implemented** |
| Date | 2026-10-25 |
| Supersedes | [ADR-002](./ADR-002-document-number-allocation.md) (mechanism only; all supporting decisions carried forward) |
| Deciders | Technical lead, business owner |
| Related | ADR-003 (engine), ADR-008 (platform), [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) |

---

## Context

ADR-002 chose `UPDATE … RETURNING` on a counter table. **That statement does not execute on MySQL or MariaDB**, and the client has mandated one of those engines (ADR-008).

| PostgreSQL idiom used in ADR-002 | Availability here |
|---|---|
| `UPDATE … RETURNING last_value` | **Not supported.** MariaDB has `RETURNING` for `INSERT`/`DELETE` in some versions; MySQL has none |
| `INSERT … ON CONFLICT DO NOTHING` | **Not supported.** Equivalents are `INSERT IGNORE` and `INSERT … ON DUPLICATE KEY UPDATE` |

The *reasoning* behind ADR-002 is not being revisited: a transactional counter table beats a non-transactional sequence because a rolled-back issuance must not burn a number, and unexplained gaps invite the reuse behaviour that plausibly caused the original conflict. Only the SQL changes.

**[Confirmed]** An invoice-numbering conflict has occurred; the cause is **unconfirmed**. This ADR addresses internal duplication (Risk A) only.

---

## Decision

### D1 — Single-statement atomic claim using `LAST_INSERT_ID()`

The mechanism is a counter row claimed by `UPDATE … SET next_value = LAST_INSERT_ID(next_value + :step)`, with the value read back by `SELECT LAST_INSERT_ID()` on the same connection, inside the issuing transaction. **The full statement, its ordering constraints, and the idempotent counter-creation idiom are specified once in [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §5.1** — that is the authoritative copy; this ADR records why it was chosen.

**Why atomic.** The `UPDATE` holds an exclusive row lock for the transaction, so concurrent issuances serialise on that row and each sees the previous committed `next_value`. `LAST_INSERT_ID(expr)` sets a **connection-scoped** register and returns `expr`, so no other session can observe or overwrite the value between the update and the read-back.

### D2 — No `ON CONFLICT`; use an explicit no-op update

`INSERT IGNORE` would also satisfy idempotency but swallows **all** warnings, including data truncation — hiding a real error behind a silent success. The explicit no-op update is unambiguous about intent.

**Documented side effect:** `ON DUPLICATE KEY UPDATE` can advance an `AUTO_INCREMENT` counter on a conflicting insert even when no row is written. `number_series` therefore uses a **composite natural primary key** `(series_key, period_key)` rather than an auto-increment surrogate, removing the side effect entirely.

### D3 — One series per type, allocated at issuance only

Carried forward: one series per document type; allocation at formal issuance, never at draft creation; format is configuration; the number is never a primary key.

### D4 — The database constraint is the guarantee

`UNIQUE (series_key, document_number)` on `issued_documents` is authoritative; application checks are advisory. A unique index is enforced consistently by MySQL/MariaDB, unlike `CHECK`, which requires the version floor in ADR-003.

### D5 — Increment step and void gaps are separate mechanisms

| Concept | Behaviour |
|---|---|
| **Increment step** | Difference between consecutive allocated values. Step 5 ⇒ `00001`, `00006`, `00011` |
| **Void gap** | An issued number whose document was voided. Retired permanently; **never reused** |

**A step of 5 does not license reuse of a voided number.** Stated because the client's use of "gap" is ambiguous (**Q-NUM-07**) and because a business chasing gap-free numbering will eventually request reuse.

### D6 — The counter is never lowered below the high-water mark

`high_water_mark` records the highest value ever allocated. `CHECK (next_value > high_water_mark - 1)` and a configuration validator both refuse a starting value or step change that could collide with issued or imported numbers (**Q-NUM-08**).

### D7 — Transactional rollback returns the number

Because the increment is an ordinary row update, a failed issuance (PDF error, validation failure, snapshot failure) releases the value. A document that never existed should not consume a number.

### D8 — Isolation and deadlock policy

Allocation uses the single-row `UPDATE`, which needs no explicit lock and is unaffected by gap-lock behaviour. Where a surrounding transaction also reads stock or allocation rows, it runs at `READ COMMITTED`. Deadlocks are retried under a bounded policy (3 attempts, exponential backoff) for idempotent transactions only; otherwise surfaced.

### D9 — Period derived from the document date

Not the server clock, so a backdated document receives its own period's number. Future-dated issuance beyond a configurable window is refused.

---

## Consequences

### Positive

- **Atomic and transactional** without `RETURNING`, `ON CONFLICT`, or a sequence object.
- **Rollback returns the number**, so failures do not create unexplained gaps.
- **Series configuration is data** — prefix, pattern, separators, padding, starting value, step, and reset policy (FR-SYS-004).
- **Inspectable and correctable**, with the high-water-mark guard preventing the most dangerous manual change.
- **Format-agnostic**, so a client-specific format is a configuration change, not a code change.

### Negative

- **`LAST_INSERT_ID()` is connection-scoped and easy to misuse.** A connection pool, a read/write split, or a queued job reading it later would get the wrong value. The rule — *read it back in the same transaction, on the same connection, immediately* — must be enforced by keeping allocation inside one repository method with no intervening queries.
- **The idiom is unfamiliar.** A developer who "cleans it up" into `SELECT` then `UPDATE` reintroduces the original bug. The prohibited-patterns table exists for this reason, and the concurrency test is the guard.
- **Row contention within a series.** Irrelevant at ~3 office staff.
- **`ON DUPLICATE KEY UPDATE` has a documented `AUTO_INCREMENT` side effect**, mitigated by the composite natural key.
- **The `CHECK`-based high-water guard needs the engine floor**, so it is also enforced in configuration validation.

### Neutral

- Void gaps remain expected. Gap-free numbering is explicitly not a goal.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| `SELECT next_value` then `UPDATE` in application code | **Rejected outright** — the classic read-then-write race and the most likely original cause |
| `SELECT MAX(number) + 1` | **Rejected outright** — two concurrent transactions read the same maximum |
| `AUTO_INCREMENT` on the document table | **Rejected** — cannot express per-series, per-period numbering with a configurable start and step; no natural home for the high-water rule; counter behaviour on rolled-back inserts is not a guarantee the business can reason about |
| A stored procedure with one row per series | **Rejected for MVP** — moves business logic into the database where it is harder to test, version, and review, for no correctness gain over a single-statement `UPDATE` |
| Database sequence emulation via one table per series | **Rejected** — unbounded DDL per series and complicates configuration-as-data |
| `INSERT IGNORE` for idempotent counter creation | **Rejected as the primary idiom** — swallows data-truncation warnings |
| Advisory locks (`GET_LOCK`) around allocation | **Rejected** — adds a named-lock lifecycle with its own failure modes (leaked locks, timeouts) to solve a problem a row `UPDATE` already solves |
| Allocate outside the transaction and reconcile later | **Rejected** — guarantees gaps on every failure and breaks the "no numbered document without a PDF" invariant |

---

## Verification obligations

| # | Test | Level | Proves |
|---|---|---|---|
| V1 | 50 concurrent issuances, one series | I | No duplicates — AC-DATA-002 |
| V2 | 20 concurrent issuances across 4 series | I | No cross-series contention — AC-DATA-002 |
| V3 | Rollback after allocation | I | Number not consumed — AC-DOC-004 |
| V4 | Direct duplicate insert at DB level | I | Constraint rejects — AC-DATA-002 |
| V5 | Client supplies a number | I | Ignored — AC-DOC-004 |
| V6 | Month rollover | I | New counter at `starting_value` — AC-DOC-004 |
| V7 | Backdated issuance | I | Correct period — AC-DOC-004 |
| V8 | Void then issue | I | Voided number never reused — AC-FIN-011 |
| V9 | PDF failure during issuance | I | No numbered document — AC-FIN-002 |
| V10 | Step = 5 | I | `00001`, `00006`, `00011` — AC-SYS-004 |
| V11 | Void at `00006` with step 5 | I | Next is `00011`, never `00006` — AC-SYS-004 |
| V12 | Starting-number change colliding with history | I | Refused — AC-SYS-004 |
| V13 | Counter lowered below the high-water mark | I | Refused — AC-SYS-004 |
| V14 | Heavy contention / deadlock | I | Bounded retry, no corruption — AC-DATA-007 |
| V15 | Migration high-water mark | I | No post-migration collision — AC-DATA-004 |

**V1 must run in parallel** — a sequential test of a numbering bug passes trivially. **V11 proves the step/gap distinction is implemented** rather than merely documented.

---

## What this ADR does not solve

**Risk B remains open.** A company prefix makes external collisions unlikely, not impossible, and does not satisfy a client whose system rejects the format. Nine questions must be answered in writing before the invoice format is finalised (DOCUMENT_NUMBERING §2.4, Q-NUM-01).

**The historical root cause remains unconfirmed.** This ADR must not be cited as evidence that it has been identified.

**No allocation code exists.** This record states a design; it is not a description of running software.

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-NUM-01 | Client format and validation requirements | Internal-consistent configurable format; external constraints per client |
| Q-NUM-06 | Reset scope: monthly, yearly, or continuous | Monthly, matching the initial format example |
| Q-NUM-07 | Meaning and permitted range of the configurable step | Step = allocation increment, default 1; "gap" = void hole, never reused |
| Q-NUM-08 | How starting numbers and step changes interact with existing allocations and imports | Future allocations only; never below high-water mark; audited; refused on possible collision |
| Q-NUM-09 | Documented offset needed if historical formats are irregular? | Yes — start at a recorded offset and explain it in the migration report |

---

## Cross-references

- Superseded record: [`./ADR-002-document-number-allocation.md`](./ADR-002-document-number-allocation.md)
- Standard: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Engine: [`./ADR-003-single-relational-database.md`](./ADR-003-single-relational-database.md)
- Platform: [`./ADR-008-laravel-platform-baseline.md`](./ADR-008-laravel-platform-baseline.md)
- Tables: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §8
