# Testing Strategy — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Target strategy |
| Version | 0.2 — engine, session, settings, and PWA scenarios added 2026-10-25 |
| Scope owner | Test levels, coverage obligations, critical scenarios, verification philosophy |
| Requirement source | [`../PRD.md`](../PRD.md) §7 |
| Acceptance register | [`./ACCEPTANCE_CRITERIA.md`](./ACCEPTANCE_CRITERIA.md) |

---

## 1. Philosophy

> **A green UI interaction proves the interface rendered. It does not prove the database is consistent, the permission was enforced, or the number was unique.**

Assert at the level where the property lives: number uniqueness at the constraint under concurrency; authorisation by direct request, not a hidden button; audit completeness by reading the audit table; immutability by attempting mutation through every path; session authority by attempting backdating; cache safety by inspecting the service-worker cache after navigation and logout.

**Failure behaviour is part of the requirement.** For every critical workflow assert (1) the happy path, (2) the defined failure result, (3) the specific `BR-` rule enforced. A workflow tested only for success is half-tested.

**No stack is installed, so no commands exist.** This document specifies *what* must be verified and at which level. **Nothing here has been executed.**

---

## 2. Test levels

| Level | Scope | Runs | Purpose |
|---|---|---|---|
| **Unit** | State machines, coverage arithmetic, ageing buckets, number rendering | Every commit | Logic without I/O |
| **Integration** | Module + **real MySQL/MariaDB** + real storage disk | Every commit or PR | Transactions, constraints, concurrency |
| **E2E** | Full HTTP request with a real database | Every PR | Workflow wiring, authorisation, Turbo responses |
| **Browser** | Turbo navigation, service worker, PWA install | Every PR and pre-release | Client-side lifecycle |
| **Manual / UAT** | Real users, real devices | Phase exits | Usability, adoption |
| **Non-functional** | Concurrency, restore, performance, device matrix | Phase exits | What unit tests cannot express |

**Must be integration, never unit:** number allocation under concurrency (real row locks and `LAST_INSERT_ID()` semantics) · unique and `CHECK` constraint enforcement · reservation limits across projects · audit append-only enforcement (real grants and triggers) · transactional coupling (real rollback) · deadlock retry · generated-column uniqueness.

**Engine pinning matters.** Constraint tests must run against the **exact** version pinned for production. Against SQLite or an older MySQL they would pass while production silently ignores `CHECK`.

---

## 3. Coverage obligations

| Area | Must be tested at | § |
|---|---|---|
| Document numbering | **Concurrency, constraints, rollback, step semantics** (I) + issuance flow (E2E) | 4 |
| Audit logging | **Immutability, coupling, hash chain** (I) + override flow (E2E) | 5 |
| Authorisation | **Direct unauthorised requests** (E2E) + row scoping (I) | 6 |
| Projects | Transition table (U), team-leader constraint (I), full lifecycle (E2E) | 7 |
| Materials | **Reservation races, ledger arithmetic** (I) + issue/return (E2E) | 8 |
| Procurement | **Unapproved PO refusal, receipt accumulation** (I) + purchase flow (E2E) | 9 |
| Technician ops | **Timestamp immutability** (I/E2E) + timed phone test (M) | 10 |
| Finance | **Allocation limits, snapshot immutability** (I) + invoice→payment (E2E) | 11 |
| Settings | **Secret never exposed, snapshot non-retroactivity** (I) + SMTP test (E2E) | 12 |
| WhatsApp / maps / PWA | Encoding (U), **cache inspection and install** (B) | 12 |
| Migration | **Collision detection, high-water mark** (I) + dry-run (E2E) | 13 |
| Backup/recovery | Restore rehearsal covering **database and files** (M) | 13 |

---

## 4. Document numbering

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| N1 | 50 concurrent issuances, one series | I | 50 distinct numbers; 0 violations | AC-DATA-002 |
| N2 | 20 concurrent issuances across 4 series | I | Each series unique; no cross-contamination | AC-DATA-002 |
| N3 | Rollback after allocation | I | Number not consumed; next issuance reuses it | AC-DOC-004 |
| N4 | Direct duplicate insert at DB level | I | Rejected by constraint | AC-DATA-002 |
| N5 | Client submits a chosen number | I | Server ignores it | AC-DOC-004 |
| N6–N7 | Month rollover; backdated invoice | I | New counter at `starting_value`; period follows the document date | AC-DOC-004 |
| N8 | Void then issue another | I | Voided number retained; next is the next step | AC-FIN-011 |
| N9 | PDF generation fails during issuance | I | No numbered document; number free | AC-FIN-002 |
| N10 | Double-submit of issuance | E2E | One invoice, one number | AC-FIN-002 |
| N11–N12 | Import duplicate; counter above high-water mark | I | Commit blocked; no post-migration collision | AC-DATA-003, AC-DATA-004 |
| N13 | Format configuration changed | I | Audited; existing numbers unchanged | AC-DOC-006 |
| N14 | **Step = 5** | I | 00001, 00006, 00011 | AC-SYS-004 |
| N15 | **Void at 00006 with step 5** | I | Next is 00011; 00006 never reissued | AC-SYS-004, AC-FIN-011 |
| N16–N17 | Starting number collides with history; counter lowered | I | Both refused | AC-SYS-004 |
| N18 | Heavy contention causing a deadlock | I | Bounded retry succeeds or surfaces a conflict; no corruption | AC-DATA-007 |

**N1, N3, and N15 would have caught the original defect** — the race, the transactional property, and the step/gap distinction.

---

## 5. Audit logging

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| A1–A3 | Every transition; issuance/void/credit; payment record and confirm | I | A record per transition with actor, role snapshot, before/after | AC-AUD-001 |
| A4–A5 | `UPDATE` / `DELETE` as the application user | I | **Both fail** | AC-AUD-002 |
| A6 | `UPDATE`/`DELETE` as a privileged user | I | **Trigger raises an error** | AC-AUD-002 |
| A7 | State change forced to fail after the audit write | I | Both roll back; no orphan | AC-AUD-003 |
| A8 | Audit write forced to fail | I | State change rolls back | AC-AUD-003 |
| A9–A11 | Chain verification; altered record; deleted record | I | Intact / divergence / gap reported | AC-AUD-003 |
| A12–A13 | Override without / with a reason | E2E | Refused / recorded and visible to `PEMILIK` | AC-AUD-004 |
| A14–A15 | Technician requests the log; owner views it | E2E | 403 / the view is itself audited | AC-AUD-006 |
| A16 | Settings change with a secret value | I | Records the key, **never the value** | AC-SYS-001 |

**A8 is the test that matters most:** the system refuses to act when it cannot record.

---

## 6. Authorisation

Every test issues a **direct request** with a wrong-role session. UI hiding is not tested because it is not a control.

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| S1 | Unauthenticated request to any business endpoint | E2E | 401 | AC-SEC-001 |
| S2–S3 | Technician requests invoices; requests payment confirmation | E2E | 403 | AC-SEC-006 |
| S4–S5 | Technician requests another's work order; the company-wide list | E2E | 403/404 with no existence leak; scoped result only | AC-TECH-009 |
| S6–S7 | **`ADMIN` / `SUPER_ADMIN` attempts to approve a purchase** | E2E | **403 for both** | AC-PROC-011 |
| S8 | **`PEMILIK` attempts SMTP or numbering configuration** | E2E | **403** | AC-SEC-002 |
| S9 | Multi-role account (`SUPER_ADMIN` + `ADMIN`) | E2E | Union applies; conflict rules still enforced | AC-SEC-003 |
| S10–S11 | Session idle beyond the limit; password changed | E2E | 401; other sessions revoked | AC-SEC-004 |
| S12–S14 | Hash inspection; secret scan; log inspection after an SMTP error | I/M | Slow hash; no secrets; no credential in logs | AC-SEC-005, AC-SEC-007, AC-SEC-009 |
| S15 | Production TLS check | M | HTTP redirected or rejected | AC-SEC-008 |
| S16 | Denied sensitive action | I | Denial itself audited | AC-SEC-002 |
| S17–S18 | Self-verification; override attempt to waive approval | E2E | Refused without override; **refused even with one** | AC-TECH-008, AC-PROC-011 |

**S4:** the response must not reveal whether the record exists. "403 forbidden, this work order belongs to Budi" is a data leak dressed as an error message.

---

## 7. Project lifecycle

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| P1 | Minimal project (no quotation, PO, materials, one technician) | E2E | Reaches `CLOSED` | AC-PROJ-001 |
| P2–P4 | Non-CCTV category; no quotation; no customer PO | E2E | No CCTV field; no blocking, no forced reason | AC-PROJ-002…004 |
| P5 | Revise target completion twice | I | Two revision rows; originals preserved | AC-PROJ-005 |
| P6 | Five-day project with daily reports | E2E | One work order, five reports, progress accumulates | AC-PROJ-006 |
| P7–P8 | Three technicians, one leader; second leader | I | One leader; second rejected by constraint | AC-PROJ-007 |
| P9–P11 | `DRAFT`→`IN_PROGRESS`; `CLOSED`→anything; manual `PENDING_…` | E2E | All refused with permitted statuses named | AC-PROJ-011 |
| P12–P13 | Close with an open work order; with material issued | E2E | Refused with the condition listed | AC-PROJ-010 |
| P14 | **Close with an `ACTIVE` session** | E2E | **Refused** | AC-PROJ-010 |
| P15 | Close with outstanding AR acknowledged | E2E | Permitted; AR appears in ageing | AC-PROJ-010 |
| P16–P17 | Cancel with issued material; cancel with an active session | E2E | Records retained; cancellation refused until the session closes | AC-PROJ-012 |
| P18 | Project without coordinates | E2E | Address shown; map placeholder; no error | AC-MAP-001 |

---

## 8. Materials

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| M1 | Create a requirement line | I | `stock_status = UNKNOWN` | AC-MATL-002 |
| M2 | Reserve on an `UNKNOWN` line | I | Refused, blocking condition named | AC-MATL-005 |
| M3 | Verify with an observed quantity | I | `VERIFIED`; verification row appended | AC-MATL-003 |
| M4–M5 | Two concurrent reservations for the last 10; three projects exceeding stock | I | Exactly one succeeds; third refused | AC-MATL-006 |
| M6–M7 | Release a reservation; cancel a project holding one | I | Availability restored; released | AC-MATL-004 |
| M8–M10 | Issue / consume / return | I | Correct movement; availability updated | AC-MATL-008…010 |
| M11 | Consume more than issued | I | Refused by `CHECK` | AC-MATL-009 |
| M12 | Write off damaged material | I | Movement with reason; still visible | AC-MATL-011 |
| M13–M14 | Direct quantity edit via endpoint; via SQL | I | No such endpoint; no writable column | AC-MATL-012 |
| M15 | Inject a ledger/cache variance | I | Reported; no auto-correction | AC-MATL-013 |
| M16 | Full issue lifecycle arithmetic | I | `consumed + returned + written_off ≤ issued` | AC-MATL-010 |
| M17–M19 | Technician raises a request on own project; on another; approves own | E2E | Created `PENDING_APPROVAL`; 403; 403 | AC-MATL-014, AC-PROC-011 |

---

## 9. Procurement and approval

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| R1–R3 | Create from shortfall; `PEMILIK` approves; rejects | I | Lines linked; `APPROVED` with role snapshot; `REJECTED` terminal | AC-PROC-001, AC-PROC-002 |
| R4 | **No approval recorded** | I | PO issuance refused; no number consumed | AC-PROC-011 |
| R5 | **Direct HTTP POST to issue a PO on an unapproved request** | E2E | **Rejected with no side effects** | AC-PROC-012 |
| R6 | Crafted body setting status | E2E | Refused; client-supplied status ignored | AC-PROC-012 |
| R7–R8 | Draft PO pre-approval; issue after approval | E2E/I | Allowed with no number / number, snapshot, PDF | AC-PROC-012, AC-PROC-006 |
| R9 | **Available stock after PO issuance** | I | **Unchanged** | AC-PROC-005 |
| R10–R11 | Partial, then three receipts | I | Cumulative equals ordered; `RECEIVED` | AC-PROC-007 |
| R12–R13 | Over-receipt without / with approval | I | Refused / permitted with exception | AC-PROC-007 |
| R14–R15 | Damage discrepancy; price variance | I | Stock rises only by accepted quantity; variance requires resolution | AC-PROC-008 |
| R16–R18 | Partial payment; record then confirm | I/E2E | `PARTIALLY_PAID`; `RECORDED` ≠ `CONFIRMED` | AC-FIN-004, AC-FIN-009 |
| R19 | Approve the same request twice concurrently | I | One current decision; second refused or explicitly superseded | AC-PROC-002 |

**R9 verifies BR-MATL-005**, the rule whose violation causes on-site aborts. **R5 verifies that approval is enforced by the server, not a hidden button.**

---

## 10. Technician operations, sessions, verification

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| T1 | Submit a report at 360 px | M | Timed < 3 minutes | AC-TECH-004 |
| T2–T3 | Report with photo; photo upload fails | E2E | Stored privately / report still submits | AC-TECH-005 |
| T4 | Connection drop mid-form | B | Draft preserved and restored | AC-PWA-004 |
| T5–T6 | Double-submit; oversized file | E2E | One report; rejected server-side | AC-TECH-004, AC-TECH-005 |
| T7–T8 | Record / resolve a blocker | E2E | `BLOCKED`; appears in the office list | AC-TECH-006 |
| T9–T10 | Reschedule / reassign | I | Reason required; **custody not auto-transferred** | AC-TECH-007 |
| T11 | **Two technicians, three days** | I | **Six sessions; no project-level single start/end exists** | AC-TECH-011 |
| T12 | Start a second session while one is active | I | Refused | AC-TECH-011 |
| T13 | **Submit a chosen `started_at`** | E2E | **Ignored; server timestamp used** | AC-TECH-012 |
| T14 | **Edit a session timestamp via any route** | E2E | **No such route** | AC-TECH-012 |
| T15–T16 | **Admin corrects a session**; correction without a reason | I/E2E | Correction row, original retained, reason, audited / refused | AC-TECH-012 |
| T17 | Abandon a forgotten session | E2E | `ABANDONED`; reason; timestamps attributed | AC-TECH-012 |
| T18–T19 | **End a session**; **submit completion** | E2E | **Work order and project unchanged**; `COMPLETED_PENDING_VERIFICATION` | AC-TECH-013 |
| T20–T22 | **Verifier accepts**; returns without a reason; returns with one | E2E | `VERIFIED` audited / **refused** / `IN_PROGRESS` with the reason visible | AC-TECH-013 |
| T23 | Partial completion, one of three | E2E | Verified; project remains `IN_PROGRESS` | AC-TECH-008 |
| T24–T27 | Supersede a report; silence; forgotten session; progress display | E2E/M | New version, original retrievable; exception lists; estimate with timestamp | AC-TECH-004, AC-TECH-006, AC-TECH-010 |
| T28 | Session duration across a date boundary | U | Correct duration; UTC stored, Jakarta rendered | AC-OPS-013 |

---

## 11. Finance

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| F1 | Issue an invoice | I | Number, snapshot, PDF, audit | AC-FIN-002 |
| F2–F3 | Rename customer / change item price after issuance | I | **Byte-identical**; totals unchanged | AC-FIN-003 |
| F4 | **Change the company bank account after issuance** | I | **Byte-identical; old account still printed** | AC-FIN-003, AC-SYS-003 |
| F5 | Edit an issued invoice via any path | I | No path exists | AC-FIN-003 |
| F6–F10 | Partial, full, over-allocation, overpayment, concurrent payments | I | Correct outstanding; unapplied credit; serialised | AC-FIN-004 |
| F11 | Overdue at the due-date boundary | U | Bucket correct before, on, after | AC-FIN-005 |
| F12–F14 | Void; issue after void; void with payments | I/E2E | Number retained; never reused; payments resolved first | AC-FIN-011 |
| F15 | Credit note and replacement | I | Original `CREDITED`; both numbered; chain traceable | AC-FIN-012 |
| F16–F17 | Receipt + invoice + payment; composition report | I | Counted **once** | AC-FIN-007 |
| F18 | Payment recorded, not confirmed | E2E | Distinct states; reported separately | AC-FIN-009 |
| F19–F21 | Ageing boundaries; re-run a past range; quotation issue | I | Buckets correct; reproducible; numbered with no revenue effect | AC-RPT-004, AC-RPT-008, AC-FIN-001 |

**F2 and F4 together prove historical immutability** — including the case most businesses get wrong: the company's own details changing after the customer has the document.

---

## 12. Settings, WhatsApp, PWA, Turbo, maps

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| G1–G4 | Save SMTP settings; reopen; inspect JSON; force an error | E2E/I/M | Encrypted; masked and write-only; absent from HTML, JSON, logs | AC-SYS-001, AC-SEC-009 |
| G5–G7 | Bad host; test email; change while a worker runs | E2E/I | Safe Indonesian error; delivered and recorded; picked up without restart | AC-SYS-002 |
| G8–G10 | Profile edit after issuance; non-image logo; oversized logo | I/E2E | Issued PDF unchanged; rejected by MIME; rejected by limits | AC-SYS-003 |
| G11–G12 | Settings as `ADMIN`/`PEMILIK`/`TEKNISI`; numbering change | E2E/I | 403; audited, non-retroactive | AC-SEC-002, AC-SYS-004 |
| W1–W3 | Compose a report; special characters; over 2000 characters | U | Correct fields; `encodeURIComponent`; visible truncation | AC-WA-001, AC-WA-002 |
| W4–W5 | Inspect for financial data and internal URLs | I | Neither present | AC-WA-002 |
| W6–W8 | Prepare a report; WhatsApp unavailable; unassigned project | E2E/B | "Prepared" not "sent"; copy fallback; 403 | AC-WA-003 |
| Q1 | Install on Android and iOS | M | Valid manifest; standalone launch | AC-PWA-001 |
| Q2–Q3 | **Cache after authenticated navigation; cache after logout** | B | **No HTML, JSON, or private file cached** | AC-PWA-002 |
| Q4 | Release a new asset version | B | Old caches removed; update prompt shown | AC-PWA-003 |
| Q5–Q8 | Navigate away and back; five navigations then submit; modal state; upload re-init | B | Components work; **exactly one handler fires**; modal closed; preview re-initialises once | AC-OPS-009…012 |
| Q9–Q10 | Invalid form; restore from cache then submit | E2E/B | **422 with form, error bag, old input**; CSRF valid, no 419 | AC-OPS-012 |
| Q11–Q14 | With / without coordinates; directions link; tile requests | B/M | Map with attribution / placeholder; correct URL, no key; policy compliant | AC-MAP-001…003 |

**Q2 and Q3 keep a PWA from becoming a data leak.** A service worker caching authenticated pages makes business data readable to the next person who picks up the phone.

---

## 13. Migration, backup, non-functional

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| D1–D3 | Malformed CSV; dry-run only; commit | I | Every bad row reported; **nothing written on dry-run** | AC-DATA-001, AC-DATA-002 |
| D4–D5 | Duplicate historical numbers; number matching a new one | I | Commit blocked; both identified | AC-DATA-003, AC-DATA-004 |
| D6–D8 | Delete / deactivate a referenced customer; full export | I | Refused / permitted with history intact / all tables and files | AC-DATA-005, AC-DATA-006 |
| B1–B2 | Restore database; **restore the files directory** | M | Data matches; **attachments and PDFs retrievable, hashes match** | AC-DATA-004 |
| B3–B6 | Duration; recency; chain intact; invoice after restore | M | ≤8 h RTO; ≤24 h RPO; chain intact; numbering continues | AC-DATA-005, AC-AUD-003, AC-DOC-004 |
| X1 | 10 concurrent state-changing users | Load | No errors, no violations, p95 within target | AC-DATA-007 |
| X2–X4 | 3G technician flow; office LAN; device matrix | M | p95 < 2 s / < 1 s; all flows usable | AC-OPS-004, AC-OPS-008 |
| X5–X7 | Error path; environment separation; diagnostic logs | E2E/M | Indonesian, no stack trace; no prod data below prod; diagnosable | AC-OPS-006, AC-OPS-001, AC-OPS-002 |
| X8 | **Timezone rendering** | I | UTC stored, `Asia/Jakarta` rendered | AC-OPS-013 |
| X9–X10 | **Engine version assertion; `CHECK` enforced** | I | Boot fails below the floor; invalid row rejected | AC-DATA-009 |
| X11 | **Expand → migrate → contract** | I | Each step deployable; rollback works | AC-DATA-010 |

**B2 and B6 are frequently forgotten.** A restore that recovers rows but loses files, or causes a numbering collision, has failed even if the data "looks fine".

---

## 14. Regression discipline

Every fixed defect gains a test that fails before the fix · numbering, approval, and audit tests run on every build · the permission matrix is tested per release, because new endpoints are the usual source of leaks · constraint tests run after every migration **against the pinned engine**, since a migration can drop a constraint and an engine downgrade can ignore one · Turbo navigation tests run on every PR, because client-side bugs are invisible to server tests · service-worker cache inspection runs per release, because a caching regression is a data-exposure regression · restore rehearsal runs quarterly, covering files.

---

## 15. Explicitly not tested

Bank transfer execution (no integration exists) · statutory tax calculations (Q-FIN-01) · native app behaviour (Q-UX-01) · multi-currency · offline submission and sync (only local draft preservation) · multi-location stock (Q-INV-01) · WhatsApp delivery or read receipts (unobservable by design) · in-app routing · SMTP deliverability guarantees (only connection and handoff are verifiable).

---

## 16. Cross-references

- Acceptance register: [`./ACCEPTANCE_CRITERIA.md`](./ACCEPTANCE_CRITERIA.md)
- Requirements: [`../PRD.md`](../PRD.md)
- Numbering tests: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §10
- Audit tests: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md) §5
- Turbo lifecycle: [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §3.2
- Phase mapping: [`./IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md)
