# Product Requirements Document — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Document status | Draft for business sign-off |
| Version | 0.1 |
| Owner | Product / Business Analysis |
| Authoritative scope source | **This document** (see [`../AGENTS.md`](../AGENTS.md)) |
| Last updated | 2026-10-25 |
| Classification key | **[Confirmed]** / **[Proposed]** / **[Assumption]** / **[Open Question]** / **[Out of Scope]** |

This PRD is the single authoritative source of truth for scope and functional requirements. Other documents in [`./`](./README.md) elaborate on their own subject and cross-reference requirement IDs defined here; they do not redefine them.

---

## 1. Document conventions

### 1.1 Requirement identifiers

| Prefix | Meaning | Owned by |
|---|---|---|
| `FR-<AREA>-NNN` | Functional requirement | This document |
| `NFR-<AREA>-NNN` | Non-functional requirement | This document |
| `AC-<AREA>-NNN` | Acceptance criterion | [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md) |
| `ADR-NNN` | Architecture decision record | [`./decisions/`](./decisions/) |
| `BR-<AREA>-NNN` | Business rule | This document (§8) |

Areas: `CUST` (customer), `PROJ` (project), `PROC` (procurement), `MATL` (materials), `TECH` (technician operations), `FIN` (finance), `DOC` (documents), `RPT` (reporting), `DATA` (data migration), `SEC` (security), `AUD` (audit), `OPS` (operations/deployment).

IDs are **stable and never reused**. A withdrawn requirement is marked `WITHDRAWN` in place, keeping its number.

### 1.2 Classification labels

Every non-obvious claim carries exactly one label:

- **[Confirmed]** — stated in the client brief; treat as fact.
- **[Proposed]** — our recommendation, awaiting sign-off.
- **[Assumption]** — believed true, not verified; must be validated or it may invalidate dependent requirements.
- **[Open Question]** — unresolved; a decision owner and a decision trigger are named.
- **[Out of Scope]** — explicitly excluded, with rationale.

An assumption is **never** silently promoted to a requirement. Where an assumption currently gates a requirement, the requirement states the dependency.

### 1.3 Repository basis

The workspace was verified empty before authoring: zero entries including hidden files, no `.git`, no existing `AGENTS.md`, no `docs/`. Therefore:

- There is **no established technology stack**, **no codebase**, **no build or test commands**, and **no existing coding conventions**.
- Every technical recommendation in this document set is **[Proposed]**, not installed or verified.
- No repository fact is asserted that could not be derived from the client brief or from this empty-repository finding.

### 1.4 Language

**[Proposed]** The application user interface uses **Indonesian** as its primary language, including navigation, field labels, validation messages, status names displayed to users, and generated PDF documents. Internal identifiers, database column names, API field names, and this documentation set remain in English. A full i18n framework is **[Out of Scope]** for MVP — the UI is Indonesian-only, with message strings centralised so translation is possible later without a rewrite. This is a planning assumption: no repository evidence exists to confirm it, and it should be confirmed at Phase 0 sign-off.

---

## 2. Product overview

**Working name:** WJ-CCTV Business Operations System
**Category:** Business Management System (BMS) — integrated project management, lightweight Field Service Management, procurement, project-based material control, document management, and operational finance.

### 2.1 Problem statement

WJ-CCTV.com delivers CCTV-related services and other business services. Its operations — office administration, purchasing, suppliers, project execution, field technicians, invoicing, and financial administration — currently run on disconnected general-purpose tools: Excel/Google Sheets, Word, and WhatsApp.

**[Confirmed]** operating conditions:

- Approximately 3 office employees and approximately 15 technicians.
- Administration relies on spreadsheets, word-processing documents, and WhatsApp groups.
- No centralised inventory system exists; materials are prepared and tracked per project.
- Not all required materials are on hand at office/warehouse; some must be purchased from suppliers before work can proceed.
- Office administration handles purchasing and purchase-order preparation.
- The owner sometimes approves purchases and may personally transfer funds to suppliers.
- The owner assigns technicians per project requirement — sometimes one technician, sometimes a larger team.
- Projects can span multiple days.
- Technicians report progress through WhatsApp groups.
- An invoice-numbering conflict has occurred. Its exact cause is **unconfirmed**.

### 2.2 Consequences of the current state

These are the business failures the system must remove. Each maps to requirements in §6.

| # | Consequence | Evidence status | Addressed by |
|---|---|---|---|
| P1 | Invoice numbers can collide or be rejected by a client, blocking billing and payment | **[Confirmed]** that a conflict occurred; cause **[Open Question]** | §6.5, [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md) |
| P2 | Material availability per project is unknown until someone physically checks, delaying work | **[Confirmed]** materials must sometimes be bought before work proceeds | FR-MATL-002, FR-MATL-005 |
| P3 | The same material can be implicitly promised to two projects at once | **[Assumption]** — no reservation concept exists today, so double-promising is structurally possible | FR-MATL-004, FR-MATL-006 |
| P4 | Purchase approval, PO issuance, delivery, and payment are indistinguishable in conversation history | **[Confirmed]** that these are separate real-world steps; conflation is **[Assumption]** | FR-PROC-002, FR-PROC-007 |
| P5 | Multi-day progress lives in chat scrollback; no reliable project state, no history | **[Confirmed]** | FR-TECH-004, FR-PROJ-011 |
| P6 | Cost and margin per project cannot be reconstructed after the fact | **[Assumption]** | FR-FIN-007, FR-RPT-002 |
| P7 | No audit trail: who changed a number, who approved a purchase, who issued an invoice | **[Assumption]** | FR-AUD-001 |
| P8 | Knowledge is held by individuals, not the system | **[Assumption]** | FR-DATA-001, FR-RPT-001 |

**[Open Question] Q-BASE-01 — Baseline metrics.** Current volumes (projects/month, invoices/month, average project duration, current invoice-conflict frequency) are unknown. Proposed measurement method: Phase 0 extracts 3 months of spreadsheets and counts distinct projects, invoices issued, and documented conflicts; these become the baseline against which §3 targets are measured. No baseline figures are invented here.

### 2.3 Product vision

A single centralised, auditable system of record for project delivery and its money: what was promised, what was bought, what was used, who did the work, what was billed, and what was paid — each step evidenced, timestamped, attributable, and impossible to silently overwrite.

---

## 3. Goals and measurable outcomes

### 3.1 Goals

| ID | Goal | Rationale |
|---|---|---|
| G1 | Eliminate internal duplicate document numbers | Highest-severity confirmed defect |
| G2 | Make material availability per project explicit before work starts | Prevents site aborts and rework |
| G3 | Make procurement states unambiguous | Stops "approved" being mistaken for "paid" |
| G4 | Replace WhatsApp progress reporting with structured, phone-friendly reporting | Removes scrollback dependency |
| G5 | Produce a per-project cost and billing trail | Enables margin visibility |
| G6 | Make every state change attributable and reversible only by documented procedure | Auditability and dispute defence |
| G7 | Keep operating overhead low enough for ~3 office staff | Adoption risk is the dominant failure mode |

### 3.2 Measurable outcomes

Targets are stated as verifiable conditions. Where a numeric baseline is unknown, the target is expressed as an absolute or a measurement procedure, not a fabricated percentage improvement.

| ID | Outcome | Measurement method | Status |
|---|---|---|---|
| M1 | **0** duplicate document numbers across concurrent issuance tests | Automated concurrency test: N parallel issuance requests on one series; assert N distinct numbers, 0 constraint violations | **[Proposed]** |
| M2 | **0** invoice numbers may be produced by client-side code | Code review assertion + architecture test that issuance endpoint is the only writer of `document_number` | **[Proposed]** |
| M3 | Daily progress report produced in **< 3 minutes** on a phone, from project open to submit | Timed usability test with 3 technicians on their own devices, median of 5 attempts | **[Proposed]** |
| M4 | **100%** of issued invoices retrievable with their original issuance snapshot, byte-identical PDF, after any later master-data edit | Regression test: issue invoice → edit customer name and product price → re-download → diff against stored hash | **[Proposed]** |
| M5 | **100%** of material reservations traceable to a verification event or an approved purchase | Query audit: count reservations lacking a `stock_verification` or `purchase_order` link; must be 0 | **[Proposed]** |
| M6 | **0** technician accounts able to read company-wide financial data | Permission test matrix executed per release; assert 403 on every finance endpoint for technician role | **[Proposed]** |
| M7 | Every approval, issuance, cancellation, and payment confirmation present in the audit log within the same transaction as the state change | Integration test: force failure after state write, assert no orphan audit gap; and assert audit row exists for every transition | **[Proposed]** |
| M8 | Backup restore verified quarterly with a documented recovery time | Restore rehearsal into a clean environment; record actual RTO against NFR-DATA-005 | **[Proposed]** |
| M9 | **0** projects in `IN_PROGRESS` with an unverified material line marked as available | Daily reconciliation report; must return 0 rows | **[Proposed]** |

**Explicitly not targeted in MVP:** reducing headcount, replacing formal accounting software, real-time GPS tracking, or bank integration. See §5.3.

---

## 4. Users, roles, and representative workflows

### 4.1 Roles

Roles are **permission bundles**, not job titles. One account may hold several roles (NFR-SEC-003). The authoritative permission matrix lives in [`./architecture/SECURITY_AND_ACCESS_CONTROL.md`](./architecture/SECURITY_AND_ACCESS_CONTROL.md); this section defines *who exists*, not *what they may do*.

| Role | Who | Typical scope | Notes |
|---|---|---|---|
| `OWNER` | Business owner | Everything, including approvals, permission changes, audit log | **[Confirmed]** owner approves purchases and may transfer funds personally |
| `OPS_ADMIN` | Operations administrator (office) | Customers, projects, work orders, material requirements, purchase requests, PO preparation, scheduling | **[Confirmed]** office administration handles purchasing and PO preparation |
| `FINANCE` | Finance / administrative finance | Invoices, payments, payables, expenses, AR/AP reports | **[Assumption]** that a distinct finance person exists; may be combined with `OPS_ADMIN` in practice |
| `TECHNICIAN` | Field technician (~15 people) | Own assignments, own daily reports, own material issue/return records | **[Confirmed]** population size; **[Assumption]** that access needs are non-uniform |
| `TEAM_LEADER` | Senior technician leading a team | Technician scope + review of team members' daily reports + team-level material requests | **[Open Question] Q-ROLE-01** — whether this role is warranted |

**[Open Question] Q-ROLE-01 — Team leader role.** The brief confirms the owner assigns "one technician or a larger team" but does not confirm that a designated team leader exists, nor whether they review others' reports. **Proposed default, to keep planning moving:** include `TEAM_LEADER` as an *optional* role assignment on a project (FR-TECH-002) rather than a separate job function, so a project can have a lead without inventing an org chart. Rationale: multi-day, multi-person projects need a single accountable person on site, and the review step is cheap to add now but expensive to retrofit into the report workflow. **Decision trigger:** Phase 0 process mapping — if the owner states that all technicians report directly to the office, drop the review step and keep only the lead designation for accountability.

**[Assumption] A-ROLE-02 — Non-uniform technician access.** The brief warns against assuming all 15 technicians need identical access. This PRD therefore treats technician visibility as **assignment-scoped by default** (FR-TECH-009), with no company-wide read access for any technician. If some technicians genuinely need broader access (e.g. a senior who also quotes), that is achieved by granting an additional role to that account, not by widening the technician role.

**[Assumption] A-ROLE-03 — Combined office roles.** With ~3 office employees, at least one person will hold `OPS_ADMIN` and `FINANCE` simultaneously. The system must support this without pretending a separation of duties exists that does not. Where one person holds conflicting permissions, a compensating control is required (see [`./architecture/SECURITY_AND_ACCESS_CONTROL.md`](./architecture/SECURITY_AND_ACCESS_CONTROL.md) §6).

### 4.2 Representative workflows

Narratives only; the authoritative step-by-step flows and state machines are in [`./workflows/`](./workflows/).

**W1 — Multi-day installation with procurement.** A customer requests work → office records the customer and creates a project with a service category, location, planned start, and target completion → material requirements are listed → the system shows some lines as stock-status `UNKNOWN` → office verifies on-hand quantity for those lines → shortfall becomes a purchase request → owner approves → PO issued to supplier → goods arrive, possibly in two deliveries → materials issued to the assigned technician and team → technician submits a daily report each day with photos → final day: technician marks work order complete → office verifies → invoice issued from an immutable snapshot → customer pays in two instalments → project closed.

**W2 — Small single-technician job with no purchasing.** Office creates a project for a service that needs no purchased materials → assigns one technician → technician reports and completes in one day → invoice issued → payment recorded → closed. **No quotation, no customer PO, no purchase request, and no material lines are required.** The system must not force these steps (BR-PROJ-001).

**W3 — Owner-funded direct purchase.** Office raises a purchase request → owner approves → owner transfers funds personally and records the transfer → office confirms receipt → supplier delivers → goods receipt recorded → supplier invoice recorded as payable → payment confirmation closes the payable. The system records the transfer; it **does not** move money (FR-PROC-010).

**W4 — Correction after issuance.** An issued invoice has a wrong line quantity → office does **not** edit it → office issues a credit note and a replacement invoice, each with its own number and its own audit trail → original remains retrievable and marked `CREDITED`. (BR-FIN-004, FR-FIN-011, FR-FIN-012.)

---

## 5. Scope

### 5.1 MVP scope

MVP delivers one coherent loop: customer → project → materials/procurement → field execution → invoice → payment → closure, with audit and document control underneath.

| Capability | Included in MVP | Requirement block |
|---|---|---|
| Authentication, roles, multi-role accounts | Yes | NFR-SEC-001…008 |
| Customer and contact records | Yes | FR-CUST-001…006 |
| Projects with categories, schedule, assignment | Yes | FR-PROJ-001…012 |
| Quotations and customer POs | Yes, **optional per project** | FR-FIN-001, FR-CUST-004 |
| Project material requirements + reservation | Yes | FR-MATL-001…013 |
| Purchase request → approval → PO → goods receipt | Yes | FR-PROC-001…010 |
| Work orders, daily reports, photos, multi-day progress | Yes | FR-TECH-001…010 |
| Sales invoices, payments, AR | Yes | FR-FIN-002…005 |
| Supplier invoices/payables, operational expenses | Yes | FR-FIN-006, FR-FIN-009 |
| Project cost accumulation | Yes (operational, not ledger) | FR-FIN-007 |
| Private document storage + PDF generation | Yes | FR-DOC-001…008 |
| Audit logging | Yes — **foundation phase** | FR-AUD-001…006 |
| Document numbering | Yes — **foundation phase** | FR-DOC-004, [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md) |
| Spreadsheet migration of master data + open items | Yes | FR-DATA-001…006 |
| Management reports | Yes, limited set | FR-RPT-001…008 |
| Mobile-first responsive web | Yes | NFR-OPS-003 |

### 5.2 Later phases

| Capability | Phase | Trigger |
|---|---|---|
| Centralised stock with warehouse locations | Later | Multiple concurrent projects repeatedly contend for the same stock, or stock value exceeds a threshold the owner sets (**[Open Question] Q-INV-01**) |
| Project profitability analytics | Later | Cost data has been captured reliably for ≥2 months |
| Formal accounting (journals, ledgers, tax reporting, statutory statements) | Later / possibly never | Accountant or tax authority requires statements the operational layer cannot produce (**[Open Question] Q-FIN-01**) |
| Native mobile application | Later | Measured web friction on real devices, not preference |
| Supplier portal, customer self-service portal | Later | Volume justifies it |
| Integrations (accounting software, payment gateways, WhatsApp Business API) | Later | Named business need |
| Multi-branch / multi-company | Later | Second legal entity or location exists |

### 5.3 Explicit exclusions

| Excluded | Rationale | Label |
|---|---|---|
| Full ERP (manufacturing, HR/payroll, CRM campaigns, general ledger) | Not justified by ~3 office staff and 15 technicians; scope would not survive delivery | **[Out of Scope]** |
| Bank integration, payment initiation, automatic transfer | The app records and confirms transfers; it never moves money. No banking API in MVP | **[Out of Scope]** |
| Full warehouse management (bin locations, cycle counting, lot/serial tracking, barcode scanning) | Project-scoped material control is sufficient for MVP; see Q-INV-01 for the trigger to expand | **[Out of Scope]** |
| Microservices, event streaming, message brokers | No scale or team-size justification; adds operational burden | **[Out of Scope]** |
| Native iOS/Android app | Mobile-first responsive web covers the need; a native app doubles build and release cost | **[Out of Scope]** |
| Real-time GPS technician tracking | Not requested; privacy cost | **[Out of Scope]** |
| Customer-facing portal / login | Not requested | **[Out of Scope]** |
| Payroll, attendance, leave management | Distinct problem domain | **[Out of Scope]** |
| Multi-currency, multi-language UI | Single currency, Indonesian UI for MVP | **[Out of Scope]** |
| Statutory tax filing / e-invoicing integration | Depends on Q-FIN-01 and client-side validation rules (Q-NUM-01) | **[Out of Scope]** |

---

## 6. Functional requirements

Each requirement states expected behaviour and points to its acceptance criteria. Criteria live **only** in [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md) to avoid duplication; the `AC-` reference column is the contract.

### 6.1 Customer management (`FR-CUST`)

| ID | Requirement | AC |
|---|---|---|
| FR-CUST-001 | The system shall maintain a customer master record with at least: display name, customer type (individual / company), primary phone, address, and active/inactive status. Records are **deactivated, never hard-deleted**, once referenced by a project or document. | AC-CUST-001 |
| FR-CUST-002 | A customer may have multiple contact persons, exactly one of which is flagged primary. A contact holds name, role/title, phone, and optional email. | AC-CUST-002 |
| FR-CUST-003 | Each customer shall carry one or more **service categories** describing the kinds of work they buy (e.g. CCTV installation, maintenance, networking, other services). Categories are configurable master data, not a fixed enumeration in code. | AC-CUST-003 |
| FR-CUST-004 | The system shall optionally record a customer purchase-order reference (number, date, and an uploaded copy) against a project, where the customer issues one. This is optional; a project without a customer PO is valid. | AC-CUST-004 |
| FR-CUST-005 | A customer record shall present a consolidated view of that customer's projects, issued invoices, outstanding balances, and document history. | AC-CUST-005 |
| FR-CUST-006 | On creating a customer, the system shall warn when a probable duplicate exists (normalised name similarity or identical primary phone) and require explicit confirmation to proceed. It shall not silently block creation. | AC-CUST-006 |

### 6.2 Project management (`FR-PROJ`)

| ID | Requirement | AC |
|---|---|---|
| FR-PROJ-001 | The system shall create a project with: customer, service category, title, site location, planned start date, target completion date, and assigned owner-of-record among office staff. | AC-PROJ-001 |
| FR-PROJ-002 | Service category is a first-class field on the project and drives no hard-coded CCTV assumptions. No workflow step, field, or report may require CCTV-specific data (e.g. camera count, channel count) as a condition of project completion. Category-specific fields, if any, are optional attributes. | AC-PROJ-002 |
| FR-PROJ-003 | A project may optionally reference a quotation. Absence of a quotation is valid and shall not block any later step. | AC-PROJ-003 |
| FR-PROJ-004 | A project may optionally reference a customer PO (FR-CUST-004). Absence shall not block any later step. | AC-PROJ-004 |
| FR-PROJ-005 | The system shall record planned start date, target completion date, and (where known) a per-day or per-phase schedule. Target completion may be revised, and each revision is recorded with reason, actor, and timestamp. | AC-PROJ-005 |
| FR-PROJ-006 | The system shall support projects spanning an arbitrary number of days. Progress is expressed as accumulated daily reports and work-order completion, never as a single binary completed flag. | AC-PROJ-006 |
| FR-PROJ-007 | The system shall assign one or more technicians to a project, and may designate at most one of them as team leader for that project. Assignment and unassignment are recorded with actor and timestamp. | AC-PROJ-007 |
| FR-PROJ-008 | The system shall record a project budget (estimated revenue and estimated material cost) where the business has one, and display actual accumulated cost against it. Budget is optional. | AC-PROJ-008 |
| FR-PROJ-009 | The system shall derive project material requirements from the project's scope and expose them for verification, reservation, and purchasing (see FR-MATL-001). | AC-PROJ-009 |
| FR-PROJ-010 | A project may move to `CLOSED` only when all closure conditions in BR-PROJ-004 are satisfied. The system shall refuse closure and enumerate the unmet conditions. | AC-PROJ-010 |
| FR-PROJ-011 | The system shall enforce the project status state machine in §9.1. Transitions outside the allowed set are rejected with the current status and the permitted next statuses named in the error. | AC-PROJ-011 |
| FR-PROJ-012 | The system shall support suspending a project (`ON_HOLD`) and cancelling a project (`CANCELLED`), each requiring a reason code and free-text reason. Cancellation does not delete the project, its documents, or its financial records. | AC-PROJ-012 |

### 6.3 Procurement (`FR-PROC`)

| ID | Requirement | AC |
|---|---|---|
| FR-PROC-001 | The system shall create a purchase request containing one or more lines, each linked to a project material requirement line and a supplier (or a supplier to be determined). A purchase request aggregates shortfalls across requirements and may serve more than one project line. | AC-PROC-001 |
| FR-PROC-002 | The system shall record an approval decision on a submitted purchase request as exactly one of `APPROVED` or `REJECTED`, capturing the deciding user, timestamp, and reason. A rejected request cannot proceed to PO. Approval is a decision, not a purchase, not a delivery, and not a payment. | AC-PROC-002 |
| FR-PROC-003 | Approval requirements shall be driven by a **configurable policy** (e.g. amount thresholds, supplier category, project flag), not hard-coded. Where no policy is configured, the system shall require approval by default (fail-closed). | AC-PROC-003 |
| FR-PROC-004 | The system shall maintain a supplier master record (name, contact, payment terms, active status) with soft deactivation, and shall support multiple suppliers per material category where the business does. | AC-PROC-004 |
| FR-PROC-005 | The system shall generate a supplier purchase order from an approved purchase request only. PO lines inherit quantities and prices, which may be adjusted before issuance with the change recorded. | AC-PROC-005 |
| FR-PROC-006 | Issuing a PO shall allocate a document number per [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md), freeze a snapshot of supplier and line data, and produce a PDF. Issuance is irreversible except by cancellation (FR-FIN-011). | AC-PROC-006 |
| FR-PROC-007 | The system shall record goods receipt against a PO at line level, supporting **partial** receipt across multiple events. Received quantity is cumulative and may never exceed ordered quantity without an explicit recorded over-receipt approval. | AC-PROC-007 |
| FR-PROC-008 | Goods receipt shall capture discrepancies (short delivery, damaged goods, wrong item, price variance) as recorded exceptions that require resolution before the PO line is considered fully received. | AC-PROC-008 |
| FR-PROC-009 | The system shall record a supplier invoice as a payable, linked to one or more goods receipts or PO lines, with amount, due date, and an optional attached file. A supplier invoice may exist before or after payment. | AC-PROC-009 |
| FR-PROC-010 | The system shall record a payment to a supplier with amount, date, method, reference, and the payables it settles, and shall support **partial** payment. Recording a payment does **not** move money; the system has no banking integration and must never imply that it executed a transfer. A separate confirmation step by an authorised user marks the payment confirmed. | AC-PROC-010 |

### 6.4 Project materials (`FR-MATL`)

| ID | Requirement | AC |
|---|---|---|
| FR-MATL-001 | The system shall record material requirement lines per project: item, required quantity, unit, and an optional specification note. Lines are independent of whether the item is in stock. | AC-MATL-001 |
| FR-MATL-002 | Every material requirement line shall carry an explicit **stock status** of `UNKNOWN` or `VERIFIED`. A new line defaults to `UNKNOWN`. The system shall never treat an `UNKNOWN` quantity as available. | AC-MATL-002 |
| FR-MATL-003 | A verification action shall record the observed on-hand quantity for a line, the verifying user, timestamp, and an optional note. Verification sets the line to `VERIFIED` with the observed quantity. | AC-MATL-003 |
| FR-MATL-004 | The system shall support a reservation lifecycle on verified quantity: create, release, and consume. Each transition is an append-only movement record. | AC-MATL-004 |
| FR-MATL-005 | The system shall refuse to create a reservation that would be treated as "covered" while the line's stock status is `UNKNOWN`. The refusal names the blocking condition and the required action. | AC-MATL-005 |
| FR-MATL-006 | Reservations shall be evaluated against total on-hand quantity net of reservations held by other projects. The system shall never allow reservations across projects to exceed verified on-hand quantity. | AC-MATL-006 |
| FR-MATL-007 | The system shall compute quantity-to-purchase per line as required quantity minus verified available quantity net of competing reservations, and expose that shortfall to procurement (FR-PROC-001). | AC-MATL-007 |
| FR-MATL-008 | The system shall record issue of materials to a named technician for a named project, reducing available quantity and creating an issue record. | AC-MATL-008 |
| FR-MATL-009 | The system shall record consumption of issued materials against a project or work order. | AC-MATL-009 |
| FR-MATL-010 | The system shall record return of unused issued materials to stock or to the office, restoring availability and closing the issue record. | AC-MATL-010 |
| FR-MATL-011 | The system shall record damage, loss, and write-off with a reason code, keeping the quantity traceable rather than silently vanishing. | AC-MATL-011 |
| FR-MATL-012 | All quantity changes shall be recorded as append-only movement rows. The system shall provide no user-facing or API path that directly edits a stored quantity field. A correction is a compensating movement, not an overwrite. | AC-MATL-012 |
| FR-MATL-013 | The system shall provide a reconciliation checkpoint per project (and later per stock location) that recomputes expected quantity from the movement ledger and reports variance against recorded balances. | AC-MATL-013 |

### 6.5 Technician operations (`FR-TECH`)

| ID | Requirement | AC |
|---|---|---|
| FR-TECH-001 | The system shall maintain a technician record (name, phone, active status, skills/categories where known) linked to exactly one user account. | AC-TECH-001 |
| FR-TECH-002 | The system may designate one technician per project as team leader for that project, granting report-review scope over that project's team only. | AC-TECH-002 |
| FR-TECH-003 | The system shall create an individual work order per technician per project, holding planned dates, scope description, and its own status. | AC-TECH-003 |
| FR-TECH-004 | A technician shall submit a daily work report from a mobile browser containing: date, project, work performed (free text), progress indicator, optional material usage, and optional photos. Submission shall be possible in under 3 minutes (M3). | AC-TECH-004 |
| FR-TECH-005 | The system shall accept photo and file evidence on a daily report, storing files privately (FR-DOC-001), with per-file size and count limits enforced server-side. | AC-TECH-005 |
| FR-TECH-006 | The system shall record blockers, delays, and suspension reasons on a daily report or work order, with a reason code, and shall surface blocked work orders to office staff. | AC-TECH-006 |
| FR-TECH-007 | The system shall support rescheduling a work order (new planned dates, reason) and reassigning it to a different technician, both recorded with actor and timestamp. | AC-TECH-007 |
| FR-TECH-008 | The system shall support partial completion: a work order may be `COMPLETED_PENDING_VERIFICATION` while the project remains `IN_PROGRESS`, and verification may accept or return it for rework with a reason. | AC-TECH-008 |
| FR-TECH-009 | A technician shall see only projects and work orders to which they are assigned, and only their own reports. No technician-facing view shall expose company-wide financial, supplier, or other technicians' personal data. | AC-TECH-009 |
| FR-TECH-010 | A work order shall carry a progress percentage that is derived from reported progress and completion events, and is displayed as an estimate with its last-updated timestamp — never presented as an authoritative measure of project completion. | AC-TECH-010 |

### 6.6 Finance and documents (`FR-FIN`, `FR-DOC`)

| ID | Requirement | AC |
|---|---|---|
| FR-FIN-001 | The system shall create and issue quotations with lines, validity period, and status, and shall produce a PDF on issuance. Quotations do not affect revenue or receivables until accepted and invoiced. | AC-FIN-001 |
| FR-FIN-002 | The system shall issue sales invoices against a project, with lines, tax/discount fields where configured, due date, and an allocated document number from the invoice series. | AC-FIN-002 |
| FR-FIN-003 | On issuance, the system shall freeze an immutable snapshot of the invoice: customer identity and address, line descriptions, quantities, unit prices, totals, and currency. Later edits to customer or item master data shall not alter the issued invoice or its PDF. | AC-FIN-003 |
| FR-FIN-004 | The system shall record customer payments with amount, date, method, reference, and allocation to one or more invoices, and shall support **partial** payments with an accurate outstanding balance. | AC-FIN-004 |
| FR-FIN-005 | The system shall compute and display per-invoice outstanding balance and an aged receivables view (current / overdue buckets). Overdue is derived from due date versus current date, not stored as a manually set flag. | AC-FIN-005 |
| FR-FIN-006 | The system shall record operational expenses (non-project and project-attributed) with category, amount, date, and optional attachment. | AC-FIN-006 |
| FR-FIN-007 | The system shall accumulate project cost from recognised sources — goods receipts, supplier invoices, and recorded payments — without double counting the same cost through multiple sources, and shall present cost composition by source. | AC-FIN-007 |
| FR-FIN-008 | The system shall record due dates on invoices and payables and shall surface overdue items with days overdue. | AC-FIN-008 |
| FR-FIN-009 | Payment confirmation shall be a distinct action from payment recording, performed by an authorised role, capturing confirmer, timestamp, and optional evidence reference. | AC-FIN-009 |
| FR-FIN-010 | The system shall provide payables and receivables summaries with outstanding totals per counterparty. | AC-FIN-010 |
| FR-FIN-011 | The system shall support voiding an issued document with a mandatory reason, preserving the original record, its number, and its PDF. A voided number is never reused. | AC-FIN-011 |
| FR-FIN-012 | The system shall support issuing a credit note or replacement document that references the original, with its own allocated number, leaving the original intact and marked as credited or superseded. | AC-FIN-012 |
| FR-DOC-001 | All uploaded files shall be stored in private object storage, never publicly readable, and served only through short-lived signed URLs after a permission check. | AC-DOC-001 |
| FR-DOC-002 | The system shall maintain document metadata (type, owner entity, number where applicable, status, created/issued timestamps, version) with a history of changes. | AC-DOC-002 |
| FR-DOC-003 | The system shall generate PDFs for issued quotations, supplier POs, sales invoices, credit notes, and receipts, with the generated file stored immutably alongside the record. | AC-DOC-003 |
| FR-DOC-004 | The system shall allocate every human-readable document number server-side through the mechanism in [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md). No client, template, or report may generate a document number. | AC-DOC-004 |
| FR-DOC-005 | The system shall retrieve any issued document by its document number and display its status and full history. | AC-DOC-005 |
| FR-DOC-006 | Document types shall be a registry (code, series key, prefix, format, whether approval is required), so adding a type does not require schema change. | AC-DOC-006 |
| FR-DOC-007 | Files may be attached to projects, work orders, daily reports, invoices, payables, and expenses, each attachment linked to exactly one owning record and inheriting its access rules. | AC-DOC-007 |
| FR-DOC-008 | An issued document's stored PDF and snapshot shall be immutable; regeneration is permitted only as a new derived artefact that never replaces the original bytes. | AC-DOC-008 |

### 6.7 Reporting (`FR-RPT`)

| ID | Requirement | AC |
|---|---|---|
| FR-RPT-001 | The system shall produce a daily progress report per project and per technician, generated from submitted daily reports, readable on a phone, and exportable. | AC-RPT-001 |
| FR-RPT-002 | The system shall provide a project dashboard showing, per project: status, material readiness status, schedule variance, cost to date, billed amount, and outstanding balance. | AC-RPT-002 |
| FR-RPT-003 | The system shall provide a material status report per project listing each requirement line's stock status, reserved quantity, purchased quantity, received quantity, issued quantity, consumed, returned, and remaining. | AC-RPT-003 |
| FR-RPT-004 | The system shall provide receivables and payables ageing reports by counterparty and bucket. | AC-RPT-004 |
| FR-RPT-005 | The system shall provide a procurement pipeline report by state: requested, approved, ordered, partially received, received, invoiced, paid. | AC-RPT-005 |
| FR-RPT-006 | The system shall provide a technician workload report: assignments, scheduled days, open work orders, blocked work orders. | AC-RPT-006 |
| FR-RPT-007 | All reports shall support a date range and export to CSV; documents intended for external parties export to PDF. | AC-RPT-007 |
| FR-RPT-008 | Report figures shall be reproducible: re-running a report for a past date range shall yield the same result unless underlying records were corrected, and corrections are visible in the audit trail. | AC-RPT-008 |

### 6.8 Data migration (`FR-DATA`)

| ID | Requirement | AC |
|---|---|---|
| FR-DATA-001 | The system shall import master data (customers, contacts, suppliers, technicians, service categories, item catalogue) from CSV with per-row validation and a rejection report naming the row, column, and reason. | AC-DATA-001 |
| FR-DATA-002 | Import shall run as a two-step process: a validation/dry-run producing a full preview and error list, then an explicit commit. No import writes on validation alone. | AC-DATA-002 |
| FR-DATA-003 | The system shall import open operational items (active projects, outstanding invoices and payables) with their **existing historical document numbers preserved verbatim** where the business requires continuity. | AC-DATA-003 |
| FR-DATA-004 | Imported historical document numbers shall be stored in a form that does not consume or collide with newly allocated numbers in the same series, and the collision check shall run before commit. | AC-DATA-004 |
| FR-DATA-005 | Referenced master data shall be deactivated rather than deleted; deletion of a record referenced by any transaction shall be refused. | AC-DATA-005 |
| FR-DATA-006 | The system shall provide a full export of its own data (structured tables plus files) for portability and backup verification. | AC-DATA-006 |

### 6.9 Audit (`FR-AUD`)

| ID | Requirement | AC |
|---|---|---|
| FR-AUD-001 | The system shall write an audit record for every create, state transition, approval, issuance, cancellation, payment record/confirmation, permission change, and override, capturing actor, action, target entity and ID, before/after values for changed fields, timestamp, and request correlation ID. | AC-AUD-001 |
| FR-AUD-002 | Audit records shall be append-only. No application path may update or delete an audit record. | AC-AUD-002 |
| FR-AUD-003 | Audit writes for a state change shall occur in the same database transaction as the change, so a committed state change without its audit record is impossible. | AC-AUD-003 |
| FR-AUD-004 | Any override of a blocked workflow shall require a reason and shall produce an audit record flagged as an override, visible to the owner. | AC-AUD-004 |
| FR-AUD-005 | The audit log shall be viewable by `OWNER` and by an explicitly granted audit-viewer permission, filterable by actor, entity, action, and date range. | AC-AUD-005 |
| FR-AUD-006 | The audit log shall retain records for the configured retention period and shall survive archival without loss of integrity. | AC-AUD-006 |

---

## 7. Non-functional requirements

| ID | Requirement | Target / verification | AC |
|---|---|---|---|
| NFR-SEC-001 | All access requires authentication; no anonymous read of any business data. | Every route except login/health returns 401 without a session | AC-SEC-001 |
| NFR-SEC-002 | Authorisation is enforced server-side per **action and resource**, not by hiding UI elements. | Direct API calls by an unauthorised role return 403 regardless of UI | AC-SEC-002 |
| NFR-SEC-003 | One account may hold multiple roles; effective permissions are the union, with conflicting-permission controls applied. | Role-combination test matrix | AC-SEC-003 |
| NFR-SEC-004 | Sessions expire after a configurable idle period and on password change. | Session test | AC-SEC-004 |
| NFR-SEC-005 | Passwords are stored only as a slow salted hash; minimum length and breach-list check enforced. | Hash inspection + policy test | AC-SEC-005 |
| NFR-SEC-006 | Technician accounts cannot reach financial, supplier-payment, or company-wide reporting endpoints. | Negative permission tests per release | AC-SEC-006 |
| NFR-SEC-007 | Secrets are supplied via environment/secret store; never committed, never logged, never returned by an API. | Secret scan of repo and logs | AC-SEC-007 |
| NFR-SEC-008 | All traffic is encrypted in transit; production requires TLS. | TLS configuration check | AC-SEC-008 |
| NFR-AUD-001 | Audit log is append-only at the database level, not merely by convention. | Attempt update/delete as app role; must fail | AC-AUD-001 |
| NFR-AUD-002 | Audit coverage includes all state machines in §9 and all §6.9 events. | Coverage test per entity | AC-AUD-002 |
| NFR-AUD-003 | Audit records are tamper-evident (hash chaining or equivalent) for issued-document events. | Chain verification job | AC-AUD-003 |
| NFR-AUD-004 | Overrides are always attributable and surfaced to the owner. | Override report non-empty when overrides exist | AC-AUD-004 |
| NFR-AUD-005 | Audit retention period is configurable and defaults to the documented policy. | Retention job test | AC-AUD-005 |
| NFR-AUD-006 | Audit viewer is restricted to authorised roles and is itself audited. | Access + self-audit test | AC-AUD-006 |
| NFR-DATA-001 | All multi-table state changes occur in a single ACID transaction. | Failure-injection test: no partial writes | AC-DATA-001 |
| NFR-DATA-002 | Document-number uniqueness is enforced by a database constraint, not application logic alone. | Concurrent insert test | AC-DATA-002 |
| NFR-DATA-003 | Referential integrity is enforced by foreign keys; no orphan transactions. | Schema inspection + attempted orphan insert | AC-DATA-003 |
| NFR-DATA-004 | Automated daily backups plus a restore rehearsal at least quarterly. | Restore log with measured duration | AC-DATA-004 |
| NFR-DATA-005 | Recovery targets: **RPO ≤ 24 h**, **RTO ≤ 8 h** for MVP scale. | Restore rehearsal against target | AC-DATA-005 |
| NFR-DATA-006 | Schema migrations are versioned, forward-only, and rehearsed on a copy before production. | Migration log per release | AC-DATA-006 |
| NFR-DATA-007 | The system is safe under concurrent use by ≥10 simultaneous users performing state-changing actions. | Load/concurrency test | AC-DATA-007 |
| NFR-DATA-008 | Data retention: financial and issued documents retained per policy; operational data retained at least as long as its project. | Retention configuration review | AC-DATA-008 |
| NFR-OPS-001 | Separate environments for development, staging, and production, with no production data in lower environments except anonymised copies. | Environment inventory | AC-OPS-001 |
| NFR-OPS-002 | Structured application logs and error tracking sufficient to diagnose a reported failure without reproducing it. | Log inspection exercise | AC-OPS-002 |
| NFR-OPS-003 | Mobile-first responsive web: all technician-facing flows fully usable at 360 px width with touch targets ≥ 44 px. | Device test on 360 px viewport | AC-OPS-003 |
| NFR-OPS-004 | Performance: p95 page/interaction response < 2 s on a 3G-class connection for technician flows; < 1 s on office LAN for list views. | Load test with recorded p95 | AC-OPS-004 |
| NFR-OPS-005 | Availability target 99% monthly during business hours; planned maintenance announced. | Uptime monitoring | AC-OPS-005 |
| NFR-OPS-006 | User-facing messages are in Indonesian, state what went wrong and what to do next; raw stack traces never shown to users. | Message review + error-path test | AC-OPS-006 |
| NFR-OPS-007 | UI primary language is Indonesian (see §1.4). | Localisation review | AC-OPS-007 |
| NFR-OPS-008 | Supported browsers: current Chrome, Safari (iOS), Firefox, Edge; no IE. | Browser matrix test | AC-OPS-008 |

---

## 8. Business and validation rules

Rules are normative. Requirements reference them; workflows implement them.

| ID | Rule |
|---|---|
| BR-PROJ-001 | Quotation, customer PO, material lines, and multiple technicians are **optional per project**. No step may be forced merely because it exists in the system. |
| BR-PROJ-002 | Project status, work-order status, and material readiness status are **independent** state machines. A change in one never implies a change in another; any coupling is an explicit, documented rule (BR-PROJ-003). |
| BR-PROJ-003 | A project may enter `IN_PROGRESS` while material readiness is not `READY_TO_START` only via a recorded override with reason (FR-AUD-004), because site work sometimes starts with partial materials. |
| BR-PROJ-004 | A project may be `CLOSED` only when: all work orders are `VERIFIED` or `CANCELLED`; no material line is in an open issued state; all invoices are `ISSUED`/`PAID`/`VOID`/`CREDITED` (none `DRAFT`); and any override is recorded. |
| BR-PROJ-005 | Cancelling a project never deletes financial or document records. Outstanding payables and invoices remain and must be resolved separately. |
| BR-MATL-001 | An unverified quantity is **never** treated as available. `UNKNOWN` stock status blocks reservation (FR-MATL-005). |
| BR-MATL-002 | Reserved quantity across all projects shall never exceed verified on-hand quantity at the moment of reservation. |
| BR-MATL-003 | Quantities change only through append-only movements. There is no "edit quantity" operation anywhere in the product. |
| BR-MATL-004 | Material issued to a technician remains attributable until it is consumed, returned, or written off. No state may leave quantity unattributed. |
| BR-MATL-005 | A PO does **not** increase available stock. Stock becomes available only on recorded goods receipt. |
| BR-PROC-001 | Approval, PO issuance, goods receipt, supplier invoice, and payment are five distinct states. No state may be skipped in reporting, even where one person performs several actions. |
| BR-PROC-002 | A payment recorded is not a payment confirmed. Confirmation is a separate, attributable action. |
| BR-PROC-003 | Over-receipt beyond ordered quantity requires an explicit approval and is recorded as an exception. |
| BR-PROC-004 | The system never initiates, schedules, or executes a bank transfer. It records that a transfer occurred, per the user's assertion, with evidence where available. |
| BR-FIN-001 | Issued documents are immutable in content and number. Corrections happen via credit note or replacement document (FR-FIN-012). |
| BR-FIN-002 | A document number is allocated only at issuance, never at draft creation, and never reused after void. |
| BR-FIN-003 | A payment allocation may not exceed the invoice outstanding balance; overpayment is recorded as an unapplied credit, not as a negative balance. |
| BR-FIN-004 | No issued invoice may be edited. Corrections follow BR-FIN-001. |
| BR-FIN-005 | Project cost recognition counts each cost source once; a goods receipt, its supplier invoice, and its payment are three views of one cost, not three costs (FR-FIN-007). |
| BR-TECH-001 | A technician may act only on their own assignments and only within permitted actions. |
| BR-TECH-002 | A work order may be verified only by a user holding verification permission and not by the technician who completed it, unless an override with reason is recorded (small-team reality, see SECURITY §6). |
| BR-TECH-003 | A daily report may be corrected only by superseding it with a new version; the original remains retrievable. |
| BR-DOC-001 | Every issued document has exactly one stored, immutable PDF matching its issuance snapshot. |
| BR-DOC-002 | Document numbers are generated exclusively server-side (FR-DOC-004). Frontend validation is never a control. |
| BR-AUD-001 | No state change is valid without an audit record in the same transaction. |
| BR-DATA-001 | Historical document numbers imported from spreadsheets are preserved verbatim and never re-allocated. |
| BR-DATA-002 | Referenced records are deactivated, never deleted. |

---

## 9. Status definitions (canonical)

These enumerations are **normative and reused verbatim** by all other documents. Each is a separate state machine (BR-PROJ-002). Full transition tables live in the workflow documents named below.

### 9.1 Project status

Owner: [`./workflows/PROJECT_LIFECYCLE.md`](./workflows/PROJECT_LIFECYCLE.md)

| Status | Meaning |
|---|---|
| `DRAFT` | Being prepared; not yet committed to the field. |
| `PLANNED` | Scope, customer, and schedule agreed; not yet scheduled to people. |
| `SCHEDULED` | Technicians assigned and dates set; not yet started. |
| `IN_PROGRESS` | Work has begun; daily reports expected. |
| `ON_HOLD` | Temporarily suspended with a recorded reason. |
| `PENDING_COMPLETION_VERIFICATION` | All work orders submitted; awaiting office verification. |
| `COMPLETED` | Work verified complete; billing/closure activities may remain. |
| `CLOSED` | All closure conditions met (BR-PROJ-004); read-only except audit and corrections. |
| `CANCELLED` | Abandoned with recorded reason; records retained. |

### 9.2 Work order status

Owner: [`./workflows/TECHNICIAN_OPERATIONS.md`](./workflows/TECHNICIAN_OPERATIONS.md)

| Status | Meaning |
|---|---|
| `DRAFT` | Created, not yet assigned. |
| `ASSIGNED` | Assigned to a technician, not yet acknowledged. |
| `ACCEPTED` | Technician acknowledged the assignment. |
| `IN_PROGRESS` | Work started; daily reports expected. |
| `BLOCKED` | Cannot proceed; blocker recorded (FR-TECH-006). |
| `COMPLETED_PENDING_VERIFICATION` | Technician submitted completion; awaiting verification. |
| `VERIFIED` | Completion accepted. |
| `CANCELLED` | Withdrawn with reason. |

### 9.3 Material readiness status (project level, derived)

Owner: [`./workflows/PROCUREMENT_AND_MATERIALS.md`](./workflows/PROCUREMENT_AND_MATERIALS.md)

| Status | Meaning |
|---|---|
| `NOT_ASSESSED` | No material requirements recorded yet. |
| `REQUIREMENTS_INCOMPLETE` | Lines exist but some remain `UNKNOWN` stock status. |
| `AWAITING_PROCUREMENT` | Verified shortfall exists; purchasing not yet complete. |
| `READY_TO_START` | All lines covered by verified stock or received goods. |
| `PARTIALLY_ISSUED` | Some materials issued to technicians. |
| `FULLY_ISSUED` | All required materials issued. |
| `CLOSED` | All issued material consumed, returned, or written off. |

This status is **derived**, never hand-set. It is displayed alongside, never instead of, project status.

### 9.4 Material line: stock status and fulfilment status (orthogonal)

| Field | Values |
|---|---|
| `stock_status` | `UNKNOWN`, `VERIFIED` |
| `fulfilment_status` | `DRAFT`, `VERIFIED`, `RESERVED`, `PENDING_PURCHASE`, `REQUESTED`, `ORDERED`, `PARTIALLY_RECEIVED`, `RECEIVED`, `ISSUED`, `CONSUMED`, `RETURNED`, `WRITTEN_OFF`, `CLOSED` |

### 9.5 Purchase request status

| Status | Meaning |
|---|---|
| `DRAFT` | Being prepared. |
| `SUBMITTED` | Awaiting approval decision. |
| `APPROVED` | Approved; may become a PO. |
| `REJECTED` | Rejected with reason; terminal. |
| `CONVERTED_TO_PO` | A PO has been generated. |
| `CANCELLED` | Withdrawn with reason. |

### 9.6 Purchase order status

| Status | Meaning |
|---|---|
| `DRAFT` | Prepared, number not yet allocated. |
| `ISSUED` | Numbered, snapshotted, sent to supplier. |
| `PARTIALLY_RECEIVED` | Some lines received; receipt outstanding. |
| `RECEIVED` | All lines fully received. |
| `CLOSED` | Received and financially settled to the extent required. |
| `CANCELLED` | Voided with reason; number retained. |

### 9.7 Supplier payable status

| Status | Meaning |
|---|---|
| `RECORDED` | Supplier invoice captured; unpaid. |
| `PARTIALLY_PAID` | Some amount paid. |
| `PAID` | Fully paid and confirmed. |
| `DISPUTED` | Under query; blocks closure. |
| `CANCELLED` | Voided with reason. |

### 9.8 Sales invoice status

| Status | Meaning |
|---|---|
| `DRAFT` | Prepared; no number allocated. |
| `ISSUED` | Numbered, snapshotted, PDF stored. |
| `PARTIALLY_PAID` | Some amount received. |
| `PAID` | Fully paid. |
| `OVERDUE` | **Derived display state**: issued/partially paid and past due date. Not a stored transition. |
| `VOID` | Cancelled with reason; number retained, never reused. |
| `CREDITED` | Superseded by a credit note. |

### 9.9 Payment record status

`RECORDED` → `CONFIRMED`; `REVERSED` (with reason, creating a compensating record).

### 9.10 Quotation status

`DRAFT`, `SENT`, `ACCEPTED`, `REJECTED`, `EXPIRED`, `CANCELLED`.

### 9.11 Daily work report status

`DRAFT`, `SUBMITTED`, `REVIEWED`, `REJECTED` (returned for correction; a corrected report is a new version, BR-TECH-003).

### 9.12 Document status (generic)

`DRAFT`, `ISSUED`, `VOID`, `SUPERSEDED`, `CREDITED`.

### 9.13 Import batch status

`DRAFT`, `VALIDATED`, `COMMITTED`, `FAILED`, `ROLLED_BACK`.

---

## 10. Three-layer finance model

Kept strictly separate; conflating them is a common cause of unusable small-business systems.

| Layer | Content | MVP | Rationale |
|---|---|---|---|
| **L1 — Operational finance and cash tracking** | Payments recorded/confirmed, expenses, project cost accumulation, cash movement visibility | **In scope** | This is what the owner actually needs to run the business |
| **L2 — AR/AP** | Invoices, payables, allocations, partial payments, ageing, outstanding balances | **In scope** | Directly resolves billing disputes and payment chasing |
| **L3 — Formal accounting** | Journals, chart of accounts, ledgers, trial balance, tax reporting, statutory statements | **[Out of Scope] for MVP** — **[Open Question] Q-FIN-01** | Requires accounting expertise and statutory rules; the operational layer can export to an accountant |

**[Open Question] Q-FIN-01 — Formal accounting depth.** Does the business (or its accountant/tax advisor) require statutory statements, PPN (VAT) reporting, or PPh withholding produced by this system? **Proposed default:** MVP stops at L1+L2 and provides a clean export (FR-DATA-006) for the existing accountant. **Decision trigger:** Phase 0 confirmation of the accountant's requirements; if statutory reporting is required, a Phase 7+ module is scoped separately. Rationale: building a general ledger without an accountant's input produces statements nobody can file.

---

## 11. Document numbering — summary

The full standard is [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md). PRD-level obligations:

1. Every document type has its **own series** (FR-DOC-006).
2. Numbers are allocated **atomically on the server at issuance**, never at draft creation, never by the client, never by `MAX()+1` (FR-DOC-004, BR-DOC-002).
3. Uniqueness is enforced by a **database constraint**, not by application checks alone (NFR-DATA-002).
4. Issued numbers are **immutable and never reused**, including after void (BR-FIN-002, FR-FIN-011).
5. **Two distinct risks** are analysed: internal duplication (solvable here) and conflict with external/client numbering rules (**not** solvable by an internal database alone). See Q-NUM-01.
6. Historical numbers are migrated verbatim (BR-DATA-001).

**[Open Question] Q-NUM-01 — External numbering constraints.** The confirmed conflict may be internal duplication, or a client-side system rejecting numbers that duplicate another vendor's. **Proposed default:** implement an internal-consistency-guaranteed, configurable format (`{PREFIX}/{TYPE}/{YYYY}/{MM}/{SEQ}`) with a company prefix, and treat external constraints as configurable per client. **Decision trigger:** written confirmation from each affected client on permitted length/charset, whether a prefix is allowed, whether validation occurs against a national registry, whether historical numbers must be preserved, whether gaps are acceptable, and how cancellations must be represented. **The company prefix alone does not resolve external conflicts** — that claim is explicitly rejected in the numbering standard.

---

## 12. Security, auditability, backup and recovery

Summarised here; detail in [`./architecture/SECURITY_AND_ACCESS_CONTROL.md`](./architecture/SECURITY_AND_ACCESS_CONTROL.md) and [`./standards/AUDIT_LOGGING.md`](./standards/AUDIT_LOGGING.md).

- **Least privilege by default.** Technicians have assignment-scoped access only (FR-TECH-009).
- **Action-and-resource permissions**, checked server-side (NFR-SEC-002).
- **Separation of duties proportional to team size.** With ~3 office staff, full separation is impossible; where one person must hold conflicting permissions, an override with mandatory reason and owner visibility is required (FR-AUD-004).
- **Immutability of issued artefacts** (BR-FIN-001, FR-DOC-008).
- **Append-only audit with transactional coupling** (FR-AUD-002, FR-AUD-003).
- **Backup:** automated daily, encrypted, stored separately from the primary database; restore rehearsed quarterly (NFR-DATA-004, NFR-DATA-005).
- **File recovery:** object storage with versioning or replicated backup; deletion of referenced files prohibited (FR-DOC-001, FR-DOC-008).
- **Secrets** never in the repository or logs (NFR-SEC-007).

**[Open Question] Q-OPS-01 — Hosting, backup location, retention, and access policy.** Not confirmed. **Proposed default:** single-region managed PostgreSQL plus S3-compatible private object storage in a region geographically close to Indonesia; daily automated backups with 30-day retention plus monthly archival; retention of financial records for at least 10 years to satisfy typical Indonesian tax audit expectations, subject to accountant confirmation. Rationale: these defaults keep architecture and cost planning moving; they are conservative and revisitable.

---

## 13. Assumptions, risks, dependencies

### 13.1 Assumptions

| ID | Assumption | Impact if false |
|---|---|---|
| A-01 | Users have smartphones with a modern browser and intermittent mobile data at sites. | Mobile-first web fails; native app or offline mode becomes necessary |
| A-02 | Indonesian UI is required (no bilingual need). | i18n framework needed earlier |
| A-03 | Office staff can operate a web application with minimal training. | Adoption risk; requires heavier UX investment |
| A-04 | Historical spreadsheet data is recoverable and at least partially clean. | Migration scope grows; see Q-DATA-01 |
| A-05 | One legal entity, one currency (IDR), one timezone (Asia/Jakarta). | Multi-entity/currency scope added |
| A-06 | Suppliers do not need system access. | Supplier portal scope added |
| A-07 | A single designated person will own data quality during pilot. | Migration and cleanup stall |

**[Open Question] Q-DATA-01 — Spreadsheet structures and historical data quality.** The actual sheet layouts, column consistency, and duplication level are unknown. **Proposed default:** Phase 0 collects anonymised samples of each spreadsheet type, profiles them, and produces a mapping specification before any import is built. **Decision trigger:** completion of Phase 0 profiling. Rationale: building importers against imagined schemas is the most common cause of failed migrations.

### 13.2 Risks

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | Field adoption fails; technicians keep using WhatsApp | High | High | M3 target (<3 min), mobile-first design, pilot with 2–3 technicians before full rollout |
| R-02 | Invoice conflict recurs with a client despite internal fix | Medium | High | Dual-risk analysis; per-client format configuration; Q-NUM-01 confirmation |
| R-03 | Material data quality is poor, so reservations are wrong | High | Medium | Explicit `UNKNOWN` state forces verification; reconciliation checkpoints (FR-MATL-013) |
| R-04 | Scope creep toward full ERP | Medium | High | Explicit exclusions §5.3; AGENTS.md scope boundary |
| R-05 | One person holds conflicting permissions and abuses them | Low | High | Override logging with reason + owner visibility (FR-AUD-004) |
| R-06 | Historical migration corrupts numbering | Medium | High | Verbatim preservation, pre-commit collision check (FR-DATA-004) |
| R-07 | Backup exists but restore was never tested | Medium | High | Quarterly restore rehearsal (NFR-DATA-004) |
| R-08 | Small team cannot sustain documentation upkeep | Medium | Medium | AGENTS.md documentation duties; docs reviewed at each phase exit |

### 13.3 Dependencies

| ID | Dependency | Blocks |
|---|---|---|
| D-01 | Phase 0 process mapping and open-question answers | All build phases |
| D-02 | Document numbering standard implemented and tested | Any invoice/PO/quotation issuance (Phase 2+) |
| D-03 | Audit logging implemented | Approvals, issuance, payments (Phase 1+) |
| D-04 | Client confirmation of numbering constraints (Q-NUM-01) | Final invoice format; not the internal mechanism |
| D-05 | Spreadsheet profiling (Q-DATA-01) | Migration (Phase 6) |
| D-06 | Hosting and backup decisions (Q-OPS-01) | Production deployment |
| D-07 | Accountant input (Q-FIN-01) | Any L3 accounting scope |

---

## 14. Open questions register

| ID | Question | Proposed default | Owner | Trigger |
|---|---|---|---|---|
| Q-NUM-01 | Final invoice-numbering rules and client validation requirements | Internal-consistent configurable format with company prefix; external constraints per client | Owner + affected clients | Before Phase 5 invoice go-live |
| Q-PROC-01 | Purchasing approval thresholds and mandatory-approval conditions | Approval required for all purchases above a configurable amount; fail-closed if unconfigured | Owner | Phase 3 |
| Q-FIN-01 | Required financial reports and tax responsibilities | L1+L2 only; export for accountant | Owner + accountant | Phase 0 |
| Q-DATA-01 | Existing spreadsheet structures and historical data quality | Phase 0 profiling produces mapping spec | Ops admin | Phase 0 exit |
| Q-PROC-02 | Supplier delivery and goods-receipt practices (delivery notes? partial deliveries common?) | Line-level partial receipt with discrepancy recording | Ops admin | Phase 3 |
| Q-ROLE-01 | Actual technician and team-leader reporting responsibilities | Team leader as per-project designation, optional review step | Owner | Phase 0 |
| Q-INV-01 | Required stock valuation and inventory-accounting depth | Project-scoped quantities only; no valuation in MVP | Owner | Phase 6+ |
| Q-OPS-01 | Hosting, backup, retention, and access policies | Managed PG + private object storage, daily backup, 10-year financial retention | Owner | Phase 1 |
| Q-BASE-01 | Baseline operational metrics | Phase 0 measurement | Ops admin | Phase 0 |
| Q-UX-01 | Whether a native app is required | Mobile-first responsive web; native out of scope | Owner | After pilot feedback |

---

## 15. Acceptance criteria

All acceptance criteria are held in [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md), grouped by workflow, each referencing the FR/NFR IDs above and stating its test level. Requirement tables in §6 and §7 name their `AC-` references directly.

Test approach, levels, and coverage obligations are in [`./implementation/TESTING_STRATEGY.md`](./implementation/TESTING_STRATEGY.md).

---

## 16. Document map

| Document | Owns |
|---|---|
| [`./architecture/ARCHITECTURE.md`](./architecture/ARCHITECTURE.md) | System structure, module boundaries, technology recommendation |
| [`./architecture/DATA_MODEL.md`](./architecture/DATA_MODEL.md) | Entities, relationships, integrity, snapshots, movement ledger |
| [`./architecture/SECURITY_AND_ACCESS_CONTROL.md`](./architecture/SECURITY_AND_ACCESS_CONTROL.md) | Roles, permission matrix, separation of duties, overrides |
| [`./workflows/PROJECT_LIFECYCLE.md`](./workflows/PROJECT_LIFECYCLE.md) | Project and work-order state machines, closure |
| [`./workflows/PROCUREMENT_AND_MATERIALS.md`](./workflows/PROCUREMENT_AND_MATERIALS.md) | Requirement → reservation → purchase → receipt → issue → consumption |
| [`./workflows/TECHNICIAN_OPERATIONS.md`](./workflows/TECHNICIAN_OPERATIONS.md) | Scheduling, daily reporting, evidence, verification |
| [`./workflows/FINANCE_AND_DOCUMENTS.md`](./workflows/FINANCE_AND_DOCUMENTS.md) | Quotation → invoice → payment, payables, documents |
| [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md) | Numbering format, atomic allocation, dual-risk analysis |
| [`./standards/AUDIT_LOGGING.md`](./standards/AUDIT_LOGGING.md) | Audit event model, immutability, retention |
| [`./implementation/IMPLEMENTATION_PLAN.md`](./implementation/IMPLEMENTATION_PLAN.md) | Phased roadmap with exit criteria |
| [`./implementation/TESTING_STRATEGY.md`](./implementation/TESTING_STRATEGY.md) | Test levels, critical scenarios |
| [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md) | `AC-` register |
| [`./decisions/`](./decisions/) | Architecture decision records |
