# ADR-007 — Spreadsheet migration approach

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-25 |
| Deciders | Business owner, operations administrator, technical lead |
| Related | [`../PRD.md`](../PRD.md) §6.8, [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §9 |

---

## Context

**[Confirmed]** Administration currently relies on Excel/Google Sheets, Word, and WhatsApp. There is no centralised inventory; materials are tracked per project.

**[Open Question] Q-DATA-01** — the actual spreadsheet structures, column consistency, and duplication level are **unknown**. No sample has been profiled.

Migration must achieve three things:

1. Master data available so the system is usable from day one.
2. Open items (active projects, outstanding invoices and payables) carried over so nothing is lost.
3. Historical document numbers preserved so previously issued documents remain recognisable — while not colliding with newly allocated numbers.

Two failure modes are equally bad:

- **Migrating garbage**, which makes the new system untrustworthy immediately and validates the "the old spreadsheet was better" narrative.
- **Refusing to migrate anything**, which forces staff to maintain two systems and guarantees neither is authoritative.

---

## Decision

### D1 — Migrate master data and open items; do not migrate closed history wholesale

| Migrate | Do not migrate |
|---|---|
| Customers and contacts (deduplicated) | Closed projects with no financial exposure |
| Suppliers | Historical daily reports |
| Technicians and user accounts | Historical material movements |
| Item catalogue | Closed quotations |
| Service categories | Old expenses already settled and reported |
| Active projects with current state | — |
| Outstanding invoices and payables | — |
| Historical document numbers **as references** | — |

**Rationale:** the system needs enough history to operate, not a complete archive. Migrating every closed project creates enormous cleanup work with no operational benefit, and each migrated record is a potential inconsistency. Closed history remains available in the source spreadsheets, which are retained as an archive.

### D2 — Two-step import: validate, then commit

FR-DATA-002. Validation writes nothing. It produces a per-row report naming row, column, and reason. Commit runs only after a clean validation.

### D3 — Historical document numbers preserved verbatim, flagged, excluded from counters

Imported numbers are stored exactly as they appear, flagged `is_historical_import = true`, and never advance a series counter. Counters are initialised **above** the historical high-water mark (DOCUMENT_NUMBERING §9).

### D4 — Collisions block commit; the business decides

If two imported rows carry the same number, or an imported number would collide with an allocatable one, the import fails with both rows identified. The system never silently picks one and never renumbers.

**Renumbering historical documents is prohibited.** The customer holds a copy with the original number; changing our record creates a mismatch that cannot be explained later.

### D5 — Profiling precedes importer construction

No importer is built until Phase 0 produces a mapping specification from real samples. Building importers against imagined schemas is the most common cause of failed migrations.

### D6 — Spreadsheets are frozen at cutover, not kept in parallel

An explicit cutover date is agreed, after which spreadsheet entry stops. Parallel operation produces two sources of truth and is the single most reliable way to make a migration fail quietly.

---

## Consequences

### Positive

- **Bounded effort.** Migrating master data plus open items is days of work; migrating all history is weeks with high defect rates.
- **Numbering safety.** Verbatim preservation plus high-water-mark initialisation removes the most dangerous migration failure.
- **Quality gate.** Validation before commit means malformed data is caught while the source is still available.
- **Explicit cutover** prevents the two-systems limbo that dooms adoption.
- **Auditability.** The import batch itself is recorded (FR-DATA-001, `import_batches` status).

### Negative

- **Incomplete history in the new system.** Reports spanning pre-migration periods will be incomplete. Mitigated by retaining source spreadsheets as a reference archive and documenting the cutover date.
- **Cleanup effort falls on the business.** Deduplication of customers and items requires human judgement; the system can only report candidates.
- **High-water-mark initialisation can fail** if historical formats are irregular. Mitigated by a documented fallback offset (e.g. start at 1000) with the reasoning recorded.
- **Cutover risk.** If the new system has a defect in week one, the frozen spreadsheets are the fallback — which means the cutover must follow a pilot, not precede it.

### Neutral

- Spreadsheets remain the legal/archival record for pre-migration periods. The system does not claim to replace them retroactively.

---

## Alternatives considered

### Alternative A — Migrate all historical data

**Rejected for MVP.** Weeks of effort, high defect rate, and no operational benefit: nobody needs a closed 2023 project's daily reports inside the new system. The cost is real; the value is speculative.

### Alternative B — Start clean with no migration

**Rejected.** Staff would maintain spreadsheets for open work indefinitely, and the system would be untrustworthy on day one for any customer with an outstanding invoice. Two sources of truth is worse than an imperfect migration.

### Alternative C — Renumber historical documents into the new series

**Rejected outright.** The customer holds the original number. Renumbering breaks the correspondence between our records and theirs, which is precisely the class of problem the project exists to eliminate.

### Alternative D — Direct database import bypassing validation

**Rejected.** Skips the only quality gate, and produces failures that surface weeks later as wrong balances. Validation cost is small; the cost of silent bad data is not.

### Alternative E — Parallel running for a month

**Rejected.** Parallel operation doubles staff workload, and in practice the spreadsheet stays authoritative because it is familiar. The correct sequence is pilot → cutover, not indefinite duplication.

### Alternative F — Automated fuzzy deduplication

**Rejected for automatic application.** Similar-name matching produces false merges that destroy distinct customers. The system reports candidates; a human decides.

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Import malformed CSV | Every bad row reported with reason — AC-DATA-001 |
| V2 | Dry-run only | Zero rows written — AC-DATA-002 |
| V3 | Duplicate historical numbers | Commit blocked, both rows identified — AC-DATA-003 |
| V4 | Imported number matching allocatable number | Blocked; counter adjusted — AC-DATA-004 |
| V5 | Counter initialised above high-water mark | First new document does not collide — AC-DATA-004 |
| V6 | Imported numbers re-downloaded | Identical strings to source — AC-DATA-003 |

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-DATA-01 | Actual spreadsheet structures and data quality | Phase 0 profiling produces the mapping spec |
| Q-DATA-02 | Are historical formats regular enough to compute a high-water mark? | If not, start series at a documented offset |
| Q-DATA-03 | How far back must outstanding items be migrated? | All items with any financial exposure |
| Q-DATA-04 | Who owns data-quality decisions during migration? | A single named person (A-07) |
| Q-DATA-05 | Must historical documents be reprintable as PDFs? | No; only new documents generate PDFs. Historical scans may be attached |
