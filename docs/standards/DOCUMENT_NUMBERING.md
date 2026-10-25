# Document Numbering Standard — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed — mechanism is design; final format awaits Q-NUM-01 |
| Version | 0.1 |
| Scope owner | This document is the **single authority** on document numbers, allocation, uniqueness, and cancellation |
| Requirement source | [`../PRD.md`](../PRD.md) FR-DOC-004, FR-DOC-006, FR-FIN-011, NFR-DATA-002, BR-FIN-002, BR-DOC-002, §11 |
| Data model | [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §8 |
| Decision record | [`../decisions/ADR-002-document-number-allocation.md`](../decisions/ADR-002-document-number-allocation.md) |

---

## 1. Why this document exists

**[Confirmed]** An invoice-numbering conflict has occurred at WJ-CCTV. The exact cause is **unconfirmed**.

This document does not assume a cause. It analyses both plausible causes separately, because they require different remedies and only one of them is solvable inside this system.

**[Confirmed]** The intended solution is a centralised, auditable platform. A centralised system removes exactly one class of failure — concurrent local generation — and cannot remove the other. Being precise about which is which is the difference between fixing the problem and appearing to fix it.

---

## 2. Two distinct risks

### 2.1 Risk A — Internal duplicate generation

Two documents in WJ-CCTV's own records carry the same number.

| Aspect | Analysis |
|---|---|
| **Cause** | Read-then-write race (two users read "last number = 41", both write 42); `MAX()+1` computed client-side; manual entry in a spreadsheet; number reused after a cancelled document; two spreadsheets with independent counters; copying a previous invoice as a template without clearing the number |
| **Where it lives** | Entirely inside WJ-CCTV's control |
| **Solvable here?** | **Yes, completely** |
| **Remedy** | Server-side atomic allocation + database uniqueness constraint + immutable issued numbers (§4, §5, §6) |
| **Residual risk** | Near zero if §5 and §6 are implemented and tested under concurrency |

### 2.2 Risk B — Conflict with external numbering rules

A number that is internally unique is rejected, or collides, in someone else's system.

| Aspect | Analysis |
|---|---|
| **Cause** | A client's accounting system validates incoming invoice numbers against its own registry; the client requires a specific format, length, or charset; the client treats numbers from different vendors as one namespace; a national/industry registry exists; a required prefix or suffix is missing; the number exceeds a field length in the client's system |
| **Where it lives** | Partly or wholly **outside** WJ-CCTV's control |
| **Solvable here?** | **No — not by an internal database alone** |
| **Remedy** | Per-client format configuration after written confirmation of each client's rules; possibly a client-assigned identifier carried alongside our number |
| **Residual risk** | Cannot be eliminated by this system; can only be reduced by conforming to confirmed external rules |

### 2.3 The claim this document explicitly rejects

> **A company prefix does not resolve external conflicts.**

A prefix such as `WJCCTV/INV/2026/0001` makes collisions with other vendors *unlikely*. It does not make them impossible, and it does not satisfy a client whose system rejects the format outright, requires a fixed length, forbids slashes, or validates against a registry that does not know about WJ-CCTV.

Anyone who proposes "we'll add a prefix and the problem goes away" is solving Risk A and describing it as Risk B. This document separates them so the business can see which problem is actually being fixed.

### 2.4 What must be confirmed with affected clients — Q-NUM-01

Before the invoice format is finalised, each affected client must confirm in writing:

| # | Question | Why it matters |
|---|---|---|
| 1 | Exact permitted length (min/max characters) | A format that exceeds a 20-character field will be rejected |
| 2 | Permitted character set (digits only? slashes? hyphens? case?) | Slashes are commonly rejected by ERP import fields |
| 3 | Is a prefix permitted, required, or forbidden? | Determines whether a company prefix helps or hurts |
| 4 | Does the client validate against a national/global registry? | If yes, our numbering must conform to an external authority, not our own scheme |
| 5 | Must historical numbers be preserved verbatim? | Determines whether migration may renumber old documents |
| 6 | Are gaps in the sequence acceptable? | Determines whether voided numbers may leave holes |
| 7 | How must a cancelled/voided invoice be represented? | Some clients require a cancellation document with its own number; others require a status flag |
| 8 | Is the client's validation per-vendor or global? | A global namespace means our numbers compete with other vendors' |
| 9 | What does the client do on rejection — hard fail or warning? | Determines the operational impact |

**Status of all nine: [Open Question].** None has been confirmed. The internal mechanism in §4–§6 can be built and tested before these answers arrive, because it is format-agnostic — the format is configuration, the allocation mechanism is not.

---

## 3. Design goals

| # | Goal | Non-goal |
|---|---|---|
| G1 | No two documents in one series ever share a number | — |
| G2 | Numbers are human-readable and sortable | Numbers are not meaningful beyond identity |
| G3 | Allocation is atomic under concurrency | — |
| G4 | Issued numbers are permanent | Numbers are not editable to "fix" a mistake |
| G5 | Format is configurable per document type and per client where required | Format is not hard-coded in templates |
| G6 | Historical numbers survive migration verbatim | Historical numbers are not renumbered for tidiness |
| G7 | Every allocation is attributable | — |
| G8 | Gaps are acceptable and expected | Gap-free numbering is **not** a goal |

**G8 deserves emphasis.** Businesses chase gap-free sequences because gaps look like missing documents. The consequence is number reuse — which is exactly the failure being eliminated. A gap means "a document was issued and voided", which is a normal, explainable event. Numbering must never be "repaired" by reusing a number.

---

## 4. Number structure

### 4.1 Components

```
{PREFIX}/{TYPE}/{YEAR}/{SEQUENCE}
   │        │      │        └── zero-padded counter, resets per period
   │        │      └─────────── allocation period (year, optionally month)
   │        └────────────────── document type code
   └─────────────────────────── company prefix (configurable)
```

**Example (illustrative only, not final):** `WJC/INV/2026/00042`

### 4.2 Series definition

A **series** is the unit of uniqueness and the unit of counting. It is identified by `series_key`:

```
series_key = {company_prefix}:{document_type}:{period_key}
e.g.  WJC:INV:2026
```

| Attribute | Options | Notes |
|---|---|---|
| `company_prefix` | Configurable string | e.g. `WJC`. Must be confirmable against client rules (Q-NUM-01 #3) |
| `document_type` | `INV`, `PO`, `QUO`, `CN`, `RCP`, `WO`, `PAY` | One series per type — never shared (FR-DOC-006) |
| `reset_period` | `YEAR`, `MONTH`, `NONE` | **[Proposed]** `YEAR` default |
| `padding` | Integer, default 5 | `00042` |
| `format_pattern` | Template string | Rendered server-side only |
| `is_active` | Boolean | Deactivating a series never invalidates issued numbers |

### 4.3 Scope of uniqueness

| Scope | Enforced? |
|---|---|
| Company + document type + period | **Yes** — unique constraint on `(series_key, document_number)` |
| Across document types | Not required; `INV/2026/00042` and `PO/2026/00042` may coexist because the type is part of the number |
| Across companies | Not applicable in MVP (single entity) |
| Globally / across vendors | **Cannot be guaranteed** — see §2.2 |

### 4.4 Internal ID vs document number

| | Internal ID | Document number |
|---|---|---|
| Format | UUID | Human-readable string |
| Purpose | Referential integrity, API identity | Human communication, printed documents |
| Generated | On record creation | At **issuance** only |
| Mutable | Never | Never (once issued) |
| Unique | Yes, globally | Yes, per series |
| Used in URLs | Yes (or a route key) | No — but may be used for lookup |
| Exposed to users | Rarely | Always |

**Rule:** a document number is never a primary key and never a foreign key. Applications join on UUIDs. This prevents a numbering scheme change from cascading through the schema, and prevents accidental coupling between identity and human labels.

### 4.5 Draft vs issued

| State | Has number? | Notes |
|---|---|---|
| Draft | **No** | May show a preview like `DRAFT` or a provisional placeholder clearly marked as not allocated |
| Issued | **Yes** | Allocated atomically at issuance |
| Voided | Yes, retained | Number is never released or reused |
| Superseded / credited | Yes, retained | Replacement carries its own number |

**Numbers are allocated at issuance, never at draft creation** (BR-FIN-002). Allocating at draft time burns numbers on abandoned drafts, creating large gaps and encouraging reuse.

---

## 5. Atomic allocation

### 5.1 The mechanism

A single-row update on a counter table, executed inside the transaction that creates the document:

```
BEGIN;

-- 1. Ensure the series row exists for this period (idempotent insert)
INSERT INTO document_sequences (series_key, period_key, last_value)
VALUES ($1, $2, 0)
ON CONFLICT (series_key, period_key) DO NOTHING;

-- 2. Atomically claim the next value — single statement, no read-then-write
UPDATE document_sequences
   SET last_value = last_value + 1
 WHERE series_key = $1 AND period_key = $2
RETURNING last_value;          -- this is the allocated sequence number

-- 3. Render the human-readable number server-side from the series config
-- 4. Insert the document with its number and snapshot
-- 5. Write the allocation record and the audit event
-- 6. Render and store the PDF

COMMIT;
```

**Why `UPDATE … RETURNING` is correct:** the row lock is taken by the `UPDATE` itself. Concurrent transactions serialise on that row; each receives a distinct `last_value`. There is no window between reading and writing in which another transaction can interleave.

### 5.2 Prohibited patterns

These are prohibited without exception, and the prohibition is a hard invariant (see [`../../AGENTS.md`](../../AGENTS.md)):

| Prohibited | Why it fails |
|---|---|
| `SELECT MAX(number) + 1` | Two concurrent transactions read the same max |
| `SELECT last_value` then `UPDATE` in application code | Classic read-then-write race; the gap between statements is the bug |
| Generating the number in the frontend | Client state is not authoritative; refresh, double-submit, and two tabs all break it |
| Generating in a PDF template or report | Templates are presentation; they have no transactional context |
| Using a timestamp as the sequence (`YYYYMMDDHHMMSS`) | Collides under same-second issuance; not a counter |
| Using a random suffix as the only differentiator | Not human-verifiable; not sortable; still can collide |
| Application-level uniqueness check without a DB constraint | The check and the insert are not atomic; the constraint is the only guarantee |
| A separate "numbering service" without a transaction | Introduces a distributed boundary for no benefit |
| Reusing a voided number | Directly recreates the confirmed failure |

### 5.3 Database sequence alternative

A native PostgreSQL `SEQUENCE` per series is also acceptable and equally atomic:

```
CREATE SEQUENCE invoice_2026_seq START 1;
SELECT nextval('invoice_2026_seq');
```

| | Counter table + `UPDATE … RETURNING` | Native `SEQUENCE` |
|---|---|---|
| Atomicity | Yes | Yes |
| Transactional rollback of the counter | **Yes** — a rolled-back transaction returns the number | **No** — `nextval` is non-transactional; a rollback leaves a gap |
| Gap behaviour | No gap on rollback | Gap on every rollback |
| Multiple series | Rows, easy to add | One sequence object per series, created dynamically |
| Inspectable / adjustable | Yes, a normal table row | Harder to inspect and correct |
| Configuration in DB | Natural | Requires DDL per series |

**Recommendation:** counter table with `UPDATE … RETURNING` (ADR-002).

**Rationale:** the transactional property is decisive. With a native sequence, every failed invoice issuance permanently burns a number. In a business where issuance failures will happen (PDF errors, validation failures, network drops), this produces a sequence full of unexplained gaps — and gaps invite the "reuse the missing number" behaviour that caused the original problem. The counter table also keeps series configuration as data rather than schema.

**Note on the counter table's own gap behaviour:** if the transaction rolls back, the increment rolls back too, so the number is genuinely free again. This is correct — the document never existed, so the number was never issued.

### 5.4 Concurrency guarantees required

| Property | Requirement |
|---|---|
| Two concurrent issuances in one series | Two distinct numbers, both succeed |
| N concurrent issuances | N distinct numbers, zero constraint violations |
| Rollback | Number not consumed |
| Crash mid-transaction | Number not consumed; no orphan document |
| Different series concurrently | No contention between series |
| Period rollover at midnight | New period gets its own counter starting at 1 |

**Test obligation:** these are verified by parallel execution, not sequential (AC-DATA-002). A sequential test of a numbering bug passes trivially and proves nothing.

### 5.5 Period rollover

| Concern | Handling |
|---|---|
| Year boundary | New `period_key` → new row → counter starts at 1 |
| Clock dependency | Period derived from the **invoice date** (a business fact), not the server clock, where the user may backdate |
| Backdated issuance | Uses the period of the invoice date, consuming that period's counter — correct, because the number must match the document date |
| Future-dated issuance | **[Proposed]** blocked beyond a configurable window; prevents pre-consuming next year's numbers |

**Backdating is a real scenario.** An invoice for work completed in December may be issued in January with a December date. The number must belong to December's series or the document's own date contradicts its number — a discrepancy an auditor will question.

---

## 6. Uniqueness enforcement

### 6.1 Database constraints

| Constraint | Purpose |
|---|---|
| `UNIQUE (series_key, document_number)` on `issued_documents` | The hard guarantee |
| `UNIQUE (series_key, period_key)` on `document_sequences` | One counter per series per period |
| `UNIQUE (series_key, allocated_value, period_key)` on `document_number_allocations` | Prevents double allocation of one value |
| `CHECK (document_number IS NULL OR status <> 'DRAFT')` | A numbered document is never a draft |
| `CHECK (document_number IS NOT NULL OR status = 'DRAFT')` | An issued document always has a number |

### 6.2 Why the constraint is mandatory

Application-level checks are advisory. Two application processes can both pass the check and both insert. Only the database constraint is authoritative, and it must be present even if the application logic is believed correct — because the constraint is what catches the bug that the logic does not.

**Explicit statement:** frontend validation is never a control (BR-DOC-002). It is convenience for the user. The control is the server-side transaction plus the database constraint.

### 6.3 Historical numbers

Migrated numbers occupy the same uniqueness space but are flagged `is_historical_import = true`:

| Aspect | Behaviour |
|---|---|
| Uniqueness | Enforced — an imported number cannot collide with an issued one |
| Sequence consumption | **None** — imported numbers do not advance the counter |
| Verbatim preservation | Required (BR-DATA-001) — the original string is stored as-is |
| Format | May differ from the current pattern; that is acceptable and expected |
| Collision at import | Detected in the validation phase; commit is blocked until resolved (FR-DATA-004) |

**Migration collision policy:** if an imported historical number collides with an already-imported number, the import fails with both rows identified. The business must decide which is correct. The system must not silently pick one, and must not renumber to avoid the conflict — renumbering historical documents creates a mismatch with documents the customer already holds.

---

## 7. Cancellation, voiding, and correction

### 7.1 Rules

| Rule | Statement |
|---|---|
| V1 | A voided document keeps its number permanently |
| V2 | A voided number is never reissued to a different document |
| V3 | Voiding requires a reason and an authorised actor (FR-FIN-011) |
| V4 | The void is recorded in the audit log and the allocation record |
| V5 | The document remains retrievable, marked `VOID` |
| V6 | A replacement document gets a **new** number |
| V7 | Gaps caused by voids are expected and acceptable |

### 7.2 Void vs credit note

| | Void | Credit note |
|---|---|---|
| Use when | Document should never have been issued (wrong customer, duplicate, cancelled order) | Document was valid but its value must be reduced or reversed |
| Number effect | Original retained as `VOID` | Original becomes `CREDITED`; credit note has its own number |
| Financial effect | Document removed from AR/AP | Offsetting entry in AR/AP |
| Customer communication | "This invoice is cancelled" | "Here is a credit note against invoice X" |
| Typical choice | Administrative error | Commercial adjustment |

**Choosing the wrong one is a business error, not a system error**, but the system should guide: voiding an invoice that has payments against it requires resolving those payments first (FINANCE_AND_DOCUMENTS §8.4).

### 7.3 Replacement chain

```
INV/2026/00042  (issued) ──void──►  VOID
INV/2026/00043  (replacement, issued) ──►  ISSUED
```

or

```
INV/2026/00042  (issued) ──credit note──►  CREDITED
CN/2026/00007   (credit note)
INV/2026/00051  (replacement)
```

Both chains are fully traceable through `supersedes_document_id`, `replaced_by_invoice_id`, and `credit_note_for_invoice_id`.

---

## 8. Audit trail for numbering

Every allocation produces records in three places, each serving a different purpose:

| Record | Purpose | Content |
|---|---|---|
| `document_sequences` | Current counter state | Series, period, last value |
| `document_number_allocations` | Permanent allocation history | Which value, which document, when, by whom, current status |
| `audit_events` | Forensic trail | Actor, action, timestamp, before/after, correlation ID |

**Required audit events:**

| Event | When |
|---|---|
| `document_number.allocated` | At issuance |
| `document_number.voided` | At void |
| `document_number.superseded` | When replaced |
| `document_series.created` / `.modified` | Configuration change |
| `document_number.imported` | During historical migration |

**Series configuration changes are audited** because changing a format mid-year is a significant act: it can produce numbers that look inconsistent and can break client-side parsing. The audit record shows who changed the pattern and when, so a future inconsistency has an explanation.

---

## 9. Migration of historical numbers

| Step | Action |
|---|---|
| 1 | Extract all existing document numbers from spreadsheets and documents |
| 2 | Normalise for comparison only (trim, case-fold) — **never** for storage |
| 3 | Detect duplicates within the extracted set; report each conflict with source location |
| 4 | Detect collisions against numbers already imported |
| 5 | Present the conflict list to the business for resolution |
| 6 | Import with `is_historical_import = true`, verbatim strings preserved |
| 7 | Set each series counter to a value **above** the highest existing numeric suffix, where the format is parseable |
| 8 | Verify: no imported number can be produced by the allocator |

**Step 7 is essential.** If historical invoices ran to `INV/2025/0042` and the new system starts at 1, the first new invoice collides with an existing one — a self-inflicted recurrence of the original bug.

**Step 7 caveat:** if historical numbers use an irregular format (mixed schemes over the years), a numeric high-water mark may not be computable. In that case the series starts at a documented offset (e.g. 1000) and the reasoning is recorded in the migration report.

---

## 10. Testing obligations

| # | Test | Level | Proves |
|---|---|---|---|
| T1 | N parallel issuances in one series | Integration | No duplicates under concurrency |
| T2 | Concurrent issuances across different series | Integration | No cross-series contention |
| T3 | Rollback mid-issuance | Integration | Number not consumed |
| T4 | Attempt duplicate insert directly at DB level | Integration | Constraint rejects |
| T5 | Client-side tampering: POST a chosen number | Integration | Server ignores client-supplied numbers |
| T6 | Period rollover at year boundary | Integration | New counter, starts at 1 |
| T7 | Backdated invoice uses its date's period | Integration | Number matches document date |
| T8 | Void retains number; new document gets next number | Integration | No reuse |
| T9 | Historical import collision detection | Integration | Blocked before commit |
| T10 | Counter set above historical high-water mark | Integration | No post-migration collision |
| T11 | Format configuration change is audited | Integration | Traceability |
| T12 | Issuance failure (PDF error) leaves no numbered document | Integration | Atomicity across artefacts |
| T13 | Double-submit of issuance | E2E | Idempotency |

Acceptance criteria: AC-DATA-002, AC-DOC-004, AC-FIN-002, AC-FIN-011, AC-DATA-003, AC-DATA-004.

---

## 11. Open questions

| ID | Question | Proposed default | Trigger |
|---|---|---|---|
| Q-NUM-01 | Client-side format and validation requirements (§2.4, nine questions) | Internal-consistent configurable format; external constraints added per client | Before Phase 5 invoice go-live |
| Q-NUM-02 | Should the prefix include a company legal identifier (e.g. NPWP-derived)? | No — use a short company code | After accountant input |
| Q-NUM-03 | Reset period: yearly or monthly? | Yearly | Phase 5 |
| Q-NUM-04 | Should quotations share a series with invoices? | **No** — separate series per type, always | Resolved by FR-DOC-006; recorded for clarity |
| Q-NUM-05 | Must cancelled invoices be represented by a separate cancellation document number for any client? | Not unless a client requires it | Q-NUM-01 answer #7 |

---

## 12. Invariants (restated as hard rules)

These are non-negotiable and are repeated in [`../../AGENTS.md`](../../AGENTS.md):

1. **Numbers are generated only on the server, inside the issuing transaction.**
2. **No `MAX()+1`, no read-then-write, no client-side generation.**
3. **A database constraint enforces uniqueness — not application logic alone.**
4. **Issued numbers are immutable and never reused, including after void.**
5. **Numbers are allocated at issuance, never at draft creation.**
6. **Historical numbers are preserved verbatim and never renumbered.**
7. **Gaps are acceptable; reuse is not.**
8. **A company prefix does not guarantee external uniqueness** — external constraints require client confirmation.

---

## 13. Cross-references

- Requirement definitions: [`../PRD.md`](../PRD.md) §6.6, §11
- Entities: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §8
- Issuance flow: [`../workflows/FINANCE_AND_DOCUMENTS.md`](../workflows/FINANCE_AND_DOCUMENTS.md) §4, §8
- Decision: [`../decisions/ADR-002-document-number-allocation.md`](../decisions/ADR-002-document-number-allocation.md)
- Tests: [`../implementation/TESTING_STRATEGY.md`](../implementation/TESTING_STRATEGY.md) §4
