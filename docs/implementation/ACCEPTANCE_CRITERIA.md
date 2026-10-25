# Acceptance Criteria — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Target register — nothing has been executed |
| Version | 0.2 — expanded for the reconciled baseline 2026-10-25 |
| Scope owner | The **only** register of `AC-` identifiers |
| Requirement source | [`../PRD.md`](../PRD.md) |
| Test approach | [`./TESTING_STRATEGY.md`](./TESTING_STRATEGY.md) |

---

## 1. How to read this register

Each entry states what is verified, at which level, which requirement it covers, the expected outcome, and the **failure behaviour**. A criterion without a failure behaviour only tests the happy path.

**Levels:** `U` unit · `I` integration · `E2E` end-to-end · `B` browser · `M` manual/UAT.

**Status:** all entries are `Defined`. None is `Passing`. **No criterion has been executed.**

**Verification standard:** uniqueness, constraints, and concurrency assert at `I` against a real database **at the pinned engine version**; authorisation asserts at `E2E` with a direct request, not a hidden control; immutability asserts by attempting mutation through every path; audit completeness asserts by reading the audit table; client-side lifecycle asserts at `B`; usability asserts at `M` with real users. **A passing UI interaction satisfies no criterion here.**

---

## 2. Customer (`AC-CUST`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-CUST-001 | Customer master with required fields; referenced records deactivatable, not deletable | I | FR-CUST-001 | Record created; deactivation preserves references | Hard delete of a referenced customer refused by FK |
| AC-CUST-002 | Multiple contacts, exactly one primary | I | FR-CUST-002 | Contacts created; primary flag set | Second primary rejected by the conditional unique index |
| AC-CUST-003 | Service categories are configurable data | I | FR-CUST-003 | New category without code change; customer linked to several | Removing a category in use refused |
| AC-CUST-004 | Optional customer PO reference | E2E | FR-CUST-004 | Saves with and without a PO reference | Neither path blocks later steps |
| AC-CUST-005 | Customer 360 view | E2E | FR-CUST-005 | Projects, invoices, balance, document history for that customer only | No cross-customer leakage |
| AC-CUST-006 | Duplicate warning on create | E2E | FR-CUST-006 | Similar name or identical phone triggers a warning requiring confirmation | Proceeds on explicit confirmation; never silently blocked |

---

## 3. Project (`AC-PROJ`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-PROJ-001 | Minimal project completes the lifecycle | E2E | FR-PROJ-001, BR-PROJ-001 | No quotation, no customer PO, no material lines, one technician → `CLOSED` | No screen forces an optional step |
| AC-PROJ-002 | Service category drives no CCTV assumptions | E2E | FR-PROJ-002 | Non-CCTV project completes with no camera-specific field | Completion not gated on CCTV data |
| AC-PROJ-003 | Quotation is optional | E2E | FR-PROJ-003 | Project without a quotation proceeds through all states | No warning wall, no forced reason |
| AC-PROJ-004 | Customer PO is optional | E2E | FR-CUST-004, FR-PROJ-004 | Project without a customer PO proceeds | Same |
| AC-PROJ-005 | Schedule revision recorded, not overwritten | I | FR-PROJ-005 | Two revisions → two rows; originals intact | Revision without a reason refused |
| AC-PROJ-006 | Multi-day progress is not binary | E2E | FR-PROJ-006 | 5 days → 1 work order + 5 reports; progress accumulates | No single flag marks completion mid-way |
| AC-PROJ-007 | One-or-many technicians; at most one leader | I | FR-PROJ-007 | 3 assignments; 1 leader | Second active leader rejected by the conditional unique index |
| AC-PROJ-008 | Optional budget with actual-vs-budget display | E2E | FR-PROJ-008 | Budget set → comparison shown; absent → no error | Missing budget blocks no transition |
| AC-PROJ-009 | Material requirements derived and exposed | E2E | FR-PROJ-009 | Requirement lines visible from the project; readiness shown | — |
| AC-PROJ-010 | Closure refused with enumerated unmet conditions | E2E | FR-PROJ-010, BR-PROJ-004 | Closure with an open work order lists that condition; closure with an `ACTIVE` session is refused | Closure does not partially apply |
| AC-PROJ-011 | Illegal transitions refused, options named | E2E | FR-PROJ-011 | `DRAFT`→`IN_PROGRESS` refused with current status and allowed next statuses | No status change occurs |
| AC-PROJ-012 | Cancel/suspend retains records; an open session blocks cancellation | E2E | FR-PROJ-012, BR-PROJ-005 | Cancelled project keeps invoices, POs, payables, material records | Never deletes financial records; active session must close first |

---

## 4. Procurement (`AC-PROC`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-PROC-001 | Purchase request from a shortfall | I | FR-PROC-001 | Lines link to requirement lines; may span projects | Request without a line refused |
| AC-PROC-002 | Approval is a decision, not a purchase | I | FR-PROC-002, BR-PROC-001 | `APPROVED` records actor, role snapshot, time, reason; stock unchanged; no automatic PO | A rejected request cannot become a PO; a concurrent second decision does not silently overwrite the first |
| AC-PROC-004 | Supplier master with soft deactivation | I | FR-PROC-004 | Supplier deactivated; historical POs intact | Deleting a referenced supplier refused |
| AC-PROC-005 | **PO issuance does not increase stock** | I | FR-PROC-005, BR-MATL-005 | Available quantity identical before and after issuance | — |
| AC-PROC-006 | PO issuance allocates a number, freezes a snapshot, stores a PDF | I | FR-PROC-006 | Number unique; snapshot immutable; hash recorded | Any failure rolls back the whole issuance |
| AC-PROC-007 | Partial receipt accumulates correctly | I | FR-PROC-007 | Three receipts sum to the ordered quantity; `ISSUED`→`PARTIALLY_RECEIVED`→`RECEIVED` | Over-receipt refused without approval; cumulative never exceeds ordered |
| AC-PROC-008 | Discrepancies recorded and block completion | I | FR-PROC-008 | Short/damaged/wrong/price-variance recorded; line not `RECEIVED` until resolved | Unresolved discrepancy prevents PO closure |
| AC-PROC-009 | Supplier invoice recorded as payable | I | FR-PROC-009 | Payable with due date and amount; may precede or follow delivery | Payable for a cancelled PO refused |
| AC-PROC-010 | Payment recorded with partial support; no money moves | I | FR-PROC-010, BR-PROC-004 | Partial payment leaves correct outstanding; no banking call exists | Never reports having executed a transfer |
| AC-PROC-011 | **Every purchase requires `PEMILIK` approval** | E2E | FR-PROC-002, FR-PROC-011, BR-PROC-005 | Only `PEMILIK` can approve; `ADMIN`, `SUPER_ADMIN`, `TEKNISI` receive 403; no threshold, auto-approval, or delegation | An override cannot waive it; an unapproved request cannot proceed |
| AC-PROC-012 | **PO issuance on an unapproved request refused with no side effects** | E2E | FR-PROC-012 | A direct HTTP request against a `PENDING_APPROVAL` or `REJECTED` request is rejected | **No number consumed, no snapshot, no PDF, no supplier commitment**; a draft PO stays a draft with no number |

---

## 5. Materials (`AC-MATL`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-MATL-001 | Requirement lines independent of stock | I | FR-MATL-001 | Lines created with quantity and unit regardless of availability | — |
| AC-MATL-002 | New line defaults to `UNKNOWN` | I | FR-MATL-002, BR-MATL-001 | `stock_status = UNKNOWN` on creation | An `UNKNOWN` line cannot count toward readiness |
| AC-MATL-003 | Verification records the observed quantity and history | I | FR-MATL-003 | `VERIFIED`; verification row appended; re-verification adds a row | Verification without an observed quantity refused |
| AC-MATL-004 | Reservation create/release/consume | I | FR-MATL-004 | Each transition appends a state record; release restores availability | Direct deletion of a reservation impossible |
| AC-MATL-005 | **Reservation on an `UNKNOWN` line refused** | I | FR-MATL-005 | Refusal names the blocking condition and required action | No reservation row created |
| AC-MATL-006 | **Concurrent reservations cannot exceed verified stock** | I | FR-MATL-006, BR-MATL-002 | 2 parallel requests for the last 10 units → exactly 1 succeeds | Second refused with the available quantity shown |
| AC-MATL-007 | Shortfall computed and exposed to procurement | I | FR-MATL-007 | `shortfall = required − covered` matches a hand calculation | Competing reservations reduce `covered` |
| AC-MATL-008 | Issue to a named technician | I | FR-MATL-008 | `ISSUE_OUT` movement; availability reduced; custody recorded | An issue exceeding the reservation re-checks availability |
| AC-MATL-009 | Consumption recorded | I | FR-MATL-009 | `CONSUME` movement; issue line updated | Consuming more than issued refused by `CHECK` |
| AC-MATL-010 | Return restores availability | I | FR-MATL-010, BR-MATL-004 | `RETURN_IN` movement; availability restored | `consumed + returned + written_off ≤ issued` always holds |
| AC-MATL-011 | Write-off visible with reason | I | FR-MATL-011 | `WRITE_OFF` movement with a reason code; quantity still reported | Never removes the quantity from history |
| AC-MATL-012 | No direct quantity edit path | I | FR-MATL-012, BR-MATL-003 | No endpoint or UI edits a quantity; corrections are compensating movements | A direct SQL update finds no writable balance column |
| AC-MATL-013 | Reconciliation detects variance | I | FR-MATL-013 | Injected ledger/cache divergence reported with the variance | Never auto-corrects |
| AC-MATL-014 | Technician raises an additional request on own project only | E2E | FR-MATL-014 | Created `PENDING_APPROVAL`; technician sees its status | Request on another project returns 403 without revealing existence; technician cannot approve |

---

## 6. Technician operations (`AC-TECH`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-TECH-001 | Technician linked to exactly one user | I | FR-TECH-001 | One-to-one constraint holds | Second technician record for the same user refused |
| AC-TECH-002 | Team-leader designation scoped to project, two capabilities only | E2E | FR-TECH-002 | Leader sees that project's team reports and may submit a team summary | No financial data; no verification authority; no other project visible |
| AC-TECH-003 | One work order per technician per project | I | FR-TECH-003 | Three technicians → three work orders | Duplicate assignment for the same technician/project prevented |
| AC-TECH-004 | Daily report in < 3 minutes at 360 px | M | FR-TECH-004, M3 | Median of 5 attempts under target with 3 technicians | Data preserved on connection drop; double-submit idempotent |
| AC-TECH-005 | Photo upload with server-side limits | I | FR-TECH-005 | Oversized, wrong-type, or over-dimension file rejected server-side; valid file stored privately | Photo failure does not block submission |
| AC-TECH-006 | Blocker recorded and surfaced | E2E | FR-TECH-006 | `BLOCKED`; appears in the office list; silence detected after one day | Blocker requires a reason code |
| AC-TECH-007 | Reschedule/reassign recorded with reason | I | FR-TECH-007 | Previous values preserved; actor and timestamp recorded | Reassignment does **not** auto-transfer custody; an active session must close first |
| AC-TECH-008 | Partial completion and return for correction | E2E | FR-TECH-008, BR-TECH-002 | One of three verified → project stays `IN_PROGRESS`; return sets `IN_PROGRESS` with a reason | Self-verification refused without override |
| AC-TECH-009 | Technician sees only own assignments | E2E | FR-TECH-009, NFR-SEC-006 | Direct request for another's work order returns 403/404 without revealing existence | No finance, supplier, settings, or peer data in any response |
| AC-TECH-010 | Progress presented as estimate | E2E | FR-TECH-010 | Estimate with a last-updated timestamp | Never labelled authoritative completion |
| AC-TECH-011 | **Work sessions first-class; many per project** | I | FR-TECH-011 | Two technicians over three days produce six independent sessions with distinct start and completion timestamps | A project-level single start/end is not representable; a second `ACTIVE` session for the same technician is refused |
| AC-TECH-012 | **Session timestamps server-authoritative and immutable** | E2E | FR-TECH-012, BR-TECH-004 | Server writes both timestamps; a client-supplied `started_at` is ignored; no route edits them | A correction requires an authorised user, a mandatory reason, and a new attributed record while originals remain; abandonment requires a reason |
| AC-TECH-013 | **Completion verification is an explicit audited decision** | E2E | FR-TECH-013, BR-TECH-005 | Submit → `PENDING_COMPLETION_VERIFICATION`; verify → `VERIFIED`; return → `IN_PROGRESS` | Ending a session or submitting completion never completes the work order or project; a return without a reason is refused; both outcomes audited |

---

## 7. Finance (`AC-FIN`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-FIN-001 | Quotation issued with number and PDF | I | FR-FIN-001 | Number allocated at issuance; PDF stored; no revenue effect | Draft has no number |
| AC-FIN-002 | Invoice issued with an allocated number | I | FR-FIN-002 | Unique number; snapshot; PDF; audit — one transaction | Any failure leaves no invoice and no consumed number |
| AC-FIN-003 | **Issued invoice immutable after customer, item, or company edits** | I | FR-FIN-003, BR-SYS-001, M4 | Rename customer, change price, change bank account → re-download hash identical | No path edits an issued invoice or regenerates its PDF with new details |
| AC-FIN-004 | Partial payments and accurate outstanding | I | FR-FIN-004, BR-FIN-003 | Two partial payments → `PAID`, outstanding 0; allocation never exceeds balance | Overpayment becomes unapplied credit, never a negative balance |
| AC-FIN-005 | Overdue derived from due date | U | FR-FIN-005 | Bucket correct before, on, and after the due date | No stored `OVERDUE` flag can go stale |
| AC-FIN-006 | Expenses recorded with attribution | I | FR-FIN-006 | Project expense enters project cost; operational expense does not | Expense without a category refused |
| AC-FIN-007 | **Project cost not double counted** | I | FR-FIN-007, BR-FIN-005 | Receipt + invoice + payment counts **once**; composition report shows the source | Unique `(project, source_type, source_id)` prevents duplicates |
| AC-FIN-008 | Due dates and overdue items surfaced | E2E | FR-FIN-008 | Days-overdue computed and displayed | — |
| AC-FIN-009 | **Payment recorded ≠ confirmed** | E2E | FR-FIN-009, BR-PROC-002 | Distinct states, actors, and reports | Confirmation by the recorder requires override |
| AC-FIN-010 | Payables/receivables summaries | E2E | FR-FIN-010 | Outstanding per counterparty matches allocation arithmetic | — |
| AC-FIN-011 | Void retains number and record | I | FR-FIN-011, BR-FIN-002 | `VOID`; number never reused; PDF retrievable | Void without a reason refused; void with unresolved payments refused |
| AC-FIN-012 | Credit note and replacement preserve the original | I | FR-FIN-012 | Original `CREDITED`; both new documents numbered; chain traceable | Original bytes never modified |

---

## 8. Documents (`AC-DOC`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-DOC-001 | Private local storage; no public URL | I | FR-DOC-001 | Files outside the webroot; unauthenticated direct path fails; no public URL anywhere | No `public_path()` in business logic or Blade |
| AC-DOC-002 | Document metadata and history | I | FR-DOC-002 | Type, owner, number, status, timestamps, history present | — |
| AC-DOC-003 | PDF generation for all issued types | I | FR-DOC-003 | Quotation, PO, invoice, credit note, receipt render and store with a hash | Generation failure rolls back issuance |
| AC-DOC-004 | **Server-side number allocation only** | I | FR-DOC-004, BR-DOC-002 | Client-supplied number ignored; server allocates | No template or report can produce a number |
| AC-DOC-005 | Retrieval by number with full history | E2E | FR-DOC-005 | Document, snapshot, PDF, related documents, timeline shown | Ambiguity across types resolved by showing the type |
| AC-DOC-006 | Document type registry, configurable per type | I | FR-DOC-006 | New type as data; format, padding, starting value, step configurable | Existing numbers unaffected by a configuration change |
| AC-DOC-007 | Attachments inherit owner access rules | I | FR-DOC-007 | Reachable only by users who can read the owner | A technician cannot fetch an invoice attachment |
| AC-DOC-008 | Issued PDF immutable | I | FR-DOC-008 | Re-download hash matches after a template change | Regeneration never replaces original bytes |
| AC-DOC-009 | **Storage abstraction permits S3 migration without a rewrite** | I | FR-DOC-009 | One disk abstraction; switching disk configuration needs no business-logic change; a migration procedure exists | No hard-coded local path, no `public_path()`, no driver-specific Blade assumption |
| AC-DOC-010 | **Downloads authorised and streamed** | E2E | NFR-SEC-010 | Every download checks permission on the owner, streams, and sets `Content-Disposition` | Unauthorised request returns 403 before any file handle opens; directory listing impossible |

---

## 9. Reporting (`AC-RPT`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-RPT-001 | Daily progress per project/technician, phone-readable | E2E | FR-RPT-001 | Renders at 360 px from reports and sessions; exportable | Technician view shows only own data |
| AC-RPT-002 | Project dashboard with six indicators | E2E | FR-RPT-002 | Status, readiness, variance, cost, billed, outstanding all correct | Readiness shown separately from status |
| AC-RPT-003 | Material status report per line | I | FR-RPT-003 | Required/reserved/purchased/received/issued/consumed/returned/remaining reconcile | Arithmetic matches the ledger |
| AC-RPT-004 | AR/AP ageing by counterparty and bucket | I | FR-RPT-004 | Buckets correct at boundaries | Disputed payables excluded from due-now but still visible |
| AC-RPT-005 | Procurement pipeline by state | I | FR-RPT-005 | Every PO in exactly one bucket; `PENDING_APPROVAL` requests shown separately | Partially received shown as such, not as received |
| AC-RPT-006 | Technician workload and session hours | E2E | FR-RPT-006 | Assignments, days, open and blocked counts, summed session duration correct | Active sessions not counted as completed hours |
| AC-RPT-007 | Date range and export | E2E | FR-RPT-007 | CSV for all reports; PDF for external documents | Export audited |
| AC-RPT-008 | Report reproducibility | I | FR-RPT-008 | Re-running a past range yields identical figures unless corrections occurred | Corrections visible in the audit trail |

---

## 10. Data (`AC-DATA`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-DATA-001 | CSV import with per-row rejection report | I | FR-DATA-001 | Every malformed row reported with row, column, reason | Valid rows not partially committed during validation |
| AC-DATA-002 | **Dry-run then explicit commit; concurrent uniqueness** | I | FR-DATA-002, NFR-DATA-002 | Dry-run writes nothing; commit only after clean validation; N parallel issuances → N unique numbers, 0 violations | Concurrency never produces a duplicate |
| AC-DATA-003 | Historical numbers preserved verbatim | I | FR-DATA-003, BR-DATA-001 | Imported strings identical to source; flagged historical | Never renumbered to resolve a conflict |
| AC-DATA-004 | **Imported numbers cannot collide; files backed up** | I | FR-DATA-004, NFR-DATA-004 | Collision detected pre-commit; counter above the high-water mark; database and files both restorable | Import blocked when a collision is unresolvable |
| AC-DATA-005 | Soft deactivation; delete refused | I | FR-DATA-005, BR-DATA-002 | Deactivation succeeds; deletion of a referenced record refused by FK | — |
| AC-DATA-006 | Full export including files | I | FR-DATA-006 | All tables plus stored files; re-importable | — |
| AC-DATA-007 | Safe under ≥10 concurrent state-changing users | I | NFR-DATA-007 | No errors, no violations, no lost updates | Deadlocks retried under a bounded policy or surfaced, never silently swallowed |
| AC-DATA-008 | Retention configuration applied | I | NFR-DATA-008 | Policy configurable; archival does not delete | — |
| AC-DATA-009 | **Engine floor enforced and `CHECK` constraints live** | I | NFR-DATA-009, NFR-PLAT-004 | Boot fails on MySQL < 8.0.16 / MariaDB < 10.2.1; an invalid row is rejected | A silently ignored `CHECK` is a test failure, not a warning |
| AC-DATA-010 | **Non-transactional DDL handled by expand → migrate → contract** | I | NFR-DATA-010 | Each step independently deployable; rollback possible without data loss | No destructive DDL in the same release as code depending on it |

---

## 11. Audit (`AC-AUD`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-AUD-001 | Audit record for every transition, including sessions, verification, settings | I | FR-AUD-001, NFR-AUD-002 | Every transition produces a record with actor, role snapshot, action, before/after | A missing record fails the test |
| AC-AUD-002 | **Append-only enforced at the database level** | I | FR-AUD-002, NFR-AUD-001 | `UPDATE`/`DELETE` fail for the application user; triggers also refuse a privileged session | A privilege or trigger restored by a migration is caught by the test |
| AC-AUD-003 | **Transactional coupling; hash chain intact** | I | FR-AUD-003, NFR-AUD-003 | Forced audit failure rolls back the change; an altered record breaks chain verification | No committed change without its audit record |
| AC-AUD-004 | Override requires a reason and is visible to `PEMILIK` | E2E | FR-AUD-004, NFR-AUD-004 | Reason mandatory; appears in the owner report | Override never bypasses an invariant or the purchase-approval requirement |
| AC-AUD-005 | Retention and archival without loss | I | FR-AUD-006, NFR-AUD-005 | Archived records remain verifiable | A routine job never deletes |
| AC-AUD-006 | Restricted access; viewing is audited | E2E | FR-AUD-005, NFR-AUD-006 | `PEMILIK`, `SUPER_ADMIN`, or `audit.view` only; every view audited | `TEKNISI` and `ADMIN` receive 403 |

---

## 12. Security (`AC-SEC`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-SEC-001 | No anonymous business data access | E2E | NFR-SEC-001 | Every business endpoint returns 401 without a session | Login and health excepted |
| AC-SEC-002 | **Server-side authorisation per action and resource** | E2E | NFR-SEC-002 | Direct unauthorised requests return 403 regardless of UI; `PEMILIK` cannot reach settings | Denial itself audited for sensitive actions |
| AC-SEC-003 | Multi-role union with conflict controls | E2E | NFR-SEC-003 | Combined roles grant the union; conflicting pairs still require override | No implicit escalation from combining roles |
| AC-SEC-004 | Session expiry and revocation | E2E | NFR-SEC-004 | Idle expiry enforced; password change revokes other sessions | — |
| AC-SEC-005 | Password hashing and policy | I | NFR-SEC-005 | Slow salted hash; minimum length; no plaintext anywhere | Weak password rejected |
| AC-SEC-006 | Technician blocked from financial, settings, company-wide endpoints | E2E | NFR-SEC-006 | 403 on every such endpoint | Verified per release, not once |
| AC-SEC-007 | No secrets in repo or logs | M | NFR-SEC-007 | Secret scan clean; logs redact sensitive fields | — |
| AC-SEC-008 | TLS enforced in production | M | NFR-SEC-008 | Plain HTTP redirected or rejected | — |
| AC-SEC-009 | **Stored credentials encrypted, write-only, never exposed** | I | NFR-SEC-009 | SMTP password encrypted at rest; field masked and write-only; absent from HTML source, JSON, logs, error output | A credential appearing anywhere is a hard failure |

---

## 13. System settings (`AC-SYS`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-SYS-001 | SMTP configuration with encrypted credential and no disclosure | E2E | FR-SYS-001 | All fields savable; password encrypted; reopening shows a masked empty field; audit records the change without the value | No credential in page source, response, log, or audit row |
| AC-SYS-002 | Connection test, test email, safe Indonesian messages, safe config pickup | E2E | FR-SYS-002 | Both actions return actionable Indonesian outcomes; a settings change is picked up by the mail path and queue workers | An invalid host produces a clear message, never a stack trace; workers do not keep stale credentials |
| AC-SYS-003 | Company profile and bank details, snapshotted at issuance | I | FR-SYS-003, BR-SYS-001 | Editable by `SUPER_ADMIN` only; logo validated by MIME, dimensions, size; issued documents keep their snapshot | A later profile change never alters an issued PDF; a non-image logo rejected |
| AC-SYS-004 | **Numbering configuration complete, audited, non-retroactive** | I | FR-SYS-004, BR-FIN-006 | Prefix, type code, pattern, separators, padding, starting value, step, reset policy, and effective timing configurable; step 5 yields 00001/00006/00011 | A starting value or step that could collide with issued or imported numbers is refused; a voided number is never reissued; existing numbers unchanged |

---

## 14. WhatsApp, maps, PWA (`AC-WA`, `AC-MAP`, `AC-PWA`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-WA-001 | Report content correct and complete | U | FR-WA-001 | Status, work date, progress summary, blockers, human-readable document numbers | No financial amount, no private storage link |
| AC-WA-002 | **URL construction and content safety** | U | FR-WA-002 | `https://wa.me/?text=` with `encodeURIComponent`; special characters survive; ~2000-character limit handled with a visible truncation notice | No sensitive financial data, no authenticated internal URL, no credential in the text |
| AC-WA-003 | **Authorisation and manual-send semantics** | E2E | FR-WA-003 | Only an assigned technician can prepare; destination manual; "report prepared" recorded, never "message sent"; a "mark as sent" action is labelled unverified | When WhatsApp cannot open, the text is shown with a copy-to-clipboard fallback and instructions — never a silent failure |
| AC-MAP-001 | Map displays with attribution; projects without coordinates work | B | FR-PROJ-013, FR-MAP-001 | OpenStreetMap tiles with visible attribution; no coordinates → address plus a graceful placeholder | No error, no blank frame, no blocked workflow |
| AC-MAP-002 | Directions deep link without an API key | B | FR-MAP-002 | Google Maps URL launches with the correct destination | No API key required; no in-app routing attempted |
| AC-MAP-003 | Tile usage compliant and configurable | M | FR-MAP-003 | No bulk downloading, no heavy caching; tile source configurable | A policy breach is a release blocker |
| AC-PWA-001 | Installable with a valid manifest and icons | M | FR-PWA-001 | Installs to the home screen on Android and iOS; launches standalone | Missing icon or invalid manifest fails the test |
| AC-PWA-002 | **No authenticated business data cached** | B | FR-PWA-002, NFR-OPS-010 | After navigation and after logout, the cache holds only versioned static assets | Any cached HTML, JSON, or private file is a hard failure |
| AC-PWA-003 | Cache versioning and update flow | B | FR-PWA-003 | Old caches removed on activation; in-app update prompt; `skipWaiting`/`clients.claim` strategy documented | A stale cache never serves an old shell indefinitely |
| AC-PWA-004 | **Local draft preservation, distinct from submission** | B | FR-PWA-004 | An interrupted form restores its draft on reload; the draft is never a submitted record | Draft loss on logout or device change is documented (Q-PWA-01); the UI never implies submission |

---

## 15. Operations (`AC-OPS`)

| ID | Criterion | Level | Covers | Expected | Failure behaviour |
|---|---|---|---|---|---|
| AC-OPS-001 | Environment separation | M | NFR-OPS-001 | Dev/staging/prod isolated; no live personal data below production | — |
| AC-OPS-002 | Diagnosable logs | M | NFR-OPS-002 | A reported failure is diagnosable without reproduction | No secret in those logs |
| AC-OPS-003 | Mobile-first at 360 px | M | NFR-OPS-003 | All technician flows usable; targets ≥44 px; no horizontal scroll | — |
| AC-OPS-004 | Performance targets | M | NFR-OPS-004 | Technician p95 < 2 s on 3G-class; office p95 < 1 s on LAN | — |
| AC-OPS-005 | Availability 99% business hours | M | NFR-OPS-005 | Monitoring confirms; maintenance announced | — |
| AC-OPS-006 | Indonesian, actionable error messages | E2E | NFR-OPS-006 | States what failed and what to do; no stack trace | — |
| AC-OPS-007 | UI language Indonesian, `Asia/Jakarta` timezone | M | NFR-OPS-007, NFR-PLAT-005 | Navigation, labels, statuses, PDFs in Indonesian; dates rendered in Jakarta time | — |
| AC-OPS-008 | Browser matrix | M | NFR-OPS-008 | Chrome, Safari iOS, Firefox, Edge all functional | — |
| AC-OPS-009 | **Components survive Turbo navigation** | B | NFR-OPS-011, NFR-PLAT-002 | Navigate away and back; dropdowns, modals, sidebar still work | No reliance on `DOMContentLoaded` for re-initialisation |
| AC-OPS-010 | **No duplicate handlers after repeated navigation** | B | NFR-OPS-011 | After five navigations, a submit fires exactly one handler | Idempotent initialisation via `data-initialized`; no anonymous handlers surviving cache |
| AC-OPS-011 | **Transient UI state reset across navigation** | B | NFR-OPS-011 | Modal open → navigate → return: modal closed, inline errors cleared | `turbo:before-cache` teardown verified |
| AC-OPS-012 | **Upload widgets re-initialise; validation returns 422 with the form** | E2E, B | NFR-OPS-011 | Upload preview works after navigation; an invalid form returns 422 with error bag and old input; CSRF valid after cache restore | No redirect-with-flash-only for validation failures; no 419 after restore |
| AC-OPS-013 | **UTC storage, Jakarta rendering** | I | NFR-OPS-009 | A fixed UTC timestamp renders as the correct Jakarta time; stored value unchanged | No reliance on the database server's local timezone |
| AC-OPS-014 | **Laravel version floor and release gate** | I, M | NFR-PLAT-001, NFR-PLAT-008, NFR-PLAT-009 | Resolved version ≥ 13.29.0; `composer audit` run and output archived; advisory finding recorded before deployment | Deployment blocked below the floor or with an unrecorded audit; a downgrade to Laravel 12 or earlier is a hard failure |
| AC-OPS-015 | **TailAdmin compatibility and licence sign-off** | M | NFR-PLAT-003 | Written record of package, tag, licence including attribution, framework constraint, Tailwind/Vite versions, and Laravel 13 status | Phase 1 cannot exit without it; substituting Filament or a paid edition is a hard failure |
| AC-OPS-016 | **Module boundaries enforced** | I | NFR-PLAT-007 | An automated dependency check fails the build on an illegal cross-module import | A boundary violation cannot reach a release |

---

## 16. Cross-cutting gates

A phase cannot exit until these hold.

| Gate | Criteria | Why it blocks |
|---|---|---|
| G1 — Numbering proven | AC-DATA-002, AC-DOC-004, AC-FIN-002, AC-FIN-011, AC-SYS-004 | Issuing documents on unproven numbering repeats the original defect |
| G2 — Audit proven | AC-AUD-001, AC-AUD-002, AC-AUD-003 | Approvals without audit are unattributable forever |
| G3 — Authorisation proven | AC-SEC-002, AC-SEC-006, AC-TECH-009, AC-PROC-011 | A leak found after go-live cannot be un-leaked |
| G4 — Immutability proven | AC-FIN-003, AC-DOC-008, AC-TECH-012 | Mutated documents or timestamps destroy dispute defensibility |
| G5 — Material integrity proven | AC-MATL-005, AC-MATL-006, AC-PROC-005 | Double-promising causes on-site failure |
| G6 — Cost integrity proven | AC-FIN-007 | Double-counted cost drives wrong pricing decisions |
| G7 — Recovery proven | AC-DATA-004, AC-DATA-005 | An untested backup is not a backup; a database-only restore loses evidence |
| **G8 — Platform gates proven** | AC-OPS-014, AC-OPS-015, AC-DATA-009 | A framework downgrade, an unlicensed template, or a silently ignored constraint invalidates every other guarantee |
| **G9 — Approval proven** | AC-PROC-011, AC-PROC-012 | Unapproved spending is the financial control the client asked for |
| **G10 — Client-side lifecycle proven** | AC-OPS-009…012, AC-PWA-002 | A broken Turbo page or a leaking service worker makes the app unreliable or unsafe |

---

## 17. Status summary

| Area | Defined | Executed | Passing |
|---|---|---|---|
| Customer | 6 | 0 | 0 |
| Project | 12 | 0 | 0 |
| Procurement | 11 | 0 | 0 |
| Materials | 14 | 0 | 0 |
| Technician | 13 | 0 | 0 |
| Finance | 12 | 0 | 0 |
| Documents | 10 | 0 | 0 |
| Reporting | 8 | 0 | 0 |
| Data | 10 | 0 | 0 |
| Audit | 6 | 0 | 0 |
| Security | 9 | 0 | 0 |
| Settings | 4 | 0 | 0 |
| WhatsApp / maps / PWA | 10 | 0 | 0 |
| Operations | 16 | 0 | 0 |
| **Total** | **141** | **0** | **0** |

**No criterion has been executed.** There is no application code, test framework, or build system in this repository. This table records definitions only and must never be presented as evidence of passing tests.

---

## 18. Cross-references

- Requirements: [`../PRD.md`](../PRD.md) §6, §7
- Test levels and scenarios: [`./TESTING_STRATEGY.md`](./TESTING_STRATEGY.md)
- Phase mapping and gates: [`./IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md)
- Numbering obligations: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Audit obligations: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
