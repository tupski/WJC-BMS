# Acceptance Criteria — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed — nothing has been executed |
| Version | 0.1 |
| Scope owner | This document is the **only** register of `AC-` identifiers |
| Requirement source | [`../PRD.md`](../PRD.md) |
| Test approach | [`./TESTING_STRATEGY.md`](./TESTING_STRATEGY.md) |

---

## 1. How to read this register

Each entry states: what is being verified, at which test level, which requirement it covers, the expected outcome, and — critically — the **failure behaviour** where one is specified. A criterion without a failure behaviour is a criterion that only tests the happy path.

**Test levels:** `U` unit · `I` integration · `E2E` end-to-end · `M` manual/UAT.

**Status column:** all entries are `Defined`. None is `Passing`, because no code exists. No criterion in this document has been executed.

### 1.1 Verification standard

A criterion is met only when the assertion is made **at the level where the property lives**:

| Property type | Must assert at |
|---|---|
| Uniqueness, constraints, concurrency | `I` against a real database |
| Authorisation | `E2E` with a direct request, not a hidden control |
| Immutability | `I` by attempting mutation through every path |
| Audit completeness | `I` by reading the audit table |
| Usability, adoption | `M` with real users on real devices |

**A passing UI interaction does not satisfy any criterion in this register.**

---

## 2. Customer (`AC-CUST`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-CUST-001 | Customer master record with required fields; referenced records deactivatable, not deletable | I | FR-CUST-001 | Record created; deactivation preserves all references | Hard delete of referenced customer refused by database |
| AC-CUST-002 | Multiple contacts with exactly one primary | I | FR-CUST-002 | Contacts created; primary flag set | Second primary for same customer rejected by partial unique index |
| AC-CUST-003 | Service categories are configurable data | I | FR-CUST-003 | New category added without code change; customer linked to multiple | Removing a category in use is refused |
| AC-CUST-004 | Optional customer PO reference | E2E | FR-CUST-004 | Project with PO reference saves; project without one also saves | Neither path blocks later steps |
| AC-CUST-005 | Customer 360 view | E2E | FR-CUST-005 | Projects, invoices, outstanding balance, document history all shown for that customer only | No cross-customer data leakage |
| AC-CUST-006 | Duplicate warning on create | E2E | FR-CUST-006 | Similar name or identical phone triggers warning requiring confirmation | Creation proceeds on explicit confirmation; never silently blocked |

---

## 3. Project (`AC-PROJ`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-PROJ-001 | Minimal project completes full lifecycle | E2E | FR-PROJ-001, BR-PROJ-001 | Project with no quotation, no customer PO, no material lines, one technician reaches `CLOSED` | No screen forces an optional step |
| AC-PROJ-002 | Service category drives no CCTV assumptions | E2E | FR-PROJ-002 | Non-CCTV category project completes with no camera-specific field | Completion is not gated on CCTV data |
| AC-PROJ-003 | Quotation is optional | E2E | FR-PROJ-003 | Project without quotation proceeds through all states | No warning wall, no forced reason |
| AC-PROJ-004 | Customer PO is optional | E2E | FR-PROJ-004 | Project without customer PO proceeds | Same |
| AC-PROJ-005 | Schedule revision is recorded, not overwritten | I | FR-PROJ-005 | Two revisions produce two rows; original values intact | Revision without reason refused |
| AC-PROJ-006 | Multi-day progress is not binary | E2E | FR-PROJ-006 | 5-day project yields 1 work order + 5 reports; progress accumulates | No single flag marks the project complete mid-way |
| AC-PROJ-007 | One-or-many technicians; at most one team leader | I | FR-PROJ-007 | 3 assignments possible; 1 leader | Second active leader rejected by constraint |
| AC-PROJ-008 | Optional budget with actual-vs-budget display | E2E | FR-PROJ-008 | Budget set → comparison shown; budget absent → no error | Missing budget never blocks any transition |
| AC-PROJ-009 | Material requirements derived and exposed | E2E | FR-PROJ-009 | Requirement lines visible from project; readiness shown | — |
| AC-PROJ-010 | Closure refused with enumerated unmet conditions | E2E | FR-PROJ-010, BR-PROJ-004 | Closure attempted with open work order lists that condition specifically | Closure does not partially apply |
| AC-PROJ-011 | Illegal transitions refused, permitted options named | E2E | FR-PROJ-011 | `DRAFT`→`IN_PROGRESS` refused with current status and allowed next statuses | No status change occurs |
| AC-PROJ-012 | Cancel/suspend retains all records | E2E | FR-PROJ-012, BR-PROJ-005 | Cancelled project keeps invoices, POs, payables, material records | Cancellation never deletes financial records |

---

## 4. Procurement (`AC-PROC`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-PROC-001 | Purchase request from material shortfall | I | FR-PROC-001 | Request lines link to requirement lines; may span projects | Request without any line refused |
| AC-PROC-002 | Approval is a decision, not a purchase | I | FR-PROC-002, BR-PROC-001 | `APPROVED` records actor, time, reason, policy snapshot; stock unchanged; no PO created automatically | Rejected request cannot become a PO |
| AC-PROC-003 | Approval policy configurable; fail-closed | I | FR-PROC-003 | Below-threshold request proceeds without approval; above-threshold requires it; no policy → approval required | Misconfiguration never permits unapproved spending |
| AC-PROC-004 | Supplier master with soft deactivation | I | FR-PROC-004 | Supplier deactivated; historical POs intact | Deleting a referenced supplier refused |
| AC-PROC-005 | **PO issuance does not increase stock** | I | FR-PROC-005, BR-MATL-005 | Available quantity identical before and after PO issuance | — |
| AC-PROC-006 | PO issuance allocates number, freezes snapshot, stores PDF | I | FR-PROC-006 | Number unique; snapshot immutable; PDF hash recorded | Failure in any step rolls back the whole issuance |
| AC-PROC-007 | Partial goods receipt accumulates correctly | I | FR-PROC-007 | Three receipts sum to ordered quantity; status transitions `ISSUED`→`PARTIALLY_RECEIVED`→`RECEIVED` | Over-receipt refused without approval; cumulative never exceeds ordered |
| AC-PROC-008 | Discrepancies recorded and block completion | I | FR-PROC-008 | Short/damaged/wrong/price-variance recorded; line not `RECEIVED` until resolved | Unresolved discrepancy prevents PO closure |
| AC-PROC-009 | Supplier invoice recorded as payable | I | FR-PROC-009 | Payable created with due date and amount; may exist before or after delivery | Payable for a cancelled PO refused |
| AC-PROC-010 | Payment recorded with partial support; no money moves | I | FR-PROC-010, BR-PROC-004 | Partial payment leaves correct outstanding; no banking call exists | System never reports having executed a transfer |

---

## 5. Materials (`AC-MATL`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-MATL-001 | Requirement lines independent of stock | I | FR-MATL-001 | Lines created with quantity and unit regardless of availability | — |
| AC-MATL-002 | New line defaults to `UNKNOWN` | I | FR-MATL-002, BR-MATL-001 | `stock_status = UNKNOWN` on creation | An `UNKNOWN` line cannot count toward readiness |
| AC-MATL-003 | Verification records observed quantity and history | I | FR-MATL-003 | Status `VERIFIED`; verification row appended; re-verification adds a row | Verification without observed quantity refused |
| AC-MATL-004 | Reservation lifecycle create/release/consume | I | FR-MATL-004 | Each transition appends a movement or state record; release restores availability | Direct deletion of a reservation impossible |
| AC-MATL-005 | **Reservation on `UNKNOWN` line refused** | I | FR-MATL-005 | Refusal names the blocking condition and required action | No reservation row created |
| AC-MATL-006 | **Concurrent reservations cannot exceed verified stock** | I | FR-MATL-006, BR-MATL-002 | 2 parallel requests for last 10 units → exactly 1 success | Second refused with available quantity shown |
| AC-MATL-007 | Shortfall computed and exposed to procurement | I | FR-MATL-007 | `shortfall = required − covered` matches hand calculation across scenarios | Competing reservations reduce `covered` |
| AC-MATL-008 | Issue to named technician | I | FR-MATL-008 | `ISSUE_OUT` movement; availability reduced; custody recorded | Issue exceeding reservation re-checks availability |
| AC-MATL-009 | Consumption recorded | I | FR-MATL-009 | `CONSUME` movement; issue line updated | Consuming more than issued refused by CHECK |
| AC-MATL-010 | Return restores availability | I | FR-MATL-010, BR-MATL-004 | `RETURN_IN` movement; availability restored; issue line updated | `consumed+returned+written_off ≤ issued` always holds |
| AC-MATL-011 | Write-off visible with reason | I | FR-MATL-011 | `WRITE_OFF` movement with reason code; quantity still appears in reports | Write-off never removes the quantity from history |
| AC-MATL-012 | No direct quantity edit path exists | I | FR-MATL-012, BR-MATL-003 | No endpoint or UI edits a quantity; corrections create compensating movements | Direct SQL update finds no writable balance column |
| AC-MATL-013 | Reconciliation detects variance | I | FR-MATL-013 | Injected ledger/cache divergence is reported with the variance | Reconciliation never auto-corrects |

---

## 6. Technician operations (`AC-TECH`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-TECH-001 | Technician linked to exactly one user | I | FR-TECH-001 | One-to-one constraint holds | Second technician record for same user refused |
| AC-TECH-002 | Team leader scoped to project | E2E | FR-TECH-002 | Leader sees that project's team work orders only | Leader sees no financial data |
| AC-TECH-003 | One work order per technician per project | I | FR-TECH-003 | Three technicians → three work orders | Duplicate assignment for same technician/project prevented |
| AC-TECH-004 | Daily report in < 3 minutes at 360 px | M | FR-TECH-004, M3 | Median of 5 attempts under target with 3 technicians | Data preserved on connection drop; double-submit idempotent |
| AC-TECH-005 | Photo upload with server-side limits | I | FR-TECH-005 | Oversized/wrong-type file rejected server-side; valid file stored privately | Photo failure does not block report submission |
| AC-TECH-006 | Blocker recorded and surfaced | E2E | FR-TECH-006 | Work order `BLOCKED`; appears in office blocked list; silence detected after one day | Blocker requires a reason code |
| AC-TECH-007 | Reschedule/reassign recorded with reason | I | FR-TECH-007 | Previous values preserved; actor and timestamp recorded | Reassignment does **not** auto-transfer material custody |
| AC-TECH-008 | Partial completion and rework rejection | E2E | FR-TECH-008, BR-TECH-002 | One of three verified → project stays `IN_PROGRESS`; rejection returns to `IN_PROGRESS` with reason | Self-verification refused without override |
| AC-TECH-009 | Technician sees only own assignments | E2E | FR-TECH-009, NFR-SEC-006 | Direct request for another's work order returns 403/404 without revealing existence | No finance, supplier, or peer data in any response |
| AC-TECH-010 | Progress presented as estimate | E2E | FR-TECH-010 | Displayed as estimate with last-updated timestamp | Never labelled as authoritative completion |

---

## 7. Finance (`AC-FIN`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-FIN-001 | Quotation issued with number and PDF | I | FR-FIN-001 | Number allocated at issuance; PDF stored; no revenue effect | Draft has no number |
| AC-FIN-002 | Invoice issued with allocated number | I | FR-FIN-002 | Unique number; snapshot; PDF; audit — all in one transaction | Any step failing leaves no invoice and no consumed number |
| AC-FIN-003 | **Issued invoice immutable after master-data edit** | I | FR-FIN-003, M4 | Rename customer + change price → re-download hash identical | No code path edits an issued invoice |
| AC-FIN-004 | Partial payments and accurate outstanding | I | FR-FIN-004, BR-FIN-003 | Two partial payments → `PAID`, outstanding 0; allocation never exceeds balance | Overpayment becomes unapplied credit, never negative balance |
| AC-FIN-005 | Overdue derived from due date | U | FR-FIN-005 | Bucket correct day before, on, and after due date | No stored `OVERDUE` flag can go stale |
| AC-FIN-006 | Expenses recorded with attribution | I | FR-FIN-006 | Project expense enters project cost; operational expense does not | Expense without category refused |
| AC-FIN-007 | **Project cost not double counted** | I | FR-FIN-007, BR-FIN-005 | Receipt + invoice + payment for one purchase counts **once**; composition report shows source | Unique `(project, source_type, source_id)` prevents duplicates |
| AC-FIN-008 | Due dates and overdue items surfaced | E2E | FR-FIN-008 | Days-overdue computed and displayed | — |
| AC-FIN-009 | **Payment recorded ≠ confirmed** | E2E | FR-FIN-009, BR-PROC-002 | Distinct states, distinct actors, distinct reports | Confirmation by the recorder requires override |
| AC-FIN-010 | Payables/receivables summaries | E2E | FR-FIN-010 | Outstanding per counterparty matches allocation arithmetic | — |
| AC-FIN-011 | Void retains number and record | I | FR-FIN-011, BR-FIN-002 | Status `VOID`; number never reused; PDF retrievable | Void without reason refused; void with unresolved payments refused |
| AC-FIN-012 | Credit note and replacement preserve original | I | FR-FIN-012 | Original `CREDITED`; both new documents numbered; chain traceable | Original bytes never modified |

---

## 8. Documents (`AC-DOC`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-DOC-001 | Private storage, signed URLs only | I | FR-DOC-001 | Unauthenticated direct URL fails; signed URL expires | No permanent public URL exists anywhere |
| AC-DOC-002 | Document metadata and history | I | FR-DOC-002 | Type, owner, number, status, timestamps, history present | — |
| AC-DOC-003 | PDF generation for all issued types | I | FR-DOC-003 | Quotation, PO, invoice, credit note, receipt each render and store | Generation failure rolls back issuance |
| AC-DOC-004 | **Server-side number allocation only** | I | FR-DOC-004, BR-DOC-002 | Client-supplied number ignored; server allocates | No template or report can produce a number |
| AC-DOC-005 | Retrieval by number with full history | E2E | FR-DOC-005 | Document, snapshot, PDF, related documents, timeline all shown | Ambiguity across types resolved by showing type |
| AC-DOC-006 | Document type registry | I | FR-DOC-006 | New type added as data; format change audited | Existing numbers unaffected by config change |
| AC-DOC-007 | Attachments inherit owner access rules | I | FR-DOC-007 | Attachment reachable only by users who can read its owner | Technician cannot fetch an invoice attachment |
| AC-DOC-008 | Issued PDF immutable | I | FR-DOC-008 | Re-download hash matches original after template change | Regeneration never replaces original bytes |

---

## 9. Reporting (`AC-RPT`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-RPT-001 | Daily progress per project/technician, phone-readable | E2E | FR-RPT-001 | Renders at 360 px from submitted reports; exportable | Technician view shows only own data |
| AC-RPT-002 | Project dashboard with six indicators | E2E | FR-RPT-002 | Status, readiness, schedule variance, cost, billed, outstanding all correct | Readiness shown separately from status |
| AC-RPT-003 | Material status report per line | I | FR-RPT-003 | Required/reserved/purchased/received/issued/consumed/returned/remaining reconcile | Arithmetic matches ledger |
| AC-RPT-004 | AR/AP ageing by counterparty and bucket | I | FR-RPT-004 | Buckets correct at boundaries | Disputed payables excluded from due-now but still visible |
| AC-RPT-005 | Procurement pipeline by state | I | FR-RPT-005 | Every PO appears in exactly one state bucket | Partially received shown as such, not as received |
| AC-RPT-006 | Technician workload | E2E | FR-RPT-006 | Assignments, scheduled days, open and blocked counts correct | — |
| AC-RPT-007 | Date range and export | E2E | FR-RPT-007 | CSV for all reports; PDF for external documents | Export audited |
| AC-RPT-008 | Report reproducibility | I | FR-RPT-008 | Re-running a past range yields identical figures unless corrections occurred | Corrections visible in audit trail |

---

## 10. Data (`AC-DATA`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-DATA-001 | CSV import with per-row rejection report | I | FR-DATA-001 | Every malformed row reported with row, column, reason | Valid rows not partially committed during validation |
| AC-DATA-002 | **Dry-run then explicit commit** | I | FR-DATA-002, NFR-DATA-002 | Dry-run writes nothing; commit writes only after successful validation | Concurrency: N parallel issuances → N unique numbers, 0 violations |
| AC-DATA-003 | Historical numbers preserved verbatim | I | FR-DATA-003, BR-DATA-001 | Imported strings identical to source; flagged historical | Never renumbered to resolve a conflict |
| AC-DATA-004 | **Imported numbers cannot collide with new numbers** | I | FR-DATA-004 | Collision detected pre-commit; counter set above high-water mark | Import blocked when collision unresolvable |
| AC-DATA-005 | Soft deactivation; delete refused | I | FR-DATA-005, BR-DATA-002 | Deactivation succeeds; deletion of referenced record refused by FK | — |
| AC-DATA-006 | Full export | I | FR-DATA-006 | All tables plus files exported; re-importable | — |
| AC-DATA-007 | Safe under ≥10 concurrent state-changing users | I | NFR-DATA-007 | No errors, no constraint violations, no lost updates | — |
| AC-DATA-008 | Retention configuration applied | I | NFR-DATA-008 | Retention policy configurable; archival does not delete | — |

---

## 11. Audit (`AC-AUD`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-AUD-001 | Audit record for every state transition | I | FR-AUD-001, NFR-AUD-002 | Every transition in every state machine produces a record with actor, action, before/after | Missing record fails the test |
| AC-AUD-002 | **Append-only enforced at database level** | I | FR-AUD-002, NFR-AUD-001 | `UPDATE` and `DELETE` as application role both fail | Privilege restoration by migration caught by test |
| AC-AUD-003 | **Transactional coupling; hash chain intact** | I | FR-AUD-003, NFR-AUD-003 | Forced audit failure rolls back the state change; altered record breaks chain verification | No committed change without its audit record |
| AC-AUD-004 | Override requires reason and is visible to owner | E2E | FR-AUD-004, NFR-AUD-004 | Reason mandatory; appears in owner override report | Override never bypasses a data-integrity invariant |
| AC-AUD-005 | Retention and archival without loss | I | FR-AUD-006, NFR-AUD-005 | Archived records remain verifiable | Routine job never deletes |
| AC-AUD-006 | Restricted access; viewing is audited | E2E | FR-AUD-005, NFR-AUD-006 | Owner and `audit.view` only; every view audited | Technician and ops admin receive 403 |

---

## 12. Security (`AC-SEC`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-SEC-001 | No anonymous business data access | E2E | NFR-SEC-001 | Every business endpoint returns 401 without a session | Login and health endpoints excepted |
| AC-SEC-002 | **Server-side authorisation per action and resource** | E2E | NFR-SEC-002 | Direct unauthorised requests return 403 regardless of UI | Denial itself audited for sensitive actions |
| AC-SEC-003 | Multi-role union with conflict controls | E2E | NFR-SEC-003 | Combined roles grant union; conflicting pairs still require override | No implicit escalation from combining roles |
| AC-SEC-004 | Session expiry and revocation | E2E | NFR-SEC-004 | Idle expiry enforced; password change revokes other sessions | — |
| AC-SEC-005 | Password hashing and policy | I | NFR-SEC-005 | Slow salted hash; minimum length; no plaintext anywhere | Weak password rejected |
| AC-SEC-006 | Technician blocked from financial endpoints | E2E | NFR-SEC-006 | 403 on every finance, supplier-payment, and company-report endpoint | Verified per release, not once |
| AC-SEC-007 | No secrets in repo or logs | M | NFR-SEC-007 | Secret scan clean; logs redact sensitive fields | — |
| AC-SEC-008 | TLS enforced in production | M | NFR-SEC-008 | Plain HTTP redirected or rejected | — |

---

## 13. Operations (`AC-OPS`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-OPS-001 | Environment separation | M | NFR-OPS-001 | Dev/staging/prod isolated; no live personal data below prod | — |
| AC-OPS-002 | Diagnosable logs | M | NFR-OPS-002 | A reported failure is diagnosable from logs without reproduction | — |
| AC-OPS-003 | Mobile-first at 360 px | M | NFR-OPS-003 | All technician flows usable; targets ≥44 px; no horizontal scroll | — |
| AC-OPS-004 | Performance targets | M | NFR-OPS-004 | Technician p95 < 2 s on 3G-class; office p95 < 1 s on LAN | — |
| AC-OPS-005 | Availability 99% business hours | M | NFR-OPS-005 | Uptime monitoring confirms; maintenance announced | — |
| AC-OPS-006 | Indonesian, actionable error messages | E2E | NFR-OPS-006 | Message states what failed and what to do; no stack trace | — |
| AC-OPS-007 | UI primary language Indonesian | M | NFR-OPS-007 | Navigation, labels, statuses, PDFs in Indonesian | — |
| AC-OPS-008 | Browser matrix | M | NFR-OPS-008 | Current Chrome, Safari iOS, Firefox, Edge all functional | — |

---

## 14. Cross-cutting acceptance gates

Phase cannot exit until these hold. They are the criteria that protect the two highest-severity properties.

| Gate | Criteria | Why it blocks |
|---|---|---|
| G1 — Numbering proven | AC-DATA-002, AC-DOC-004, AC-FIN-002, AC-FIN-011 | Issuing documents on unproven numbering repeats the original defect |
| G2 — Audit proven | AC-AUD-001, AC-AUD-002, AC-AUD-003 | Approvals without audit are unattributable forever |
| G3 — Authorisation proven | AC-SEC-002, AC-SEC-006, AC-TECH-009 | A leak found after go-live cannot be un-leaked |
| G4 — Immutability proven | AC-FIN-003, AC-DOC-008 | Mutated historical documents destroy dispute defensibility |
| G5 — Material integrity proven | AC-MATL-005, AC-MATL-006, AC-PROC-005 | Double-promising causes on-site failure |
| G6 — Cost integrity proven | AC-FIN-007 | Double-counted cost drives wrong pricing decisions |
| G7 — Recovery proven | AC-DATA-004, AC-DATA-005 | An untested backup is not a backup |

---

## 15. Status summary

| Area | Criteria defined | Executed | Passing |
|---|---|---|---|
| Customer | 6 | 0 | 0 |
| Project | 12 | 0 | 0 |
| Procurement | 10 | 0 | 0 |
| Materials | 13 | 0 | 0 |
| Technician | 10 | 0 | 0 |
| Finance | 12 | 0 | 0 |
| Documents | 8 | 0 | 0 |
| Reporting | 8 | 0 | 0 |
| Data | 8 | 0 | 0 |
| Audit | 6 | 0 | 0 |
| Security | 8 | 0 | 0 |
| Operations | 8 | 0 | 0 |
| **Total** | **109** | **0** | **0** |

**No criterion has been executed.** There is no application code, no test framework, and no build system in this repository. This table records definitions only; it must never be presented as evidence of passing tests.

---

## 16. Cross-references

- Requirements: [`../PRD.md`](../PRD.md) §6, §7
- Test levels and scenarios: [`./TESTING_STRATEGY.md`](./TESTING_STRATEGY.md)
- Phase mapping: [`./IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md)
- Numbering obligations: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Audit obligations: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
