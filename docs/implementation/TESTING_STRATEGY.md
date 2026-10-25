# Testing Strategy — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed |
| Version | 0.1 |
| Scope owner | This document owns test levels, coverage obligations, critical scenarios, and verification philosophy |
| Requirement source | [`../PRD.md`](../PRD.md) §7 |
| Acceptance register | [`./ACCEPTANCE_CRITERIA.md`](./ACCEPTANCE_CRITERIA.md) |

---

## 1. Testing philosophy

### 1.1 A successful UI action is not proof

The central rule of this strategy:

> **A green UI interaction proves the interface rendered. It does not prove the database is consistent, the permission was enforced, or the number was unique.**

Every critical scenario therefore asserts at the level where the property actually lives:

| Property | Asserted at |
|---|---|
| Document number uniqueness | Database constraint + concurrent execution |
| Reservation correctness | Database state after parallel transactions |
| Authorisation | Direct request with an unauthorised session, not a hidden button |
| Audit completeness | Audit table contents, not a success message |
| Immutability | Attempted mutation via every available path |
| Cost accuracy | Aggregated project cost rows, not a dashboard total |

**Corollary:** tests that only click buttons and check for "success" text are treated as smoke tests, not verification.

### 1.2 Failure behaviour is part of the requirement

For every critical workflow, the test suite asserts three things:

1. **Expected outcome** — the happy path produces the right state.
2. **Failure behaviour** — invalid input, concurrency, and interruption produce a defined, safe result.
3. **Business rule** — the specific `BR-` rule that governs the case is enforced.

A workflow tested only for success is half-tested.

### 1.3 No stack exists, so no commands exist

**[Confirmed]** the repository is empty: there is no test runner, no framework, no CI configuration. This document specifies **what must be verified and at which level**, not which command to run. Concrete tooling is chosen at Phase 1 and recorded in the ADRs and `AGENTS.md` at that point.

**Nothing in this document has been executed.** No test has been run, because no code exists.

---

## 2. Test levels

| Level | Scope | Runs | Purpose |
|---|---|---|---|
| **Unit** | Pure domain logic: state machine transitions, coverage arithmetic, ageing buckets, allocation rendering | Every commit, fast | Logic correctness without I/O |
| **Integration** | Module + real database + real storage adapter | Every commit or PR | Transactional correctness, constraints, concurrency |
| **E2E** | Full request through the application boundary with a real database | Every PR | Workflow wiring, authorisation at the boundary |
| **Manual / UAT** | Real users on real devices with real data shapes | Phase exits, pre-release | Usability, adoption, business fit |
| **Non-functional** | Concurrency, restore, performance, device matrix | Phase exits, pre-release | The properties unit tests cannot express |

### 2.1 What must be integration, never unit

Some properties cannot be proven with mocks:

| Property | Why integration is mandatory |
|---|---|
| Document number allocation under concurrency | Requires real row locks |
| Unique constraint enforcement | Requires a real database |
| Reservation limits across projects | Requires real isolation behaviour |
| Audit append-only enforcement | Requires real privileges |
| Transactional coupling of audit and state | Requires real rollback |
| Foreign key and CHECK constraints | Requires a real schema |
| Restore and backup | Requires real infrastructure |

**Mocked database tests are explicitly insufficient for these.** A mock that returns what the test tells it to return proves nothing about concurrency.

---

## 3. Coverage obligations by requirement area

| Area | Unit | Integration | E2E | Manual | Required scenarios |
|---|---|---|---|---|---|
| Document numbering | Rendering, period derivation | **Concurrency, constraints, rollback** | Issuance flow | — | §4 |
| Audit logging | Event construction | **Immutability, transactional coupling, hash chain** | Override flow | Owner review of override report | §5 |
| Authorisation | Policy evaluation | Row scoping | **Direct unauthorised requests** | Role assignment walkthrough | §6 |
| Projects | Transition table | Constraint on team leader | Full lifecycle | Minimal-project usability | §7 |
| Materials | Coverage arithmetic | **Reservation races, ledger arithmetic** | Issue/return flow | Reconciliation walkthrough | §8 |
| Procurement | Approval policy evaluation | PO→receipt accumulation | Full purchase flow | Office staff walkthrough | §9 |
| Technician ops | Progress banding | Scoped queries | Report submission | **Timed phone test** | §10 |
| Finance | Ageing, balance arithmetic | **Allocation limits, snapshot immutability** | Invoice→payment flow | Owner review of AR report | §11 |
| Migration | Row validation | **Collision detection, high-water mark** | Import dry-run + commit | Data-quality review | §12 |
| Backup/recovery | — | Restore rehearsal | — | Owner sign-off | §13 |

Bold entries are the ones most likely to be skipped and most likely to matter.

---

## 4. Document numbering tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| N1 | 50 concurrent invoice issuances, one series | Integration | 50 distinct numbers; 0 constraint violations | AC-DATA-002 |
| N2 | 20 concurrent issuances across 4 series | Integration | Each series internally unique; no cross-contamination | AC-DATA-002 |
| N3 | Transaction rolled back after allocation | Integration | Number not consumed; next issuance gets the same value | AC-DOC-004 |
| N4 | Direct duplicate insert at DB level | Integration | Rejected by constraint | AC-DATA-002 |
| N5 | Client submits a chosen document number | Integration | Server ignores it; allocates its own | AC-DOC-004 |
| N6 | Year rollover | Integration | New period counter starts at 1 | AC-DOC-004 |
| N7 | Backdated invoice (December date, January issuance) | Integration | Number belongs to December series | AC-DOC-004 |
| N8 | Void then issue another | Integration | Voided number retained; next number is +1, never reused | AC-FIN-011 |
| N9 | PDF generation fails during issuance | Integration | No numbered document exists; number free | AC-FIN-002 |
| N10 | Double-submit of issuance request | E2E | Exactly one invoice and one number | AC-FIN-002 |
| N11 | Historical import containing a duplicate | Integration | Commit blocked; both rows identified | AC-DATA-003 |
| N12 | Counter set above historical high-water mark | Integration | No post-migration collision possible | AC-DATA-004 |
| N13 | Series format configuration changed | Integration | Audit event recorded; existing numbers unchanged | AC-DOC-006 |

**N1 and N3 are the tests that would have caught the original defect.** N1 because it exercises the actual race; N3 because it proves the allocator is transactional rather than merely sequential.

---

## 5. Audit logging tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| A1 | Every project status transition | Integration | One audit record per transition, with actor and before/after | AC-AUD-001 |
| A2 | Every invoice issuance, void, credit note | Integration | Audit record present, correct action name | AC-AUD-001 |
| A3 | Every payment record and confirmation | Integration | Two distinct audit records with different actors where applicable | AC-AUD-001 |
| A4 | `UPDATE audit_events` as application role | Integration | **Fails** | AC-AUD-002 |
| A5 | `DELETE FROM audit_events` as application role | Integration | **Fails** | AC-AUD-002 |
| A6 | State change forced to fail after audit write | Integration | Both roll back; no orphan audit record | AC-AUD-003 |
| A7 | Audit write forced to fail | Integration | State change rolls back; action does not occur | AC-AUD-003 |
| A8 | Hash chain verification after normal operation | Integration | Chain intact | AC-AUD-003 |
| A9 | Manually alter one audit record | Integration | Chain verification reports divergence at that record | AC-AUD-003 |
| A10 | Delete one audit record | Integration | Chain verification reports the gap | AC-AUD-003 |
| A11 | Override without reason | E2E | Refused | AC-AUD-004 |
| A12 | Override with reason | E2E | Recorded; appears in owner override report | AC-AUD-004 |
| A13 | Technician requests audit log | E2E | 403 | AC-AUD-006 |
| A14 | Owner views audit log | E2E | View itself is audited | AC-AUD-006 |

**A7 is the test that matters most:** it proves the system refuses to act when it cannot record. A system that proceeds when auditing fails has no audit trail during incidents.

---

## 6. Authorisation tests

Every test in this section issues a **direct request** with an authenticated session of the wrong role. UI-level hiding is not tested because it is not a control.

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| S1 | Unauthenticated request to any business endpoint | E2E | 401 | AC-SEC-001 |
| S2 | Technician requests invoice list | E2E | 403 | AC-SEC-006 |
| S3 | Technician requests payment confirmation | E2E | 403 | AC-SEC-006 |
| S4 | Technician requests another technician's work order by ID | E2E | 403 or 404 (no data leak) | AC-TECH-009 |
| S5 | Technician requests company-wide project list | E2E | Scoped result only | AC-TECH-009 |
| S6 | `OPS_ADMIN` issues an invoice | E2E | 403 | AC-SEC-002 |
| S7 | `FINANCE` approves a purchase | E2E | 403 | AC-SEC-002 |
| S8 | Multi-role account (OPS_ADMIN + FINANCE) | E2E | Union of permissions applies; conflict rules still enforced | AC-SEC-003 |
| S9 | Session idle beyond limit | E2E | 401 on next request | AC-SEC-004 |
| S10 | Password changed | E2E | All other sessions revoked | AC-SEC-004 |
| S11 | Password hash inspection | Integration | Not plaintext; slow hash format | AC-SEC-005 |
| S12 | Secret scan of repository | Automated | No secrets present | AC-SEC-007 |
| S13 | Log inspection after error | Manual | No secrets in logs | AC-SEC-007 |
| S14 | Production TLS check | Manual | HTTP redirected/rejected | AC-SEC-008 |
| S15 | Denied sensitive action | Integration | Denial itself audited | AC-SEC-002 |
| S16 | Technician verifies own completed work | E2E | Refused without override | AC-TECH-008 |

**S4 deserves emphasis:** the test must confirm the response does not reveal whether the record exists. Returning "403 forbidden, this work order belongs to Budi" is a data leak dressed as an error message.

---

## 7. Project lifecycle tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| P1 | Create minimal project (no quotation, no PO, no materials, one technician) | E2E | Completes all lifecycle steps to closure | AC-PROJ-001 |
| P2 | Create project with non-CCTV service category | E2E | No CCTV-specific field required | AC-PROJ-002 |
| P3 | Project without quotation | E2E | No blocking, no forced reason | AC-PROJ-003 |
| P4 | Project without customer PO | E2E | No blocking | AC-PROJ-004 |
| P5 | Revise target completion date twice | Integration | Two revision rows; original values preserved | AC-PROJ-005 |
| P6 | Five-day project with daily reports | E2E | One work order, five reports, progress accumulates | AC-PROJ-006 |
| P7 | Assign three technicians, designate one leader | Integration | Three assignments; one leader | AC-PROJ-007 |
| P8 | Assign a second team leader | Integration | **Constraint violation** | AC-PROJ-007 |
| P9 | Attempt `DRAFT` → `IN_PROGRESS` | E2E | Refused; permitted next statuses named | AC-PROJ-011 |
| P10 | Attempt `CLOSED` → anything | E2E | Refused | AC-PROJ-011 |
| P11 | Attempt manual `PENDING_COMPLETION_VERIFICATION` | E2E | Refused (system-derived only) | AC-PROJ-011 |
| P12 | Close with open work order | E2E | Refused; unmet condition listed | AC-PROJ-010 |
| P13 | Close with material still issued | E2E | Refused | AC-PROJ-010 |
| P14 | Close with outstanding AR acknowledged | E2E | Permitted; AR appears in ageing | AC-PROJ-010 |
| P15 | Cancel project with issued material | E2E | Project cancelled; material still requires resolution; records retained | AC-PROJ-012 |
| P16 | Cancel project with outstanding payable | E2E | Payable persists | AC-PROJ-012 |

---

## 8. Material tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| M1 | Create requirement line | Integration | `stock_status = UNKNOWN` by default | AC-MATL-002 |
| M2 | Attempt reservation on `UNKNOWN` line | Integration | Refused with blocking condition named | AC-MATL-005 |
| M3 | Verify line with observed quantity | Integration | Status `VERIFIED`; verification row appended | AC-MATL-003 |
| M4 | Two concurrent reservations for last 10 units | Integration | Exactly one succeeds | AC-MATL-006 |
| M5 | Reservation across three projects exceeding stock | Integration | Third refused | AC-MATL-006 |
| M6 | Release reservation | Integration | Availability restored; other project can now reserve | AC-MATL-004 |
| M7 | Cancel project with active reservation | Integration | Reservation released automatically | AC-MATL-004 |
| M8 | Issue materials to technician | Integration | `ISSUE_OUT` movement; availability reduced | AC-MATL-008 |
| M9 | Consume issued material | Integration | `CONSUME` movement; issue line updated | AC-MATL-009 |
| M10 | Return unused material | Integration | `RETURN_IN` movement; availability restored | AC-MATL-010 |
| M11 | Attempt consume > issued | Integration | Refused by CHECK | AC-MATL-009 |
| M12 | Write off damaged material | Integration | `WRITE_OFF` movement with reason code; still visible in report | AC-MATL-011 |
| M13 | Attempt direct quantity edit via any endpoint | Integration | No such endpoint exists; 404/405 | AC-MATL-012 |
| M14 | Attempt direct SQL quantity update | Integration | No writable quantity column; ledger is append-only | AC-MATL-012 |
| M15 | Inject a ledger/cache variance | Integration | Reconciliation reports it; no auto-correction | AC-MATL-013 |
| M16 | Full issue lifecycle arithmetic | Integration | `consumed + returned + written_off ≤ issued` holds at all times | AC-MATL-010 |

---

## 9. Procurement tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| R1 | Create purchase request from shortfall | Integration | Lines link to requirement lines | AC-PROC-001 |
| R2 | Approve a request | Integration | `APPROVED` with actor, timestamp, policy snapshot | AC-PROC-002 |
| R3 | Reject a request | Integration | `REJECTED` with reason; cannot become PO | AC-PROC-002 |
| R4 | No approval policy configured | Integration | Approval required (fail-closed) | AC-PROC-003 |
| R5 | Threshold configured; request below | Integration | Approval not required; decision recorded as auto-permitted | AC-PROC-003 |
| R6 | Attempt PO from unapproved request | Integration | Refused | AC-PROC-005 |
| R7 | Issue PO | Integration | Number allocated; snapshot frozen; PDF stored | AC-PROC-006 |
| R8 | **Check available stock after PO issuance** | Integration | **Unchanged** | AC-PROC-005 |
| R9 | Post partial goods receipt | Integration | `RECEIPT_IN` for received quantity only; PO `PARTIALLY_RECEIVED` | AC-PROC-007 |
| R10 | Post three receipts totalling the order | Integration | Cumulative equals ordered; PO `RECEIVED` | AC-PROC-007 |
| R11 | Over-receipt without approval | Integration | Refused | AC-PROC-007 |
| R12 | Over-receipt with approval | Integration | Permitted; exception recorded | AC-PROC-007 |
| R13 | Receipt with damage discrepancy | Integration | Stock increases only by accepted quantity; discrepancy unresolved blocks line | AC-PROC-008 |
| R14 | Supplier invoice with price variance | Integration | Variance recorded; requires resolution | AC-PROC-008 |
| R15 | Partial payment of payable | Integration | `PARTIALLY_PAID`; outstanding correct | AC-FIN-004 |
| R16 | Record payment, do not confirm | E2E | Status `RECORDED`; appears in unconfirmed report | AC-FIN-009 |
| R17 | Confirm payment | E2E | Status `CONFIRMED` with confirmer identity | AC-FIN-009 |

**R8 is the single most important procurement test.** It directly verifies BR-MATL-005 — the rule whose violation causes on-site aborts.

---

## 10. Technician operations tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| T1 | Submit daily report on 360 px viewport | Manual | Completable; timed < 3 minutes (M3) | AC-TECH-004 |
| T2 | Submit report with photo | E2E | Report and attachment persisted; file private | AC-TECH-005 |
| T3 | Photo upload fails | E2E | Report still submits; photo marked failed with retry | AC-TECH-005 |
| T4 | Connection drop mid-form | Manual | Entered data preserved; resubmission offered | AC-TECH-004 |
| T5 | Double-submit report | E2E | One report created | AC-TECH-004 |
| T6 | Submit oversized file | E2E | Rejected server-side with clear message | AC-TECH-005 |
| T7 | Record blocker | E2E | Work order `BLOCKED`; appears in office blocked list | AC-TECH-006 |
| T8 | Resolve blocker | E2E | Returns to `IN_PROGRESS`; both events recorded | AC-TECH-006 |
| T9 | Reschedule work order | Integration | Reason required; previous dates preserved | AC-TECH-007 |
| T10 | Reassign work order | Integration | New technician sees it; old one loses access; **material custody not auto-transferred** | AC-TECH-007 |
| T11 | Partial completion: one of three technicians | E2E | Work order verified; project remains `IN_PROGRESS` | AC-TECH-008 |
| T12 | Verification rejected for rework | E2E | Returns to `IN_PROGRESS` with reason | AC-TECH-008 |
| T13 | Supersede a submitted report | E2E | New version; original retrievable | AC-TECH-004 |
| T14 | Work order with no report for one day | Manual | Appears in silence exception list | AC-TECH-006 |
| T15 | Progress display | E2E | Shown as estimate with last-updated timestamp | AC-TECH-010 |

---

## 11. Finance tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| F1 | Issue invoice | Integration | Number, snapshot, PDF, audit all present | AC-FIN-002 |
| F2 | Edit customer name after issuance, re-download PDF | Integration | **Byte-identical** to original (hash match) | AC-FIN-003 |
| F3 | Edit item price after issuance | Integration | Invoice totals unchanged | AC-FIN-003 |
| F4 | Attempt to edit issued invoice via any path | Integration | No path exists; 403/404 | AC-FIN-003 |
| F5 | Partial payment | Integration | `PARTIALLY_PAID`; outstanding = total − paid | AC-FIN-004 |
| F6 | Two payments completing the invoice | Integration | `PAID`; outstanding 0 | AC-FIN-004 |
| F7 | Allocation exceeding balance | Integration | Refused; remaining balance shown | AC-FIN-004 |
| F8 | Overpayment | Integration | Unapplied credit; no negative balance | AC-FIN-004 |
| F9 | Concurrent payments on one invoice | Integration | Serialised; total never exceeds balance | AC-FIN-004 |
| F10 | Overdue at due date boundary | Unit | Bucket correct on the day before, on, and after due date | AC-FIN-005 |
| F11 | Void invoice | Integration | Number retained; status `VOID`; PDF retrievable | AC-FIN-011 |
| F12 | Issue after void | Integration | Next number; voided number never reused | AC-FIN-011 |
| F13 | Void invoice with payments | E2E | Payments must be resolved first | AC-FIN-011 |
| F14 | Credit note and replacement | Integration | Original `CREDITED`; both new documents numbered; chain traceable | AC-FIN-012 |
| F15 | Project cost: receipt + invoice + payment | Integration | Counted **once** | AC-FIN-007 |
| F16 | Cost composition report | Integration | Shows source breakdown without duplication | AC-FIN-007 |
| F17 | Payment recorded, not confirmed | E2E | Distinct states; reported separately | AC-FIN-009 |
| F18 | AR ageing report at boundaries | Integration | Buckets correct | AC-RPT-004 |
| F19 | Re-run report for past range | Integration | Same result unless corrections occurred | AC-RPT-008 |
| F20 | Quotation issue | Integration | Numbered, PDF, no revenue effect | AC-FIN-001 |

**F2 is the test that proves historical immutability.** Comparing a hash rather than eyeballing the PDF is what makes it a real assertion.

---

## 12. Migration tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| D1 | Import CSV with malformed rows | Integration | Dry-run lists every bad row with row, column, reason | AC-DATA-001 |
| D2 | Dry-run alone | Integration | **No data written** | AC-DATA-002 |
| D3 | Commit after successful validation | Integration | Rows created; batch marked `COMMITTED` | AC-DATA-002 |
| D4 | Import duplicate historical invoice numbers | Integration | Commit blocked; both identified | AC-DATA-003 |
| D5 | Import historical number matching a would-be new number | Integration | Blocked; counter adjusted or import rejected | AC-DATA-004 |
| D6 | Delete a referenced customer | Integration | Refused | AC-DATA-005 |
| D7 | Deactivate a referenced customer | Integration | Permitted; historical records intact | AC-DATA-005 |
| D8 | Full export | Integration | All tables and files exported; re-importable | AC-DATA-006 |

---

## 13. Backup and recovery tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| B1 | Restore from latest backup into clean environment | Manual | Application starts; data matches source | AC-DATA-004 |
| B2 | Measure restore duration | Manual | ≤ 8 hours (RTO) | AC-DATA-005 |
| B3 | Verify backup recency | Manual | ≤ 24 hours (RPO) | AC-DATA-005 |
| B4 | Restore file storage | Manual | Attachments retrievable and hash-verified | AC-DOC-001 |
| B5 | Restore then verify audit chain | Manual | Chain intact after restore | AC-AUD-003 |
| B6 | Restore then issue an invoice | Manual | Numbering continues without collision | AC-DOC-004 |

**B5 and B6 are frequently forgotten.** A restore that loses audit integrity or causes a numbering collision is a restore that failed, even if the data "looks fine".

---

## 14. Non-functional tests

| # | Scenario | Level | Assertion | AC |
|---|---|---|---|---|
| X1 | 10 concurrent users performing state changes | Load | No errors, no constraint violations, p95 within target | AC-DATA-007 |
| X2 | Technician flow on 3G-class connection | Manual | p95 < 2 s | AC-OPS-004 |
| X3 | Office list views on LAN | Manual | p95 < 1 s | AC-OPS-004 |
| X4 | Device matrix (Chrome, Safari iOS, Firefox, Edge) | Manual | All flows usable | AC-OPS-008 |
| X5 | Error path shows Indonesian message, no stack trace | E2E | Message actionable, no internals | AC-OPS-006 |
| X6 | Environment separation check | Manual | No production data in staging | AC-OPS-001 |
| X7 | Diagnostic log inspection | Manual | A reported failure diagnosable from logs | AC-OPS-002 |

---

## 15. Regression discipline

| Rule | Reason |
|---|---|
| Every fixed defect gains a test that fails before the fix | Prevents recurrence |
| Numbering and audit tests run on every build | They protect the two highest-severity properties |
| Permission matrix tested per release | New endpoints are the usual source of leaks |
| Constraint tests run after every migration | A migration can silently drop a constraint |
| Restore rehearsal quarterly | Untested backups are not backups |

---

## 16. What is explicitly not tested, and why

| Not tested | Reason |
|---|---|
| Bank transfer execution | No banking integration exists |
| Statutory tax calculations | Out of scope (Q-FIN-01) |
| Native app behaviour | Out of scope (Q-UX-01) |
| Multi-currency | Out of scope |
| Offline-first sync | Not a requirement |
| Multi-location stock | Deferred (Q-INV-01) |

---

## 17. Cross-references

- Acceptance register: [`./ACCEPTANCE_CRITERIA.md`](./ACCEPTANCE_CRITERIA.md)
- Requirements: [`../PRD.md`](../PRD.md)
- Numbering tests in context: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §10
- Audit tests in context: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md) §5
- Phase mapping: [`./IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md)
