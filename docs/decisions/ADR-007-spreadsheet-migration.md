# ADR-007 — Spreadsheet migration approach

| Field | Value |
|---|---|
| Status | **Accepted (design) — verified against the reconciled baseline 2026-10-25, no change required** |
| Date | 2026-10-25 |
| Revision | Verified only |
| Deciders | Business owner, operations administrator, technical lead |
| Related | [`../PRD.md`](../PRD.md) §6.8, [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §9 |

> **Verification note.** Reviewed against the reconciled baseline (Laravel 13, MySQL/MariaDB, four roles, local-first storage, mandatory purchase approval). **No substantive change was required** — migration is about data semantics, not the engine or framework, and none of the six decisions depends on a superseded mechanism. Two clarifications were added, marked below.

---

## Context

**[Confirmed]** Administration relies on Excel/Google Sheets, Word, and WhatsApp. There is no centralised inventory; materials are tracked per project.

**[Open Question] Q-DATA-01** — the actual spreadsheet structures, column consistency, and duplication level are **unknown**. No sample has been profiled.

Migration must achieve three things: master data available so the system is usable on day one; open items (active projects, outstanding invoices and payables) carried over; and historical document numbers preserved so previously issued documents remain recognisable without colliding with newly allocated numbers.

Two failure modes are equally bad: **migrating garbage**, which makes the new system untrustworthy immediately; and **refusing to migrate**, which forces staff to maintain two systems and guarantees neither is authoritative.

---

## Decision

### D1 — Migrate master data and open items; not closed history wholesale

| Migrate | Do not migrate |
|---|---|
| Customers and contacts (deduplicated) | Closed projects with no financial exposure |
| Suppliers, technicians and user accounts | Historical daily reports |
| Item catalogue, service categories | Historical material movements |
| Active projects with current state | Closed quotations |
| Outstanding invoices and payables | Old expenses already settled and reported |
| Historical document numbers **as references** | — |

**Rationale:** the system needs enough history to operate, not a complete archive. Migrating every closed project creates enormous cleanup work with no operational benefit, and each migrated record is a potential inconsistency. Closed history remains in the source spreadsheets, retained as an archive.

### D2 — Two-step import: validate, then commit

FR-DATA-002. Validation writes nothing and produces a per-row report naming row, column, and reason. Commit runs only after a clean validation.

### D3 — Historical numbers preserved verbatim, flagged, excluded from counters

Imported numbers are stored exactly as they appear, flagged `is_historical_import = true`, and never advance a counter. Counters are initialised **above** the historical high-water mark.

*Clarification (2026-10-25):* the high-water mark is stored explicitly in `number_series.high_water_mark` and the counter may never be lowered below it — a `CHECK` constraint plus configuration validation, not an operational convention (ADR-009 D6).

### D4 — Collisions block commit; the business decides

If two imported rows carry the same number, or an imported number would collide with an allocatable one, the import fails with both rows identified. The system never silently picks one and never renumbers.

**Renumbering historical documents is prohibited.** The customer holds a copy with the original number; changing our record creates a mismatch that cannot be explained later.

### D5 — Profiling precedes importer construction

No importer is built until Phase 0 produces a mapping specification from real samples. Building importers against imagined schemas is the most common cause of failed migrations.

### D6 — Spreadsheets are frozen at cutover, not kept in parallel

An explicit cutover date is agreed, after which spreadsheet entry stops. Parallel operation produces two sources of truth and is the most reliable way to make a migration fail quietly.

---

## Consequences

### Positive

- **Bounded effort** — master data plus open items is days of work; all history is weeks with high defect rates.
- **Numbering safety** — verbatim preservation plus high-water-mark initialisation removes the most dangerous migration failure.
- **Quality gate** — malformed data is caught while the source is still available.
- **Explicit cutover** prevents the two-systems limbo that dooms adoption.
- **Auditability** — the import batch itself is recorded.

### Negative

- **Incomplete history in the new system.** Reports spanning pre-migration periods will be incomplete. Mitigated by retaining the source spreadsheets and documenting the cutover date.
- **Cleanup effort falls on the business.** Deduplication requires human judgement; the system only reports candidates.
- **High-water-mark initialisation can fail** if historical formats are irregular. Mitigated by a documented fallback offset (e.g. start at 1000) with the reasoning recorded.
- **Cutover risk.** If the new system has a defect in week one, the frozen spreadsheets are the fallback — so cutover must follow a pilot, not precede it.
- **Historical scans are files too.** Any scanned document attached during migration lands on the private disk, so the archive is only as good as the file backup (ADR-006 D9).

### Neutral

- Spreadsheets remain the legal/archival record for pre-migration periods. The system does not claim to replace them retroactively.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Migrate all historical data | **Rejected for MVP** — weeks of effort, high defect rate, no operational benefit: nobody needs a closed 2023 project's daily reports inside the new system |
| Start clean with no migration | **Rejected** — staff would maintain spreadsheets for open work indefinitely, and the system would be untrustworthy on day one for any customer with an outstanding invoice |
| Renumber historical documents into the new series | **Rejected outright** — the customer holds the original number; renumbering breaks the correspondence, which is precisely the class of problem the project exists to eliminate |
| Direct database import bypassing validation | **Rejected** — skips the only quality gate; failures surface weeks later as wrong balances |
| Parallel running for a month | **Rejected** — doubles staff workload, and the spreadsheet stays authoritative because it is familiar. The correct sequence is pilot → cutover |
| Automated fuzzy deduplication | **Rejected for automatic application** — similar-name matching produces false merges that destroy distinct customers. The system reports candidates; a human decides |
| Import historical documents as new numbered documents in the current series | **Rejected** — would consume current-period numbers for historical work, produce numbers the customer has never seen, and create exactly the mismatch D4 forbids |

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Import malformed CSV | Every bad row reported with reason — AC-DATA-001 |
| V2 | Dry-run only | Zero rows written — AC-DATA-002 |
| V3 | Duplicate historical numbers | Commit blocked, both rows identified — AC-DATA-003 |
| V4 | Imported number matching an allocatable number | Blocked; counter adjusted — AC-DATA-004 |
| V5 | Counter initialised above the high-water mark | First new document does not collide — AC-DATA-004 |
| V6 | Imported numbers re-downloaded | Identical strings to source — AC-DATA-003 |
| V7 | Starting-number change after import | Refused if it could collide with an imported number — AC-SYS-004 |

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-DATA-01 | Actual spreadsheet structures and data quality | Phase 0 profiling produces the mapping spec |
| Q-DATA-02 | Are historical formats regular enough to compute a high-water mark? | If not, start the series at a documented offset |
| Q-DATA-03 | How far back must outstanding items be migrated? | All items with any financial exposure |
| Q-DATA-04 | Who owns data-quality decisions during migration? | A single named person (A-07) |
| Q-DATA-05 | Must historical documents be reprintable as PDFs? | No; only new documents generate PDFs. Historical scans may be attached |

---

## Cross-references

- Requirements: [`../PRD.md`](../PRD.md) §6.8
- Numbering: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §9
- Allocation: [`./ADR-009-mysql-document-number-allocation.md`](./ADR-009-mysql-document-number-allocation.md)
- Storage of scanned historical files: [`./ADR-006-private-file-storage.md`](./ADR-006-private-file-storage.md)
- Phase 6: [`../implementation/IMPLEMENTATION_PLAN.md`](../implementation/IMPLEMENTATION_PLAN.md) §9
