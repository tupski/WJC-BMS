# Implementation Plan — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed — planning only, no implementation has begun |
| Version | 0.1 |
| Scope owner | This document owns phasing, sequencing, prerequisites, deliverables, risks, and exit criteria |
| Requirement source | [`../PRD.md`](../PRD.md) |
| Repository basis | Verified empty: no code, no stack, no build or test commands exist |

---

## 1. Sequencing principle

Phases are ordered by **dependency**, not by visible business value. Two foundations must exist before anything that touches money:

| Foundation | Must precede | Why |
|---|---|---|
| **Audit logging** | Every approval, issuance, payment, override | Retrofitting audit into existing workflows means auditing code paths nobody remembers writing, and historical gaps that can never be filled |
| **Document numbering** | Any quotation, PO, or invoice issuance | The entire point of the project is to stop numbering failures; issuing documents before the numbering mechanism exists would repeat the original defect in the new system |

**[Confirmed]** the numbering conflict is the highest-severity known defect. It is therefore addressed in Phase 1, not Phase 5, even though invoices are a Phase 5 feature. The *mechanism* ships early; the *invoice screens* ship later.

A third dependency is less obvious but equally binding: **authorisation before data**. Building screens before the permission model exists means every screen is written without a policy check and must be revisited. Phase 1 establishes auth, roles, and audit; everything after inherits them.

---

## 2. Phase overview

| Phase | Theme | Key outcome | Requirement blocks |
|---|---|---|---|
| 0 | Requirements validation and process mapping | Confirmed answers, profiled data, agreed scope | — (enables all) |
| 1 | Foundations: auth, roles, master data, audit, numbering | Secure, audited, numbered platform | NFR-SEC-*, FR-AUD-*, FR-DOC-004/006, FR-CUST-001/002, FR-PROC-004 |
| 2 | Customers, quotations, projects, work orders, documents | Projects exist and are managed | FR-CUST-003…006, FR-PROJ-001…008/011/012, FR-FIN-001, FR-DOC-001…003/005/007 |
| 3 | Materials, procurement, approvals, goods receipt | Materials controlled end to end | FR-MATL-001…007/012, FR-PROC-001…008 |
| 4 | Technician scheduling, daily reports, material usage | Field operations captured | FR-TECH-001…010, FR-MATL-008…011/013, FR-PROJ-009/010 |
| 5 | Invoices, payments, payables, expenses, reports | Money tracked and chased | FR-FIN-002…012, FR-RPT-001…008 |
| 6 | Pilot, migration, UAT, go-live, stabilisation | Business running on the system | FR-DATA-001…006 |
| 7+ | Later phases | Centralised stock, profitability, accounting | Deferred (see §9) |

---

## 3. Phase 0 — Requirements validation and process mapping

**Purpose:** convert assumptions into facts before building. This phase produces no software, and skipping it is the most common cause of a system that fits the spec but not the business.

### Prerequisites
None.

### Activities

| # | Activity | Output |
|---|---|---|
| 0.1 | Walk through current process with owner and office staff | Process map of actual (not assumed) workflow |
| 0.2 | Collect samples of every spreadsheet, Word template, and WhatsApp report format | Data inventory |
| 0.3 | Profile spreadsheet structures, column consistency, duplicate rates | Data-quality report (answers Q-DATA-01) |
| 0.4 | Confirm client invoice-numbering constraints in writing | Answers to Q-NUM-01 (nine questions) |
| 0.5 | Confirm purchasing approval practice | Answers to Q-PROC-01 |
| 0.6 | Confirm supplier delivery and goods-receipt practice | Answers to Q-PROC-02 |
| 0.7 | Confirm technician/team-leader reporting model | Answers to Q-ROLE-01 |
| 0.8 | Consult accountant on tax and statutory needs | Answers to Q-FIN-01 |
| 0.9 | Confirm hosting, backup, retention policy | Answers to Q-OPS-01 |
| 0.10 | Measure baseline metrics | Answers to Q-BASE-01 |
| 0.11 | Validate mobile device and network reality with 2–3 technicians | Answers to A-01, Q-UX-01 input |

### Deliverables
- Process map (current state)
- Data inventory and profiling report
- Answered open-questions register
- Confirmed MVP scope (signed off)
- Baseline metrics for M1–M9 measurement

### Risks
| Risk | Mitigation |
|---|---|
| Process mapping reveals the business operates differently than the brief suggests | Re-baseline scope before Phase 1; this is the cheapest possible moment to discover it |
| Spreadsheets are unusable for migration | Scope a manual re-entry path; quantify the effort now |
| Owner unavailable for decisions | Schedule fixed decision sessions; document decisions in writing |

### Exit criteria
- [ ] All nine Q-NUM-01 questions answered in writing
- [ ] Q-PROC-01, Q-PROC-02, Q-ROLE-01, Q-FIN-01, Q-OPS-01 answered or explicitly deferred with a default accepted
- [ ] Spreadsheet profiling complete with a documented mapping specification
- [ ] MVP scope signed off by the owner
- [ ] Baseline metrics recorded or measurement method agreed
- [ ] At least one technician confirms the device/network assumption

**Satisfies:** enables all requirement blocks; no FR/NFR is completed here.

---

## 4. Phase 1 — Foundations

**Purpose:** build the platform that every later phase depends on. No business workflow ships in this phase, and that is intentional.

### Prerequisites
Phase 0 exit criteria met.

### Deliverables

| # | Deliverable | Requirement IDs |
|---|---|---|
| 1.1 | Application skeleton, environments (dev/staging/prod), deployment pipeline | NFR-OPS-001 |
| 1.2 | Database schema baseline, migration tooling, forward-only policy | NFR-DATA-006 |
| 1.3 | Authentication: password hashing, sessions, idle expiry, lockout | NFR-SEC-001, NFR-SEC-004, NFR-SEC-005 |
| 1.4 | Authorisation: roles, permissions, multi-role accounts, policy checks | NFR-SEC-002, NFR-SEC-003 |
| 1.5 | Audit logging: append-only table, transactional coupling, hash chain | FR-AUD-001…003, NFR-AUD-001…003 |
| 1.6 | Document numbering: series registry, atomic allocation, constraints | FR-DOC-004, FR-DOC-006, NFR-DATA-002 |
| 1.7 | Override mechanism with reason capture and owner report | FR-AUD-004, NFR-AUD-004 |
| 1.8 | Customer master + contacts | FR-CUST-001, FR-CUST-002 |
| 1.9 | Supplier master | FR-PROC-004 |
| 1.10 | Item catalogue, service categories | FR-PROJ-002 (category), FR-MATL-001 (item) |
| 1.11 | Technician records linked to accounts | FR-TECH-001 |
| 1.12 | Private object storage integration | FR-DOC-001 |
| 1.13 | Backup automation and first restore rehearsal | NFR-DATA-004, NFR-DATA-005 |

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Numbering implemented with a subtle race | Repeats the original defect | Concurrency tests written **with** the feature, not after (AC-DATA-002) |
| Audit coupling forgotten on some paths | Unattributable changes | Database trigger on high-risk tables; test asserting audit presence per transition |
| Over-engineering the permission model | Delay with no business value | Permission codes tied to actual §6 requirements; no speculative permissions |
| Phase feels like "no progress" | Stakeholder pressure to skip ahead | Demonstrate numbering and audit working via tests; show the concurrency proof |

### Exit criteria
- [ ] N concurrent issuance tests produce N distinct numbers, zero violations (M1)
- [ ] Direct DB duplicate insert is rejected by constraint
- [ ] Audit `UPDATE`/`DELETE` attempts fail with the application role
- [ ] A failed state change leaves no audit record and no state change
- [ ] Multi-role account permission union verified
- [ ] Technician account receives 403 on finance endpoints (even though those endpoints do not exist yet — the policy denies by default)
- [ ] Restore rehearsal completed with recorded duration
- [ ] Numbering format configurable without code change

**Satisfies:** FR-AUD-001…004, FR-CUST-001/002, FR-PROC-004, FR-TECH-001, FR-DOC-001/004/006, NFR-SEC-001…005/007/008, NFR-AUD-001…004, NFR-DATA-002/004/005/006, NFR-OPS-001.

---

## 5. Phase 2 — Customers, quotations, projects, work orders, documents

**Purpose:** projects exist as first-class entities with schedule, assignment, and document attachment.

### Prerequisites
Phase 1 exit criteria met (particularly numbering and audit).

### Deliverables

| # | Deliverable | Requirement IDs |
|---|---|---|
| 2.1 | Customer service categories, duplicate warning, customer 360 view | FR-CUST-003, FR-CUST-005, FR-CUST-006 |
| 2.2 | Customer PO reference capture | FR-CUST-004 |
| 2.3 | Quotation creation and issuance (numbered, PDF) | FR-FIN-001 |
| 2.4 | Project creation with optional links | FR-PROJ-001…004 |
| 2.5 | Schedule, revisions, multi-day support | FR-PROJ-005, FR-PROJ-006 |
| 2.6 | Assignment: one-to-many technicians, single team leader | FR-PROJ-007, FR-TECH-002 |
| 2.7 | Budget fields and cost display placeholder | FR-PROJ-008 |
| 2.8 | Project state machine with guarded transitions | FR-PROJ-011, FR-PROJ-012 |
| 2.9 | Work-order creation and assignment | FR-TECH-003 |
| 2.10 | PDF generation and immutable storage | FR-DOC-002, FR-DOC-003, FR-DOC-008 |
| 2.11 | Attachment upload with private storage and signed URLs | FR-DOC-001, FR-DOC-007 |
| 2.12 | Document retrieval by number with history | FR-DOC-005 |

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Forcing optional steps into the UI | Abandonment on small jobs | AC-PROJ-001 test: a minimal project must complete without touching optional screens |
| State machine leaks (illegal transitions possible) | Data inconsistency | Centralised transition function; no direct status writes; test every illegal transition |
| PDF template coupled to live data | Issued documents mutate | Snapshot-first rendering; test M4 |

### Exit criteria
- [ ] A minimal project (no quotation, no PO, no materials, one technician) is creatable and schedulable
- [ ] A project with a service category other than CCTV creates without any CCTV-specific field
- [ ] Every illegal transition is refused with permitted options named
- [ ] Schedule revision preserves previous values
- [ ] Only one team leader per project is possible (constraint verified)
- [ ] Quotation issued with number and stored PDF; re-download byte-identical after customer rename

**Satisfies:** FR-CUST-003…006, FR-PROJ-001…008/011/012, FR-TECH-002/003, FR-FIN-001, FR-DOC-001…003/005/007/008.

---

## 6. Phase 3 — Materials, procurement, approvals, goods receipt

**Purpose:** the material and purchasing chain, which is where the business currently has the least control.

### Prerequisites
Phase 2 (projects exist to attach requirements to); Phase 1 numbering (POs are numbered documents).

### Deliverables

| # | Deliverable | Requirement IDs |
|---|---|---|
| 3.1 | Material requirement lines with `UNKNOWN` default | FR-MATL-001, FR-MATL-002 |
| 3.2 | Stock verification action and history | FR-MATL-003 |
| 3.3 | Coverage calculation and shortfall exposure | FR-MATL-007 |
| 3.4 | Reservation lifecycle with release rules | FR-MATL-004, FR-MATL-006 |
| 3.5 | Reservation refusal on unverified lines | FR-MATL-005 |
| 3.6 | Append-only movement ledger | FR-MATL-012 |
| 3.7 | Purchase request creation and submission | FR-PROC-001 |
| 3.8 | Approval decision with policy engine (fail-closed) | FR-PROC-002, FR-PROC-003 |
| 3.9 | PO creation, issuance, snapshot, PDF | FR-PROC-005, FR-PROC-006 |
| 3.10 | Goods receipt with partial delivery and discrepancies | FR-PROC-007, FR-PROC-008 |
| 3.11 | Material readiness status derivation | PRD §9.3 |
| 3.12 | Procurement pipeline report | FR-RPT-005 |

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Reservation race across projects | Double-promising — the core failure | Locked evaluation + concurrency test (AC-MATL-006) |
| PO treated as stock increase | Same failure, different route | Explicit test: PO issuance leaves available quantity unchanged (AC-PROC-005) |
| Verification skipped under time pressure | Unknown quantities treated as available | Refusal is hard, not a warning; verification takes seconds |
| Partial receipt edge cases | Wrong cumulative quantities | Receipt-line arithmetic with CHECK constraints; test multi-delivery |

### Exit criteria
- [ ] New requirement line is `UNKNOWN` and cannot be reserved
- [ ] Two concurrent reservations for the last units: exactly one succeeds
- [ ] PO issuance does not change available stock
- [ ] Three partial receipts accumulate to the ordered quantity exactly
- [ ] Over-receipt is refused without explicit approval
- [ ] Discrepancy blocks line completion until resolved
- [ ] No API or UI path allows direct quantity editing
- [ ] Procurement pipeline report reflects every state accurately

**Satisfies:** FR-MATL-001…007/012, FR-PROC-001…008, FR-RPT-005.

---

## 7. Phase 4 — Technician operations

**Purpose:** replace WhatsApp reporting with structured, verified field data.

### Prerequisites
Phase 2 (work orders), Phase 3 (material issue requires requirements and stock).

### Deliverables

| # | Deliverable | Requirement IDs |
|---|---|---|
| 4.1 | Mobile-first daily report form (360 px, < 3 min) | FR-TECH-004, NFR-OPS-003 |
| 4.2 | Photo/file evidence with server-side limits | FR-TECH-005 |
| 4.3 | Material issue, consumption, return, write-off | FR-MATL-008…011 |
| 4.4 | Blocker recording and office surfacing | FR-TECH-006 |
| 4.5 | Reschedule and reassign with reasons | FR-TECH-007 |
| 4.6 | Partial completion and verification queue | FR-TECH-008 |
| 4.7 | Assignment-scoped access enforcement | FR-TECH-009, NFR-SEC-006 |
| 4.8 | Progress estimate display | FR-TECH-010 |
| 4.9 | Reconciliation checkpoint per project | FR-MATL-013 |
| 4.10 | Project closure with condition checking | FR-PROJ-009, FR-PROJ-010 |
| 4.11 | Daily progress and workload reports | FR-RPT-001, FR-RPT-006 |

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Report form too slow → WhatsApp continues | Total adoption failure | Timed usability test with real technicians on real devices (M3); iterate until under target |
| Photo upload fails and loses the report | Technicians stop trusting the system | Submit text independently of photos; local persistence until confirmed |
| Technician sees another technician's data | Privacy and trust breach | Structural scoping; negative tests per release |
| Verification backlog | Project cannot close | Verification queue as a primary office view; silence detection |

### Exit criteria
- [ ] Median report submission < 3 minutes on a 360 px device (M3)
- [ ] Report survives a simulated connection drop without data loss
- [ ] Photo failure does not block report submission
- [ ] Technician account cannot reach any other technician's assignment (verified by direct API call, not UI)
- [ ] Issue/consume/return arithmetic cannot exceed issued quantity
- [ ] Closure refused with enumerated unmet conditions
- [ ] Reconciliation detects an injected variance

**Satisfies:** FR-TECH-004…010, FR-MATL-008…011/013, FR-PROJ-009/010, FR-RPT-001/006, NFR-OPS-003.

---

## 8. Phase 5 — Finance

**Purpose:** billing, collection, and payables on top of a project record that is now trustworthy.

### Prerequisites
Phase 1 numbering (already built and tested), Phase 2 projects, Phase 3 goods receipts (payables), Phase 4 completion verification (billing trigger).

### Deliverables

| # | Deliverable | Requirement IDs |
|---|---|---|
| 5.1 | Invoice creation, issuance, snapshot, PDF | FR-FIN-002, FR-FIN-003 |
| 5.2 | Customer payments with allocation and partial payment | FR-FIN-004 |
| 5.3 | Outstanding balance and ageing | FR-FIN-005, FR-FIN-008 |
| 5.4 | Supplier payables and payment recording | FR-PROC-009, FR-PROC-010 |
| 5.5 | Payment confirmation as separate action | FR-FIN-009 |
| 5.6 | Operational and project expenses | FR-FIN-006 |
| 5.7 | Project cost accumulation without double counting | FR-FIN-007 |
| 5.8 | Void, credit note, replacement procedures | FR-FIN-011, FR-FIN-012 |
| 5.9 | AR/AP reports and project dashboard | FR-FIN-010, FR-RPT-002, FR-RPT-004 |
| 5.10 | Report export (CSV/PDF) and reproducibility | FR-RPT-007, FR-RPT-008 |

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Invoice issued before numbering is proven | Repeats the original defect | Numbering proven in Phase 1; concurrency tests re-run at Phase 5 |
| Double-counted project cost | Wrong pricing decisions | Unique source constraint; composition report makes it visible |
| Payment recorded but never confirmed | Hidden receivables | Confirmation as a separate queue; unconfirmed payments reported |
| Client rejects the number format | Billing blocked | Q-NUM-01 answered in Phase 0; format configurable per client |

### Exit criteria
- [ ] Invoice issuance under concurrency produces unique numbers (re-run M1)
- [ ] Issued invoice unchanged after customer and item master edits (M4)
- [ ] Partial payments produce accurate outstanding balances
- [ ] Allocation cannot exceed balance; overpayment becomes unapplied credit
- [ ] Payment recorded ≠ confirmed, enforced by permission and reflected in reports
- [ ] Void retains the number and the document remains retrievable
- [ ] Project cost composition shows no source counted twice
- [ ] Ageing buckets correct at boundary dates

**Satisfies:** FR-FIN-002…012, FR-PROC-009/010, FR-RPT-002/004/007/008.

---

## 9. Phase 6 — Pilot, migration, go-live

**Purpose:** move the business onto the system without losing history or trust.

### Prerequisites
Phases 1–5 complete; UAT scenarios prepared; Q-DATA-01 answered.

### Deliverables

| # | Deliverable | Requirement IDs |
|---|---|---|
| 6.1 | CSV import with validation dry-run and rejection report | FR-DATA-001, FR-DATA-002 |
| 6.2 | Historical document import with verbatim preservation | FR-DATA-003, FR-DATA-004 |
| 6.3 | Soft-deactivation enforcement and delete refusal | FR-DATA-005 |
| 6.4 | Full data export | FR-DATA-006 |
| 6.5 | Pilot with 2–3 technicians and 1 office user | — |
| 6.6 | UAT execution and sign-off | [`./ACCEPTANCE_CRITERIA.md`](./ACCEPTANCE_CRITERIA.md) |
| 6.7 | Backup-restore rehearsal in production-like environment | NFR-DATA-004, NFR-DATA-005 |
| 6.8 | Training materials in Indonesian | NFR-OPS-007 |
| 6.9 | Go-live and stabilisation period | — |

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Migration produces duplicate numbers | Recreates the core defect | Collision detection before commit; counter set above historical high-water mark |
| Pilot technicians revert to WhatsApp | Adoption failure | Pilot with volunteers first; fix friction before full rollout |
| Office staff continue maintaining spreadsheets in parallel | Two sources of truth | Agree an explicit cutover date; stop spreadsheet entry |
| Go-live during a busy period | Operational chaos | Schedule in a low-activity window |

### Exit criteria
- [ ] Import dry-run rejects every malformed row with a specific reason
- [ ] No imported historical number collides with any allocatable number
- [ ] UAT scenarios pass with business sign-off
- [ ] Restore rehearsal meets RPO ≤ 24 h and RTO ≤ 8 h (NFR-DATA-005)
- [ ] Pilot technicians complete reports without assistance
- [ ] Owner can answer "who approved this purchase" from the system in under a minute
- [ ] Spreadsheet entry formally stopped

**Satisfies:** FR-DATA-001…006, NFR-DATA-004/005, NFR-OPS-007.

---

## 10. Later phases (not MVP)

| Phase | Content | Trigger |
|---|---|---|
| 7 | Centralised stock: locations, transfers, opening balances | Any trigger in [`../workflows/PROCUREMENT_AND_MATERIALS.md`](../workflows/PROCUREMENT_AND_MATERIALS.md) §9.3 |
| 8 | Project profitability analytics | ≥2 months of reliable cost data |
| 9 | Formal accounting (L3) | Q-FIN-01 answered affirmatively |
| 10 | Integrations (accounting software, WhatsApp Business API) | Named business need |
| 11 | Native mobile app | Measured web friction (Q-UX-01) |
| 12 | Customer/supplier portals | Volume justifies |

**No later phase is authorised by this document.** Each requires explicit approval (see [`../../AGENTS.md`](../../AGENTS.md)).

---

## 11. Dependency graph

```
Phase 0 (validation)
    │
    ▼
Phase 1 (auth · audit · numbering · master data)
    │
    ├──────────────► Phase 2 (customers · projects · work orders · documents)
    │                      │
    │                      ▼
    │                Phase 3 (materials · procurement · receipts)
    │                      │
    │                      ▼
    │                Phase 4 (technician operations · issue/return · closure)
    │                      │
    └──────────────────────┴──────► Phase 5 (invoices · payments · reports)
                                          │
                                          ▼
                                    Phase 6 (migration · pilot · go-live)
```

**Hard edges:**
- Numbering (1.6) → any issuance (2.3, 3.9, 5.1)
- Audit (1.5) → any approval or override (3.8, 5.5)
- Goods receipt (3.10) → payables (5.4)
- Completion verification (4.6) → invoicing trigger (5.1)

---

## 12. Effort and capacity considerations

**[Assumption]** delivery capacity is small (one or two developers part-time). This plan is sized accordingly:

| Phase | Relative size | Notes |
|---|---|---|
| 0 | Small but decisive | Mostly interviews and data profiling |
| 1 | Medium | Foundations are unglamorous and easy to under-estimate; numbering and audit need real tests |
| 2 | Large | Many screens, state machines, PDF |
| 3 | Large | Highest correctness density |
| 4 | Large | Field UX requires iteration with real users |
| 5 | Large | Financial correctness and reporting |
| 6 | Medium | Migration and support are people-heavy |

**No calendar estimates are given.** Estimating without a team, stack, or velocity would be invented data. The plan states sequence and relative size; schedule estimation belongs to whoever is authorised to build it.

---

## 13. Cross-references

- Requirements: [`../PRD.md`](../PRD.md)
- Acceptance criteria: [`./ACCEPTANCE_CRITERIA.md`](./ACCEPTANCE_CRITERIA.md)
- Test approach: [`./TESTING_STRATEGY.md`](./TESTING_STRATEGY.md)
- Architecture: [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md)
- Decisions: [`../decisions/`](../decisions/)
