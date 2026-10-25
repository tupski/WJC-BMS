# Product Requirements Document — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Document status | Draft for business sign-off |
| Version | 0.2 — baseline reconciled 2026-10-25 |
| Owner | Product / Business Analysis |
| Authoritative scope source | **This document** (see [`../AGENTS.md`](../AGENTS.md)) |
| Last updated | 2026-10-25 |
| Classification key | **[Confirmed]** / **[Proposed]** / **[Assumption]** / **[Open Question]** / **[Out of Scope]** |

This PRD is the single authoritative source of truth for scope and functional requirements. Other documents cross-reference requirement IDs defined here; they do not redefine them.

**Version 0.2 reconciles a superseded baseline.** The technology stack, role model, purchasing approval rule, numbering configuration, and storage model were revised by client instruction. Requirements were updated **in place**; no ID was reused with a changed meaning. Superseded text is retained only in ADR history.

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

Areas: `CUST`, `PROJ`, `PROC`, `MATL`, `TECH`, `FIN`, `DOC`, `RPT`, `DATA`, `SEC`, `AUD`, `OPS`, `PLAT`, `SYS`, `WA`, `MAP`, `PWA`.

IDs are **stable and never reused**. A withdrawn requirement is marked `WITHDRAWN` in place, keeping its number.

### 1.2 Classification labels

- **[Confirmed]** — stated in the client brief; treat as fact.
- **[Proposed]** — our recommendation, awaiting sign-off.
- **[Assumption]** — believed true, not verified; must be validated or it may invalidate dependent requirements.
- **[Open Question]** — unresolved; a decision owner and trigger are named.
- **[Out of Scope]** — explicitly excluded, with rationale.

An assumption is **never** silently promoted to a requirement.

### 1.3 Repository basis

**[Confirmed] 2026-10-25 (measured):** `git rev-parse --is-inside-work-tree` → `true`; branch `main`; working tree clean; `origin` → `https://github.com/tupski/WJC-BMS.git`.

The client states the project is hosted at GitHub `tupski/WJC-BMS` — **[Confirmed] as a client statement**, corroborated by the configured remote. **No network access was available**, so remote branch state, commit SHAs, and hosted artefacts remain **unverifiable offline** and are not asserted anywhere in this set.

Historical note, retained for traceability only: the set was first authored against a workspace verified empty (no `.git`, no `docs/`). True then; **not** current status.

Current status: **no application code, no installed dependencies, no build or test commands, no configuration files.** Every technical statement describes a **target** baseline, never a running system.

### 1.4 Language and locale

**[Confirmed]** Bahasa Indonesia (`id`) is the **default and primary** UI language — navigation, labels, validation messages, status names shown to users, and generated PDFs. Internal identifiers, database columns, and this documentation remain English.

**[Confirmed]** Business timezone **`Asia/Jakarta`**. Timestamps are stored UTC and rendered in `Asia/Jakarta` (NFR-OPS-009).

A full i18n framework remains **[Out of Scope]**: the UI is Indonesian-only, with strings centralised in language files so translation is possible later without a rewrite.

---

## 2. Product overview

**Working name:** WJ-CCTV Business Operations System
**Category:** Business Management System (BMS) — project management, lightweight field service management, procurement, project-based material control, document management, operational finance.

### 2.1 Problem statement

WJ-CCTV.com delivers CCTV-related and other business services. Operations — office administration, purchasing, suppliers, project execution, field technicians, invoicing, financial administration — run on disconnected tools: Excel/Google Sheets, Word, and WhatsApp.

**[Confirmed]** operating conditions:

- ~3 office employees, ~15 technicians.
- Administration relies on spreadsheets, documents, and WhatsApp groups.
- No centralised inventory; materials are prepared and tracked per project.
- Not all required materials are on hand; some must be purchased before work proceeds.
- Office administration handles purchasing and PO preparation.
- The owner approves purchases — **every** purchase, not only some.
- The owner may personally transfer funds to suppliers.
- The owner assigns technicians per project — sometimes one, sometimes a team.
- Projects span multiple days; a technician's work spans several discrete sessions.
- Technicians report progress through WhatsApp groups.
- An invoice-numbering conflict has occurred. Its exact cause is **unconfirmed**.

### 2.2 Consequences of the current state

| # | Consequence | Evidence | Addressed by |
|---|---|---|---|
| P1 | Invoice numbers can collide or be rejected by a client, blocking billing and payment | **[Confirmed]** a conflict occurred; cause **[Open Question]** | §6.6, [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md) |
| P2 | Material availability is unknown until someone physically checks | **[Confirmed]** | FR-MATL-002, FR-MATL-005 |
| P3 | The same material can be implicitly promised to two projects | **[Assumption]** — no reservation concept exists | FR-MATL-004, FR-MATL-006 |
| P4 | Approval, PO issuance, delivery, and payment are indistinguishable in chat history | **[Confirmed]** they are separate real-world steps | FR-PROC-002, FR-PROC-011 |
| P5 | Multi-day progress lives in chat scrollback; no reliable state or history | **[Confirmed]** | FR-TECH-004, FR-TECH-011 |
| P6 | Cost and margin per project cannot be reconstructed afterwards | **[Assumption]** | FR-FIN-007, FR-RPT-002 |
| P7 | No audit trail: who changed a number, approved a purchase, issued an invoice | **[Assumption]** | FR-AUD-001 |
| P8 | Knowledge is held by individuals, not the system | **[Assumption]** | FR-DATA-001, FR-RPT-001 |

**[Open Question] Q-BASE-01 — Baseline metrics.** Current volumes (projects/month, invoices/month, project duration, conflict frequency) are unknown. Proposed: Phase 0 extracts 3 months of spreadsheets and counts distinct projects, invoices issued, and documented conflicts. No figures are invented here.

### 2.3 Product vision

A single centralised, auditable system of record for project delivery and its money: what was promised, what was bought, what was used, who did the work, when they actually worked, what was billed, and what was paid — each step evidenced, timestamped, attributable, and impossible to silently overwrite.

```mermaid
flowchart LR
    A["Customer and project"] --> B["Material readiness"]
    B --> C["Purchase request"]
    C --> D["Pemilik approval"]
    D --> E["PO issued"]
    E --> F["Goods received"]
    F --> G["Material issued"]
    G --> H["Work sessions and daily reports"]
    H --> I["Completion submitted"]
    I --> J["Admin verification"]
    J --> K["Invoice issued"]
    K --> L["Payment recorded and confirmed"]
    L --> M["Project closed"]
```

### 2.4 Mandatory technology baseline

**[Confirmed]** by the client. These are requirements, not proposals. Mechanism and rationale: [`./decisions/ADR-008-laravel-platform-baseline.md`](./decisions/ADR-008-laravel-platform-baseline.md).

| # | Baseline | Requirement |
|---|---|---|
| 1 | **Laravel 13.x** — latest stable 13.x patch at implementation time | NFR-PLAT-001. **Never** downgrade to Laravel 12 or earlier to accommodate a template or package |
| 2 | **Laravel Blade**, server-rendered | NFR-PLAT-002 |
| 3 | **Hotwire Turbo** for navigation and progressive page updates | NFR-PLAT-002, NFR-OPS-011 |
| 4 | **TailAdmin Free Laravel** as the UI foundation | NFR-PLAT-003, subject to a Phase 1 compatibility and licence gate |
| 5 | **MySQL or MariaDB**, supported stable version chosen at implementation planning | NFR-PLAT-004, NFR-DATA-009 |
| 6 | **Bahasa Indonesia (`id`)** default and primary UI language | NFR-OPS-007 |
| 7 | Business timezone **`Asia/Jakarta`**; store UTC, render local | NFR-OPS-009 |
| 8 | Responsive **mobile-first** web app with **PWA** support | NFR-OPS-003, NFR-OPS-010 |
| 9 | **Modular monolith** — one deployable, enforced module boundaries | NFR-PLAT-007 |
| 10 | **Laravel ≥ 13.29.0** minimum version, stated as a security requirement | NFR-PLAT-008 |

**Security-version honesty statement.** The client mandates **≥ 13.29.0** citing CVE-2026-114509. The *minimum version* is **[Confirmed]** as a client requirement. The *CVE identity, its nature, its affected component, and whether 13.29.0 is genuinely the fixed release* are **[Assumption] / [Open Question]** — **not verified in this environment, which has no network access**. No advisory text, CVSS score, or advisory URL is asserted. Verification points are named in §7 (NFR-PLAT-008); a release gate is mandatory (AC-OPS-014).

**TailAdmin honesty statement.** The current TailAdmin Free Laravel release, licence terms, dependency set, and declared `laravel/framework` constraint are **[Open Question] Q-UI-01** — not auditable offline. Use is conditional on a Phase 1 gate (AC-OPS-015). If no Laravel 13-compatible release exists, the minimum adaptation path is to treat TailAdmin as a **view layer only** (Blade components plus Tailwind assets) vendored into the app, replacing its service providers, routes, and auth scaffolding, and pinning `laravel/framework` per this baseline. Silently downgrading Laravel, substituting Filament, or using a paid TailAdmin edition is **forbidden**.

---

## 3. Goals and measurable outcomes

### 3.1 Goals

| ID | Goal | Rationale |
|---|---|---|
| G1 | Eliminate internal duplicate document numbers | Highest-severity confirmed defect |
| G2 | Make material availability per project explicit before work starts | Prevents site aborts and rework |
| G3 | Make procurement states unambiguous, with owner approval on every purchase | Stops "approved" being mistaken for "paid", and unapproved spending |
| G4 | Replace WhatsApp progress reporting with structured, phone-friendly reporting | Removes scrollback dependency |
| G5 | Produce a per-project cost and billing trail | Enables margin visibility |
| G6 | Make every state change attributable and reversible only by documented procedure | Auditability and dispute defence |
| G7 | Keep operating overhead low enough for ~3 office staff | Adoption risk is the dominant failure mode |
| G8 | Record **when work actually happened**, not when someone typed a report | Schedule variance and disputes need real timestamps |

### 3.2 Measurable outcomes

| ID | Outcome | Measurement method | Status |
|---|---|---|---|
| M1 | **0** duplicate document numbers across concurrent issuance tests | N parallel issuance requests on one series; assert N distinct numbers, 0 constraint violations | **[Proposed]** |
| M2 | **0** document numbers producible by client-side code | Code review + architecture test that issuance is the only writer of `document_number` | **[Proposed]** |
| M3 | Daily progress report produced in **< 3 minutes** on a phone, project open to submit | Timed usability test, 3 technicians, median of 5 attempts | **[Proposed]** |
| M4 | **100%** of issued invoices retrievable with their issuance snapshot and byte-identical PDF after any later master-data or company-profile edit | Issue → edit customer name, item price, company address → re-download → diff against stored hash | **[Proposed]** |
| M5 | **100%** of material reservations traceable to a verification event or an approved purchase | Query audit: reservations lacking a `stock_verification` or approved request link; must be 0 | **[Proposed]** |
| M6 | **0** technician accounts able to read company-wide financial data | Permission matrix per release; 403 on every finance endpoint for `TEKNISI` | **[Proposed]** |
| M7 | Every approval, issuance, cancellation, and payment confirmation in the audit log within the same transaction | Force failure after state write; assert no orphan; assert a record for every transition | **[Proposed]** |
| M8 | Backup restore verified quarterly with a documented recovery time | Restore rehearsal into a clean environment; record actual RTO | **[Proposed]** |
| M9 | **0** projects in `IN_PROGRESS` with an unverified material line marked as available | Daily reconciliation report; must return 0 rows | **[Proposed]** |
| M10 | **100%** of work sessions carry server-generated start and completion timestamps no technician can alter | Attempt backdating and overwrite via every path; all refused | **[Proposed]** |
| M11 | **0** purchase orders issuable without a recorded `PEMILIK` approval | Direct HTTP test against an unapproved request; refused with no side effects | **[Proposed]** |
| M12 | **0** authenticated business responses cached by the service worker | Cache inspection after navigation and logout; only versioned static assets present | **[Proposed]** |

**Explicitly not targeted in MVP:** reducing headcount, replacing formal accounting software, real-time GPS tracking, bank integration, offline submission and synchronisation.

---

## 4. Users, roles, and representative workflows

### 4.1 Roles

Exactly **four** application roles. Roles are **permission bundles**, not job titles. The authoritative matrix lives in [`./architecture/SECURITY_AND_ACCESS_CONTROL.md`](./architecture/SECURITY_AND_ACCESS_CONTROL.md); this section defines *who exists*, not *what they may do*.

| Role | Who | Scope | Notes |
|---|---|---|---|
| `SUPER_ADMIN` | System administrator | Full application access **plus** system administration: users and role assignment, system settings, document-numbering configuration, SMTP settings, company profile and bank accounts | **[Confirmed]** All sensitive and destructive actions audited. **Still bound by business-integrity invariants** — cannot edit issued documents, reuse numbers, delete audit rows, or create an approved PO without Pemilik approval |
| `PEMILIK` | Business owner | Dashboards, projects, documents, financial reports, operational progress and exceptions | **[Confirmed]** **Mandatory approver for every purchase request.** Cannot perform designated system-destructive actions |
| `ADMIN` | Operations administrator (office) | Full operational access: projects, customers, suppliers, work orders, project materials, purchasing **preparation**, documents, permitted financial administration | **[Confirmed]** **Cannot approve purchases** and **cannot bypass the approval workflow** |
| `TEKNISI` | Field technician (~15) | Structurally scoped to assigned projects and work orders only | **[Confirmed]** Start work sessions, update permitted statuses, submit reports and uploads, request additional materials, view own request status, prepare WhatsApp report and send manually |

### 4.2 Role reconciliation

The previous five-role model is superseded. No document retains `FINANCE` or `TEAM_LEADER` as a role.

| Legacy role | New position |
|---|---|
| `OWNER` | → `PEMILIK` |
| `OPS_ADMIN` | → `ADMIN` |
| `FINANCE` | **Removed as a role.** Its actions become granular permissions grantable to `ADMIN` and/or `SUPER_ADMIN` |
| `TECHNICIAN` | → `TEKNISI` |
| `TEAM_LEADER` | **Removed as a role.** Becomes an optional per-project designation with an explicitly enumerated, narrowly scoped permission set |
| — | `SUPER_ADMIN` (new) |

**[Confirmed] Team-leader designation.** Modelled as `project_member.is_team_leader` plus a scoped policy — **not** a role. Enumerated extra capability, on that project only: submit a team-level daily summary, and see teammates' reports for that project. No financial access, no material-issue authority, no company-wide read.

**[Open Question] Q-ROLE-01 — Is the designation useful at all?** **Proposed default:** keep it, because multi-day multi-person projects need one accountable person on site and the review step is cheap now and expensive to retrofit. **Decision trigger:** Phase 0 process mapping — if the owner states all technicians report directly to the office, retain the designation for accountability and drop the review step. If it later proves a distinct function, that may justify a future fifth role; that is **not** decided here.

**[Assumption] A-ROLE-02 — Non-uniform technician access.** Technician visibility is **assignment-scoped by default** (FR-TECH-009), with no company-wide read. Broader access for an individual is achieved by granting an additional role, not by widening `TEKNISI`.

**[Assumption] A-ROLE-03 — Combined office roles.** At least one person will hold `SUPER_ADMIN` and `ADMIN` simultaneously. The system must support this without pretending a separation of duties exists that does not. Where one person holds conflicting permissions, the logged-override control applies (SECURITY §6).

**[Confirmed] Financial separation of duties inside four roles.** Financial authority is expressed through **action permissions + approval requirements + audit**, never through role names. Broad operational access (`ADMIN`) does **not** imply financial authority, and `ADMIN` may never approve a purchase.

### 4.3 Representative workflows

Narratives only; authoritative flows are in [`./workflows/`](./workflows/).

**W1 — Multi-day installation with procurement.** `ADMIN` creates the project → requirements listed, some `UNKNOWN` → verification → shortfall becomes a purchase request → `PEMILIK` approves → PO issued → goods arrive, possibly twice → materials issued → the technician starts a **work session** each day, submits a report with photos, ends the session → final day: submits completion → `ADMIN` verifies → invoice from an immutable snapshot → two instalments → closed.

**W2 — Small single-technician job, no purchasing.** Project with no purchased materials → one technician → one session, report, completion → invoice → payment → closed. **No quotation, customer PO, purchase request, or material lines are required** (BR-PROJ-001).

**W3 — Owner-funded direct purchase.** Request → `PEMILIK` approves → owner transfers funds personally and records it → `ADMIN` confirms → delivery → goods receipt → supplier invoice as payable → payment confirmation. The system records the transfer; it **does not** move money (FR-PROC-010).

**W4 — Correction after issuance.** An issued invoice has a wrong quantity → `ADMIN` does **not** edit it → credit note plus replacement invoice, each with its own number and audit trail → original marked `CREDITED` and still retrievable (BR-FIN-004, FR-FIN-011/012).

**W5 — Two technicians, three days, six sessions.** Each starts and ends a session per day. Sessions are independent records with server timestamps; a forgotten end is corrected by `ADMIN` through an audited correction with a reason, retaining the originals.

**W6 — WhatsApp report prepared, sent manually.** A `TEKNISI` opens the project, taps *Siapkan laporan WhatsApp*, reviews the text, and sends it to whichever chat they choose. The system records "report prepared", never "message sent" (FR-WA-001…003).

---

## 5. Scope

### 5.1 MVP scope

One coherent loop: customer → project → materials/procurement → field execution with work sessions → invoice → payment → closure, with audit and document control underneath, plus the administrative capabilities the four-role model requires.

| Capability | Included | Requirement block |
|---|---|---|
| Authentication, four roles, multi-role accounts | Yes | NFR-SEC-001…009 |
| Customer and contact records | Yes | FR-CUST-001…006 |
| Projects with categories, schedule, assignment, optional map location | Yes | FR-PROJ-001…013 |
| Quotations and customer POs | Yes, **optional per project** | FR-FIN-001, FR-CUST-004 |
| Material requirements + reservation | Yes | FR-MATL-001…014 |
| Purchase request → **Pemilik approval** → PO → goods receipt | Yes | FR-PROC-001…012 |
| Work orders, **work sessions**, daily reports, photos | Yes | FR-TECH-001…014 |
| Sales invoices, payments, AR; supplier payables; expenses; project cost | Yes (operational, not ledger) | FR-FIN-002…010 |
| Private local document storage + PDF generation | Yes | FR-DOC-001…009 |
| Audit logging | Yes — **foundation phase** | FR-AUD-001…006 |
| Document numbering (configurable template) | Yes — **foundation phase** | FR-DOC-004/006, [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md) |
| Spreadsheet migration of master data + open items | Yes | FR-DATA-001…006 |
| Management reports | Yes, limited set | FR-RPT-001…008 |
| System settings: SMTP, company profile, bank accounts | Yes | FR-SYS-001…004 |
| WhatsApp preparation, map display, PWA | Yes | FR-WA-001…003, FR-MAP-001…003, FR-PWA-001…004 |
| Mobile-first responsive web | Yes | NFR-OPS-003 |

### 5.2 Later phases

| Capability | Trigger |
|---|---|
| Centralised stock with warehouse locations | Concurrent projects repeatedly contend for the same stock, or stock value exceeds an owner-set threshold (**Q-INV-01**) |
| Project profitability analytics | Cost data captured reliably for ≥2 months |
| Formal accounting | Accountant or tax authority requires statements the operational layer cannot produce (**Q-FIN-01**) |
| Offline draft submission and synchronisation | Measured need after PWA launch (**Q-PWA-01**) |
| Email notification events | Business approval of proposed events (**Q-MAIL-02**) |
| Native mobile application | Measured web friction, not preference |
| Supplier/customer portals | Volume justifies it |
| Integrations (accounting, payment gateways, WhatsApp Business API, Google Maps API) | Named business need |
| Multi-branch / multi-company | Second legal entity or location exists |

### 5.3 Explicit exclusions

| Excluded | Rationale | Label |
|---|---|---|
| Full ERP (manufacturing, HR/payroll, CRM campaigns, general ledger) | Not justified by ~3 office staff and 15 technicians | **[Out of Scope]** |
| Bank integration, payment initiation, automatic transfer | The app records and confirms transfers; it never moves money | **[Out of Scope]** |
| Full warehouse management (bins, cycle counting, lot/serial, barcode) | Project-scoped control suffices; see Q-INV-01 | **[Out of Scope]** |
| Microservices, event streaming, message brokers | No scale or team-size justification | **[Out of Scope]** |
| Native iOS/Android app | Mobile-first PWA covers the need | **[Out of Scope]** |
| Real-time GPS technician tracking | Not requested; privacy cost | **[Out of Scope]** |
| Customer-facing portal / login | Not requested | **[Out of Scope]** |
| Payroll, attendance, leave management | Distinct domain; work sessions are not attendance tracking | **[Out of Scope]** |
| Multi-currency, multi-language UI | Single currency, Indonesian UI | **[Out of Scope]** |
| Statutory tax filing / e-invoicing integration | Depends on Q-FIN-01 and Q-NUM-01 | **[Out of Scope]** |
| WhatsApp Business API, automated group posting, delivery receipts | Client confirms manual send only; group IDs are not reliably available without the Business API | **[Out of Scope]** |
| In-app routing engine | OSM tiles carry no routing engine; directions are delegated to Google Maps URLs | **[Out of Scope]** |
| Google Maps API integration beyond deep links | Deep links need no key; an API integration needs separate approval | **[Out of Scope]** |
| Full offline operation, offline submission, sync queues | MVP is online-first; only local draft preservation is in scope | **[Out of Scope]** |

---

## 6. Functional requirements

Each requirement states expected behaviour and points to its acceptance criteria. Criteria live **only** in [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md).

### 6.1 Customer management (`FR-CUST`)

| ID | Requirement | AC |
|---|---|---|
| FR-CUST-001 | Maintain a customer master record with at least: display name, customer type (individual / company), primary phone, address, active/inactive status. Records are **deactivated, never hard-deleted**, once referenced by a project or document. | AC-CUST-001 |
| FR-CUST-002 | A customer may have multiple contact persons, exactly one flagged primary. A contact holds name, role/title, phone, optional email. | AC-CUST-002 |
| FR-CUST-003 | Each customer shall carry one or more **service categories** describing the work they buy. Categories are configurable master data, not a fixed enumeration in code. | AC-CUST-003 |
| FR-CUST-004 | Optionally record a customer purchase-order reference (number, date, uploaded copy) against a project, where the customer issues one. Optional; a project without one is valid. | AC-CUST-004 |
| FR-CUST-005 | A customer record shall present a consolidated view of that customer's projects, issued invoices, outstanding balances, and document history. | AC-CUST-005 |
| FR-CUST-006 | On creating a customer, warn when a probable duplicate exists (normalised name similarity or identical primary phone) and require explicit confirmation. Shall not silently block creation. | AC-CUST-006 |

### 6.2 Project management (`FR-PROJ`)

| ID | Requirement | AC |
|---|---|---|
| FR-PROJ-001 | Create a project with: customer, service category, title, site location, planned start, target completion, and assigned owner-of-record among office staff. | AC-PROJ-001 |
| FR-PROJ-002 | Service category is first-class and drives no hard-coded CCTV assumptions. No workflow step, field, or report may require CCTV-specific data as a completion condition. Category-specific fields are optional attributes. | AC-PROJ-002 |
| FR-PROJ-003 | A project may optionally reference a quotation. Absence is valid and shall not block any later step. | AC-PROJ-003 |
| FR-PROJ-004 | A project may optionally reference a customer PO (FR-CUST-004). Absence shall not block any later step. | AC-PROJ-004 |
| FR-PROJ-005 | Record planned start, target completion, and (where known) a per-day or per-phase schedule. Target completion may be revised; each revision is recorded with reason, actor, timestamp. | AC-PROJ-005 |
| FR-PROJ-006 | Support projects spanning an arbitrary number of days. Progress is expressed as accumulated daily reports and work-order completion, never a single binary flag. | AC-PROJ-006 |
| FR-PROJ-007 | Assign one or more technicians, and designate **at most one** as team leader for that project. Assignment, unassignment, and designation are recorded with actor and timestamp. | AC-PROJ-007 |
| FR-PROJ-008 | Record a project budget (estimated revenue and material cost) where one exists, and display actual accumulated cost against it. Optional. | AC-PROJ-008 |
| FR-PROJ-009 | Derive project material requirements from scope and expose them for verification, reservation, and purchasing (FR-MATL-001). | AC-PROJ-009 |
| FR-PROJ-010 | A project may move to `CLOSED` only when all BR-PROJ-004 conditions are satisfied. Refuse closure and enumerate unmet conditions. | AC-PROJ-010 |
| FR-PROJ-011 | Enforce the project status state machine in §9.1. Reject out-of-set transitions, naming the current status and permitted next statuses. | AC-PROJ-011 |
| FR-PROJ-012 | Support `ON_HOLD` and `CANCELLED`, each requiring a reason code and free-text reason. Cancellation does not delete the project, its documents, or its financial records. | AC-PROJ-012 |
| FR-PROJ-013 | Store a human-readable site address for every project and **optional** coordinates. A project without coordinates remains fully functional: it shows the address and a graceful map placeholder, not an error. | AC-MAP-001 |

### 6.3 Procurement (`FR-PROC`)

| ID | Requirement | AC |
|---|---|---|
| FR-PROC-001 | Create a purchase request with one or more lines, each linked to a material requirement line and a supplier (or a supplier to be determined). Aggregates shortfalls and may serve more than one project line. | AC-PROC-001 |
| FR-PROC-002 | Record an approval decision on a submitted request as exactly `APPROVED` or `REJECTED`, capturing the deciding user, timestamp, and reason. A rejected request cannot proceed to PO. Approval is a decision — not a purchase, delivery, or payment. | AC-PROC-002 |
| FR-PROC-003 | **WITHDRAWN 2026-10-25.** Previously: approval driven by a configurable policy (amount thresholds, supplier category, project flag). Superseded by FR-PROC-011 — no threshold, no auto-approval, no policy engine. ID retained, not reused. | — (see FR-PROC-011) |
| FR-PROC-004 | Maintain a supplier master (name, contact, payment terms, active status) with soft deactivation, and support multiple suppliers per material category. | AC-PROC-004 |
| FR-PROC-005 | Generate a supplier purchase order from an **approved** request only. PO lines inherit quantities and prices, adjustable before issuance with the change recorded. | AC-PROC-005 |
| FR-PROC-006 | Issuing a PO shall allocate a document number per [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md), freeze a snapshot of supplier, company, and line data, and produce a PDF. Irreversible except by cancellation (FR-FIN-011). | AC-PROC-006 |
| FR-PROC-007 | Record goods receipt against a PO at line level, supporting **partial** receipt across events. Received quantity is cumulative and may never exceed ordered quantity without an explicit recorded over-receipt approval. | AC-PROC-007 |
| FR-PROC-008 | Capture receipt discrepancies (short delivery, damaged, wrong item, price variance) as recorded exceptions requiring resolution before the line is considered fully received. | AC-PROC-008 |
| FR-PROC-009 | Record a supplier invoice as a payable, linked to goods receipts or PO lines, with amount, due date, and optional attachment. May exist before or after payment. | AC-PROC-009 |
| FR-PROC-010 | Record a supplier payment with amount, date, method, reference, and the payables it settles, supporting **partial** payment. Recording does **not** move money; the system has no banking integration and must never imply it executed a transfer. A separate confirmation step marks it confirmed. | AC-PROC-010 |
| FR-PROC-011 | **Every purchase request requires explicit approval by a `PEMILIK` before a PO may be issued.** No amount threshold, no auto-approval, no delegation, no below-threshold exception. The request enters `PENDING_APPROVAL` on submission; only `APPROVED` requests proceed. Enforced **server-side and transactionally** (state guard plus database constraint), not by hiding a button. | AC-PROC-011 |
| FR-PROC-012 | `ADMIN` may prepare **draft** purchasing documents before approval, but the system shall refuse to issue or send an authorised supplier PO, and shall refuse any action committing the company to a supplier, while the request is not `APPROVED`. A direct request to issue a PO against an unapproved request is rejected with **no side effects**. | AC-PROC-012 |

### 6.4 Project materials (`FR-MATL`)

| ID | Requirement | AC |
|---|---|---|
| FR-MATL-001 | Record material requirement lines per project: item, required quantity, unit, optional specification note. Independent of whether the item is in stock. | AC-MATL-001 |
| FR-MATL-002 | Every line carries an explicit **stock status** of `UNKNOWN` or `VERIFIED`. New lines default to `UNKNOWN`. The system shall never treat an `UNKNOWN` quantity as available. | AC-MATL-002 |
| FR-MATL-003 | A verification action records the observed on-hand quantity, verifying user, timestamp, and optional note. Verification sets the line to `VERIFIED` with that quantity. | AC-MATL-003 |
| FR-MATL-004 | Support a reservation lifecycle on verified quantity: create, release, consume. Each transition is an append-only movement record. | AC-MATL-004 |
| FR-MATL-005 | Refuse to create a reservation that would be treated as "covered" while stock status is `UNKNOWN`. The refusal names the blocking condition and required action. | AC-MATL-005 |
| FR-MATL-006 | Evaluate reservations against total on-hand net of reservations held by other projects. Never allow reservations across projects to exceed verified on-hand quantity. | AC-MATL-006 |
| FR-MATL-007 | Compute quantity-to-purchase per line as required minus verified available net of competing reservations, and expose that shortfall to procurement (FR-PROC-001). | AC-MATL-007 |
| FR-MATL-008 | Record issue of materials to a named technician for a named project, reducing available quantity and creating an issue record. | AC-MATL-008 |
| FR-MATL-009 | Record consumption of issued materials against a project or work order. | AC-MATL-009 |
| FR-MATL-010 | Record return of unused issued materials to stock or the office, restoring availability and closing the issue record. | AC-MATL-010 |
| FR-MATL-011 | Record damage, loss, and write-off with a reason code, keeping the quantity traceable rather than silently vanishing. | AC-MATL-011 |
| FR-MATL-012 | Record all quantity changes as append-only movement rows. Provide no user-facing or API path that directly edits a stored quantity. A correction is a compensating movement, not an overwrite. | AC-MATL-012 |
| FR-MATL-013 | Provide a reconciliation checkpoint per project (later per location) that recomputes expected quantity from the ledger and reports variance against recorded balances. | AC-MATL-013 |
| FR-MATL-014 | A `TEKNISI` assigned to a project may submit an **additional material request** for that project only. It enters the same `PENDING_APPROVAL` flow (FR-PROC-011) and the technician can see its status. A technician cannot approve it, issue a PO, or see other projects' requests. | AC-MATL-014 |

### 6.5 Technician operations (`FR-TECH`)

| ID | Requirement | AC |
|---|---|---|
| FR-TECH-001 | Maintain a technician record (name, phone, active status, skills where known) linked to exactly one user account. | AC-TECH-001 |
| FR-TECH-002 | May designate one technician per project as **team leader for that project**, granting only: submission of a team-level daily summary, and visibility of that project's team reports. No financial, material-issue, or company-wide access. | AC-TECH-002 |
| FR-TECH-003 | Create an individual work order per technician per project, holding planned dates, scope description, and its own status. | AC-TECH-003 |
| FR-TECH-004 | A technician shall submit a daily work report from a mobile browser containing: date, project, work performed (free text), progress indicator, optional material usage, optional photos. Submission in under 3 minutes (M3). | AC-TECH-004 |
| FR-TECH-005 | Accept photo and file evidence on a daily report, stored privately (FR-DOC-001), with per-file size, count, and image-dimension limits enforced server-side. | AC-TECH-005 |
| FR-TECH-006 | Record blockers, delays, and suspension reasons on a daily report or work order with a reason code, and surface blocked work orders to office staff. | AC-TECH-006 |
| FR-TECH-007 | Support rescheduling a work order (new planned dates, reason) and reassigning it to another technician, both recorded with actor and timestamp. | AC-TECH-007 |
| FR-TECH-008 | Support partial completion: a work order may be `COMPLETED_PENDING_VERIFICATION` while the project remains `IN_PROGRESS`, and verification may accept or return for rework with a reason. | AC-TECH-008 |
| FR-TECH-009 | A technician shall see only assigned projects and work orders, and only their own reports and sessions. No technician-facing view shall expose company-wide financial, supplier, or other technicians' personal data. | AC-TECH-009 |
| FR-TECH-010 | A work order shall carry a progress percentage derived from reported progress and completion events, displayed as an estimate with its last-updated timestamp — never as an authoritative measure of completion. | AC-TECH-010 |
| FR-TECH-011 | **A work session is a first-class entity.** Each records start timestamp, completion timestamp, technician identity, project reference, work-order reference, and links to the reports and documentation captured during it. A project and a technician may have **many** sessions across many days. One start and one completion per project is explicitly insufficient. | AC-TECH-011 |
| FR-TECH-012 | **The server is authoritative for session timestamps.** A technician cannot backdate, edit, or overwrite a session start or completion. Corrections occur only through an authorised, audited **correction record** retaining the original values and attributing the correction with a mandatory reason. | AC-TECH-012 |
| FR-TECH-013 | **Completion verification is an explicit decision.** Technician submits a completion report → work order enters `PENDING_COMPLETION_VERIFICATION` → `ADMIN` reviews the report and documentation → verifies completion **or** returns for correction **with a mandatory reason** → the decision is audited. A technician ending a session does **not** complete the project. | AC-TECH-013 |
| FR-TECH-014 | Prepare a standardised WhatsApp report for an assigned project or work order — status, work date, progress summary, blockers, human-readable document references — as **prefilled text** the technician reviews and sends manually. Never post automatically, claim delivery, or use the WhatsApp Business API. | AC-WA-001 |

### 6.6 Finance and documents (`FR-FIN`, `FR-DOC`)

| ID | Requirement | AC |
|---|---|---|
| FR-FIN-001 | Create and issue quotations with lines, validity period, and status, producing a PDF on issuance. Quotations do not affect revenue or receivables until accepted and invoiced. | AC-FIN-001 |
| FR-FIN-002 | Issue sales invoices against a project, with lines, tax/discount fields where configured, due date, and an allocated document number from the invoice series. | AC-FIN-002 |
| FR-FIN-003 | On issuance, freeze an immutable snapshot of the invoice: customer identity and address, line descriptions, quantities, unit prices, totals, currency, **and the issuing company's identity and bank details**. Later edits to customer, item, or company master data shall not alter the issued invoice or its PDF. | AC-FIN-003 |
| FR-FIN-004 | Record customer payments with amount, date, method, reference, and allocation to one or more invoices, supporting **partial** payments with an accurate outstanding balance. | AC-FIN-004 |
| FR-FIN-005 | Compute and display per-invoice outstanding balance and an aged receivables view. Overdue is derived from due date versus current date, not a stored flag. | AC-FIN-005 |
| FR-FIN-006 | Record operational expenses (non-project and project-attributed) with category, amount, date, and optional attachment. | AC-FIN-006 |
| FR-FIN-007 | Accumulate project cost from recognised sources — goods receipts, supplier invoices, recorded payments — without double counting, and present cost composition by source. | AC-FIN-007 |
| FR-FIN-008 | Record due dates on invoices and payables and surface overdue items with days overdue. | AC-FIN-008 |
| FR-FIN-009 | Payment confirmation shall be a distinct action from recording, performed by a user holding the confirmation permission, capturing confirmer, timestamp, and optional evidence reference. | AC-FIN-009 |
| FR-FIN-010 | Provide payables and receivables summaries with outstanding totals per counterparty. | AC-FIN-010 |
| FR-FIN-011 | Support voiding an issued document with a mandatory reason, preserving the original record, its number, and its PDF. A voided number is never reused. | AC-FIN-011 |
| FR-FIN-012 | Support issuing a credit note or replacement document referencing the original, with its own allocated number, leaving the original intact and marked credited or superseded. | AC-FIN-012 |
| FR-DOC-001 | Store all uploaded files and generated PDFs on a **private local disk outside the webroot**, never publicly readable, served only through an authorising application route after a permission check — never via a permanent public URL and never via `public_path()` from business logic. | AC-DOC-001 |
| FR-DOC-002 | Maintain document metadata (type, owner entity, number where applicable, status, created/issued timestamps, version) with a change history. | AC-DOC-002 |
| FR-DOC-003 | Generate PDFs for issued quotations, supplier POs, sales invoices, credit notes, and receipts, storing each immutably alongside the record with a SHA-256 hash. | AC-DOC-003 |
| FR-DOC-004 | Allocate every human-readable document number server-side through the mechanism in [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md). No client, template, or report may generate a document number. | AC-DOC-004 |
| FR-DOC-005 | Retrieve any issued document by its document number and display its status and full history. | AC-DOC-005 |
| FR-DOC-006 | Document types shall be a **registry** (code, series key, configurable format pattern, separators, padding width, starting number, increment step, reset policy), so adding or reconfiguring a type needs no schema change. | AC-DOC-006 |
| FR-DOC-007 | Files may be attached to projects, work orders, daily reports, work sessions, invoices, payables, and expenses, each linked to exactly one owning record and inheriting its access rules. | AC-DOC-007 |
| FR-DOC-008 | An issued document's stored PDF and snapshot shall be immutable; regeneration is permitted only as a new derived artefact that never replaces the original bytes. | AC-DOC-008 |
| FR-DOC-009 | Address the storage layer through a single disk abstraction so that migrating to S3-compatible object storage later is a **configuration change plus a verified data migration**, not a module rewrite. Business logic and Blade templates shall contain no filesystem paths and no storage-driver assumptions. | AC-DOC-009 |

### 6.7 Reporting (`FR-RPT`)

| ID | Requirement | AC |
|---|---|---|
| FR-RPT-001 | Produce a daily progress report per project and per technician from submitted daily reports and work sessions, readable on a phone, and exportable. | AC-RPT-001 |
| FR-RPT-002 | Provide a project dashboard showing, per project: status, material readiness, schedule variance, cost to date, billed amount, outstanding balance. | AC-RPT-002 |
| FR-RPT-003 | Provide a material status report per project listing each line's stock status, reserved, purchased, received, issued, consumed, returned, and remaining quantity. | AC-RPT-003 |
| FR-RPT-004 | Provide receivables and payables ageing reports by counterparty and bucket. | AC-RPT-004 |
| FR-RPT-005 | Provide a procurement pipeline report by state: requested, `PENDING_APPROVAL`, approved, rejected, ordered, partially received, received, invoiced, paid. | AC-RPT-005 |
| FR-RPT-006 | Provide a technician workload report: assignments, scheduled days, open work orders, blocked work orders, and hours per work session. | AC-RPT-006 |
| FR-RPT-007 | All reports shall support a date range and export to CSV; documents for external parties export to PDF. | AC-RPT-007 |
| FR-RPT-008 | Report figures shall be reproducible: re-running a past range yields the same result unless underlying records were corrected, and corrections are visible in the audit trail. | AC-RPT-008 |

### 6.8 Data migration (`FR-DATA`)

| ID | Requirement | AC |
|---|---|---|
| FR-DATA-001 | Import master data (customers, contacts, suppliers, technicians, service categories, item catalogue) from CSV with per-row validation and a rejection report naming row, column, and reason. | AC-DATA-001 |
| FR-DATA-002 | Import shall be a two-step process: a validation/dry-run producing a full preview and error list, then an explicit commit. No import writes on validation alone. | AC-DATA-002 |
| FR-DATA-003 | Import open operational items (active projects, outstanding invoices and payables) with **existing historical document numbers preserved verbatim**. | AC-DATA-003 |
| FR-DATA-004 | Store imported historical numbers so they neither consume nor collide with newly allocated numbers in the same series, with the collision check running before commit. | AC-DATA-004 |
| FR-DATA-005 | Referenced master data shall be deactivated rather than deleted; deletion of a record referenced by any transaction shall be refused. | AC-DATA-005 |
| FR-DATA-006 | Provide a full export of its own data (structured tables plus files) for portability and backup verification. | AC-DATA-006 |

### 6.9 Audit (`FR-AUD`)

| ID | Requirement | AC |
|---|---|---|
| FR-AUD-001 | Write an audit record for every create, state transition, approval, issuance, cancellation, payment record/confirmation, permission change, configuration change, and override, capturing actor, action, target entity and ID, before/after values for changed fields, timestamp, and request correlation ID. | AC-AUD-001 |
| FR-AUD-002 | Audit records shall be append-only. No application path may update or delete one. | AC-AUD-002 |
| FR-AUD-003 | Audit writes for a state change shall occur in the same database transaction as the change, so a committed change without its audit record is impossible. | AC-AUD-003 |
| FR-AUD-004 | Any override of a blocked workflow shall require a reason and produce an audit record flagged as an override, visible to `PEMILIK`. | AC-AUD-004 |
| FR-AUD-005 | The audit log shall be viewable by `SUPER_ADMIN` and `PEMILIK`, and by an explicitly granted audit-viewer permission, filterable by actor, entity, action, and date range. | AC-AUD-005 |
| FR-AUD-006 | Retain audit records for the configured retention period and survive archival without loss of integrity. | AC-AUD-006 |

### 6.10 System administration and configuration (`FR-SYS`)

| ID | Requirement | AC |
|---|---|---|
| FR-SYS-001 | `SUPER_ADMIN` shall configure **SMTP settings**: host, port, encryption, username, password/credential, sender email, sender display name. Credentials are **encrypted at rest** using the application key and shall **never** be displayed once stored — the field is masked and write-only. | AC-SYS-001 |
| FR-SYS-002 | The SMTP screen shall provide a **connection test** and a **send test email** action, returning safe, actionable Indonesian messages. Changes shall take effect without an undocumented manual step, and queue workers shall pick them up safely. | AC-SYS-002 |
| FR-SYS-003 | `SUPER_ADMIN` shall maintain a **company profile**: company name, logo, address, contact information, bank name, account number, account holder. These values are available to generated documents. Issued documents store an **immutable snapshot** of company details; later profile changes shall not alter issued PDFs or snapshots. | AC-SYS-003 |
| FR-SYS-004 | `SUPER_ADMIN` shall configure document numbering per type: prefix, type code, format pattern and separators, padding width (initial 5), **starting sequence number**, **increment step**, period/reset policy, new-period rules, and the effective timing of changes. Changes are audited and shall never alter an already-issued number. | AC-SYS-004 |

### 6.11 WhatsApp reporting (`FR-WA`)

| ID | Requirement | AC |
|---|---|---|
| FR-WA-001 | Compose a standardised report text for a project or work order containing: current status, work date, progress summary, blockers, and authorised documentation references expressed as **human-readable document numbers**. The technician reviews the text before sending. | AC-WA-001 |
| FR-WA-002 | Construct a `https://wa.me/?text=` URL with the message encoded via `encodeURIComponent`, observe a practical length limit of roughly 2000 characters, and include **no sensitive financial data**, no internal URLs requiring authentication, and no private storage links. | AC-WA-002 |
| FR-WA-003 | The destination shall be **selected manually by the technician** by default. The system records "report prepared", which is **not** "message sent". A "mark as sent" action, if present, is an unverified self-declaration and must be labelled as such. When WhatsApp cannot be opened, show the composed text with a copy-to-clipboard fallback and instructions; never fail silently. | AC-WA-003 |

### 6.12 Mapping (`FR-MAP`)

| ID | Requirement | AC |
|---|---|---|
| FR-MAP-001 | Display project location on an in-app map using **OpenStreetMap** data (Leaflet or equivalent) when coordinates are present, with visible attribution to OpenStreetMap contributors. | AC-MAP-001 |
| FR-MAP-002 | Offer an **Open directions** action opening a Google Maps URL (`https://www.google.com/maps/dir/?api=1&destination=lat,lng`). Requires **no API key**; deeper Google Maps API integration is out of scope without separate approval. | AC-MAP-002 |
| FR-MAP-003 | Fetch map tiles in a manner compliant with the OpenStreetMap tile usage policy: no bulk downloading, no heavy caching, and a configurable tile source so a compliant provider or self-hosted tiles can be substituted as volume grows. | AC-MAP-003 |

### 6.13 Progressive web app (`FR-PWA`)

| ID | Requirement | AC |
|---|---|---|
| FR-PWA-001 | The application shall be installable as a PWA: valid web app manifest, icons at required sizes, and a registered service worker, on a mobile-first responsive layout optimised for technician use. | AC-PWA-001 |
| FR-PWA-002 | The service worker shall cache **only versioned static assets**. It shall **never** cache authenticated HTML or JSON responses, private files, or anything fetched with a session credential. | AC-PWA-002 |
| FR-PWA-003 | Implement explicit cache versioning and a defined update flow: `skipWaiting`/`clients.claim` strategy stated, stale caches removed on activation, and an in-app prompt when a new version is available. | AC-PWA-003 |
| FR-PWA-004 | Preserve an in-progress daily report draft **locally** (client-side `localStorage`/`IndexedDB`) and restore it after a reload or connection interruption. **Draft preservation only**: a draft is never a submitted record and may be lost on logout or device change (Q-PWA-01). | AC-PWA-004 |

---

## 7. Non-functional requirements

| ID | Requirement | Target / verification | AC |
|---|---|---|---|
| NFR-SEC-001 | All access requires authentication; no anonymous read of business data. | Every route except login/health returns 401 without a session | AC-SEC-001 |
| NFR-SEC-002 | Authorisation enforced server-side per **action and resource**, not by hiding UI. | Direct requests by an unauthorised role return 403 regardless of UI | AC-SEC-002 |
| NFR-SEC-003 | One account may hold multiple roles; effective permissions are the union, with conflicting-permission controls applied. | Role-combination test matrix | AC-SEC-003 |
| NFR-SEC-004 | Sessions expire after a configurable idle period and on password change. | Session test | AC-SEC-004 |
| NFR-SEC-005 | Passwords stored only as a slow salted hash; minimum length and breach-list check enforced. | Hash inspection + policy test | AC-SEC-005 |
| NFR-SEC-006 | `TEKNISI` accounts cannot reach financial, supplier-payment, settings, or company-wide reporting endpoints. | Negative permission tests per release | AC-SEC-006 |
| NFR-SEC-007 | Secrets supplied via environment/secret store; never committed, logged, or returned by a response. | Secret scan of repository and logs | AC-SEC-007 |
| NFR-SEC-008 | All traffic encrypted in transit; production requires TLS. | TLS configuration check | AC-SEC-008 |
| NFR-SEC-009 | Stored credentials (SMTP password and similar) are encrypted at rest with a key derived from `APP_KEY`, write-only in the UI, and never appear in page source, logs, client JavaScript, error output, or any response body. | Code review + response inspection + log inspection | AC-SEC-009 |
| NFR-SEC-010 | No private file is reachable without an authorisation check; downloads are streamed with `Content-Disposition` set, and directory listing is impossible. | Direct URL attempts + download behaviour test | AC-DOC-010 |
| NFR-AUD-001 | Audit log is append-only at the database level, not merely by convention. | Attempt update/delete as the application database user; must fail | AC-AUD-001 |
| NFR-AUD-002 | Audit coverage includes all state machines in §9 and all §6.9 events. | Coverage test per entity | AC-AUD-002 |
| NFR-AUD-003 | Audit records are tamper-evident (hash chaining) for issued-document, approval, and configuration events. | Chain verification job | AC-AUD-003 |
| NFR-AUD-004 | Overrides are always attributable and surfaced to `PEMILIK`. | Override report non-empty when overrides exist | AC-AUD-004 |
| NFR-AUD-005 | Audit retention period is configurable and defaults to the documented policy. | Retention job test | AC-AUD-005 |
| NFR-AUD-006 | Audit viewer is restricted to authorised roles and is itself audited. | Access + self-audit test | AC-AUD-006 |
| NFR-DATA-001 | All multi-table state changes occur in a single ACID transaction. | Failure-injection test: no partial writes | AC-DATA-001 |
| NFR-DATA-002 | Document-number uniqueness enforced by a database constraint, not application logic alone. | Concurrent insert test | AC-DATA-002 |
| NFR-DATA-003 | Referential integrity enforced by foreign keys; no orphan transactions. | Schema inspection + attempted orphan insert | AC-DATA-003 |
| NFR-DATA-004 | Automated daily backups of **both** the database and the private files directory, plus a restore rehearsal at least quarterly. | Restore log with measured duration | AC-DATA-004 |
| NFR-DATA-005 | Recovery targets: **RPO ≤ 24 h**, **RTO ≤ 8 h** at MVP scale, covering database and files. | Restore rehearsal against target | AC-DATA-005 |
| NFR-DATA-006 | Schema migrations are versioned, forward-only, and rehearsed on a copy before production. | Migration log per release | AC-DATA-006 |
| NFR-DATA-007 | Safe under concurrent use by ≥10 simultaneous users performing state-changing actions. | Load/concurrency test | AC-DATA-007 |
| NFR-DATA-008 | Retention: financial and issued documents per policy; operational data retained at least as long as its project. | Retention configuration review | AC-DATA-008 |
| NFR-DATA-009 | The engine shall be **MySQL 8.0.16+ or MariaDB 10.2.1+**, because earlier versions parse `CHECK` constraints and silently ignore them — a silent integrity failure. The exact engine and version is pinned and verified at deploy. | Version assertion at boot + constraint test | AC-DATA-009 |
| NFR-DATA-010 | Because DDL causes implicit commits, migrations shall be backward-compatible and shall never combine destructive DDL with code depending on the new shape in the same release (expand → migrate → contract). | Migration review checklist + rehearsal | AC-DATA-010 |
| NFR-OPS-001 | Separate environments for development, staging, production, with no production data in lower environments except anonymised copies. | Environment inventory | AC-OPS-001 |
| NFR-OPS-002 | Structured logs and error tracking sufficient to diagnose a reported failure without reproducing it. | Log inspection exercise | AC-OPS-002 |
| NFR-OPS-003 | Mobile-first responsive: all technician flows usable at 360 px width with touch targets ≥ 44 px. | Device test on 360 px viewport | AC-OPS-003 |
| NFR-OPS-004 | Performance: p95 response < 2 s on a 3G-class connection for technician flows; < 1 s on office LAN for list views. | Load test with recorded p95 | AC-OPS-004 |
| NFR-OPS-005 | Availability target 99% monthly during business hours; planned maintenance announced. | Uptime monitoring | AC-OPS-005 |
| NFR-OPS-006 | User-facing messages in Indonesian, stating what went wrong and what to do next; raw stack traces never shown. | Message review + error-path test | AC-OPS-006 |
| NFR-OPS-007 | UI primary language is Indonesian (`id`), including status labels and generated PDFs. | Localisation review | AC-OPS-007 |
| NFR-OPS-008 | Supported browsers: current Chrome, Safari (iOS), Firefox, Edge; no IE. | Browser matrix test | AC-OPS-008 |
| NFR-OPS-009 | All timestamps stored in **UTC** (offset is not stored by `TIMESTAMP`/`DATETIME`); conversion at the application boundary; all user-facing rendering in `Asia/Jakarta`; business dates use date-only columns. Relying on the database server's local timezone is forbidden. | Timezone test with a fixed UTC fixture rendered in Jakarta | AC-OPS-013 |
| NFR-OPS-010 | MVP is **online-first**. PWA support means installability, static-asset caching, and safe cache versioning — not offline operation. Full offline submission and synchronisation are deferred. | Cache inspection + offline behaviour test | AC-PWA-002, AC-PWA-003 |
| NFR-OPS-011 | Front-end initialisation shall be **idempotent across Turbo navigation**: initialise on `turbo:load`/`turbo:frame-load`, not `DOMContentLoaded`; reset transient UI state on `turbo:before-cache`; no duplicate listeners accumulate. | Navigate-away-and-back test; 5-navigation duplicate-handler test | AC-OPS-009…012 |
| NFR-PLAT-001 | The application runs on **Laravel 13.x**, latest stable patch at implementation time. Downgrading to Laravel 12 or earlier to satisfy a template or package is prohibited. | `composer.json` constraint + resolved version recorded | AC-OPS-014 |
| NFR-PLAT-002 | The UI is server-rendered **Blade** with **Hotwire Turbo** for navigation and progressive updates; no SPA framework. | Architecture review + request/response inspection | AC-OPS-009 |
| NFR-PLAT-003 | **TailAdmin Free Laravel** is the UI foundation, subject to a written compatibility and licence gate before Phase 1 exits. | Signed compatibility and licence record | AC-OPS-015 |
| NFR-PLAT-004 | The engine is **MySQL or MariaDB**, supported stable version, with engine-specific behaviour documented where it differs from generic SQL. | Environment record + NFR-DATA-009 assertion | AC-DATA-009 |
| NFR-PLAT-005 | Default and primary locale `id`; business timezone `Asia/Jakarta`; currency IDR. | Configuration review | AC-OPS-007 |
| NFR-PLAT-006 | Responsive mobile-first web app with PWA support, installable on Android and iOS browsers. | Install test on two devices | AC-PWA-001 |
| NFR-PLAT-007 | **Modular monolith**: one deployable application with enforced module boundaries and one database. | Module boundary check in CI | AC-OPS-016 |
| NFR-PLAT-008 | Minimum framework version **Laravel ≥ 13.29.0**, stated by the client as a security requirement (CVE reference unverified offline). Before any deployment, `composer audit` is run and the resolved version confirmed against the official advisory, with evidence recorded. | Release gate record per deployment | AC-OPS-014 |
| NFR-PLAT-009 | The stack baseline is a client requirement, not a proposal. Any deviation requires written client authorisation and a new ADR. | ADR-008 review | AC-OPS-014 |

---

## 8. Business and validation rules

| ID | Rule |
|---|---|
| BR-PROJ-001 | Quotation, customer PO, material lines, and multiple technicians are **optional per project**. No step may be forced merely because it exists. |
| BR-PROJ-002 | Project status, work-order status, work-session state, and material readiness are **independent** state machines. A change in one never implies a change in another; any coupling is explicit (BR-PROJ-003). |
| BR-PROJ-003 | A project may enter `IN_PROGRESS` while material readiness is not `READY_TO_START` only via a recorded override with reason (FR-AUD-004). |
| BR-PROJ-004 | A project may be `CLOSED` only when: all work orders are `VERIFIED` or `CANCELLED`; no material line is in an open issued state; all invoices are `ISSUED`/`PAID`/`VOID`/`CREDITED` (none `DRAFT`); no work session is `ACTIVE`; and any override is recorded. |
| BR-PROJ-005 | Cancelling a project never deletes financial or document records. Outstanding payables and invoices remain and must be resolved separately. |
| BR-MATL-001 | An unverified quantity is **never** treated as available. `UNKNOWN` blocks reservation (FR-MATL-005). |
| BR-MATL-002 | Reserved quantity across all projects shall never exceed verified on-hand quantity at the moment of reservation. |
| BR-MATL-003 | Quantities change only through append-only movements. There is no "edit quantity" operation anywhere. |
| BR-MATL-004 | Material issued to a technician remains attributable until consumed, returned, or written off. No state may leave quantity unattributed. |
| BR-MATL-005 | A PO does **not** increase available stock. Stock becomes available only on recorded goods receipt. |
| BR-PROC-001 | Approval, PO issuance, goods receipt, supplier invoice, and payment are five distinct states. No state may be skipped in reporting, even where one person performs several actions. |
| BR-PROC-002 | A payment recorded is not a payment confirmed. Confirmation is a separate, attributable action. |
| BR-PROC-003 | Over-receipt beyond ordered quantity requires explicit approval and is recorded as an exception. |
| BR-PROC-004 | The system never initiates, schedules, or executes a bank transfer. It records that a transfer occurred, per the user's assertion, with evidence where available. |
| BR-PROC-005 | **Every purchase request requires `PEMILIK` approval.** No threshold, no auto-approval, no delegation, no below-threshold exception. An override may **never** waive this requirement: it is a business control owned by the client, not an authorisation conflict. |
| BR-FIN-001 | Issued documents are immutable in content and number. Corrections happen via credit note or replacement (FR-FIN-012). |
| BR-FIN-002 | A document number is allocated only at issuance, never at draft creation, and never reused after void. |
| BR-FIN-003 | A payment allocation may not exceed the invoice outstanding balance; overpayment is an unapplied credit, not a negative balance. |
| BR-FIN-004 | No issued invoice may be edited. Corrections follow BR-FIN-001. |
| BR-FIN-005 | Project cost recognition counts each cost source once; a goods receipt, its supplier invoice, and its payment are three views of one cost, not three costs (FR-FIN-007). |
| BR-FIN-006 | **Increment step and void gaps are different concepts.** The step is the difference between consecutive allocated values (step 5 ⇒ 00001, 00006, 00011). A gap is an issued number whose document was voided. Issued numbers are retained permanently and never reused; **a step of 5 does not grant permission to reuse a voided number**. |
| BR-TECH-001 | A technician may act only on their own assignments and only within permitted actions. |
| BR-TECH-002 | A work order may be verified only by a user holding verification permission and not by the technician who completed it, unless an override with reason is recorded (SECURITY §6). |
| BR-TECH-003 | A daily report may be corrected only by superseding it with a new version; the original remains retrievable. |
| BR-TECH-004 | Work-session start and completion timestamps are **server-generated and immutable**. A technician cannot backdate or overwrite them; corrections require an authorised, audited correction record retaining the original values. |
| BR-TECH-005 | A technician completing work or ending a session does **not** complete the project or the work order. Completion requires `ADMIN` verification, or a return for correction with a mandatory reason. |
| BR-DOC-001 | Every issued document has exactly one stored, immutable PDF matching its issuance snapshot. |
| BR-DOC-002 | Document numbers are generated exclusively server-side (FR-DOC-004). Frontend validation is never a control. |
| BR-SYS-001 | Issued documents capture an immutable snapshot of company identity and bank details at issuance. Later profile changes never alter an issued document, its snapshot, or its PDF. |
| BR-AUD-001 | No state change is valid without an audit record in the same transaction. |
| BR-DATA-001 | Historical document numbers imported from spreadsheets are preserved verbatim and never re-allocated. |
| BR-DATA-002 | Referenced records are deactivated, never deleted. |

---

## 9. Status definitions (canonical)

**Normative and reused verbatim** by all other documents. Each is a separate state machine (BR-PROJ-002). Transition tables live in the workflow documents named below.

### 9.1 Project status — owner: [`./workflows/PROJECT_LIFECYCLE.md`](./workflows/PROJECT_LIFECYCLE.md)

| Status | Meaning |
|---|---|
| `DRAFT` | Being prepared; not yet committed to the field. |
| `PLANNED` | Scope, customer, and schedule agreed; not yet scheduled to people. |
| `SCHEDULED` | Technicians assigned and dates set; not yet started. |
| `IN_PROGRESS` | Work has begun; work sessions and daily reports expected. |
| `ON_HOLD` | Temporarily suspended with a recorded reason. |
| `PENDING_COMPLETION_VERIFICATION` | All work orders submitted; awaiting office verification. |
| `COMPLETED` | Work verified complete; billing/closure activities may remain. |
| `CLOSED` | All closure conditions met (BR-PROJ-004); read-only except audit and corrections. |
| `CANCELLED` | Abandoned with recorded reason; records retained. |

### 9.2 Work order status — owner: [`./workflows/TECHNICIAN_OPERATIONS.md`](./workflows/TECHNICIAN_OPERATIONS.md)

| Status | Meaning |
|---|---|
| `DRAFT` | Created, not yet assigned. |
| `ASSIGNED` | Assigned to a technician, not yet acknowledged. |
| `ACCEPTED` | Technician acknowledged the assignment. |
| `IN_PROGRESS` | Work started; sessions and daily reports expected. |
| `BLOCKED` | Cannot proceed; blocker recorded (FR-TECH-006). |
| `COMPLETED_PENDING_VERIFICATION` | Technician submitted completion; awaiting verification (FR-TECH-013). |
| `VERIFIED` | Completion accepted by an authorised verifier. |
| `CANCELLED` | Withdrawn with reason. |

### 9.3 Material readiness status (project level, derived) — owner: [`./workflows/PROCUREMENT_AND_MATERIALS.md`](./workflows/PROCUREMENT_AND_MATERIALS.md)

| Status | Meaning |
|---|---|
| `NOT_ASSESSED` | No material requirements recorded yet. |
| `REQUIREMENTS_INCOMPLETE` | Lines exist but some remain `UNKNOWN`. |
| `AWAITING_PROCUREMENT` | Verified shortfall exists; purchasing not complete. |
| `READY_TO_START` | All lines covered by verified stock or received goods. |
| `PARTIALLY_ISSUED` | Some materials issued to technicians. |
| `FULLY_ISSUED` | All required materials issued. |
| `CLOSED` | All issued material consumed, returned, or written off. |

Derived, never hand-set, and displayed alongside — never instead of — project status.

### 9.4 Material line: stock status and fulfilment status (orthogonal)

| Field | Values |
|---|---|
| `stock_status` | `UNKNOWN`, `VERIFIED` |
| `fulfilment_status` | `DRAFT`, `VERIFIED`, `RESERVED`, `PENDING_PURCHASE`, `REQUESTED`, `ORDERED`, `PARTIALLY_RECEIVED`, `RECEIVED`, `ISSUED`, `CONSUMED`, `RETURNED`, `WRITTEN_OFF`, `CLOSED` |

### 9.5 Purchase request status

**Vocabulary revision 2026-10-25:** `SUBMITTED` renamed `PENDING_APPROVAL` to match the mandatory-approval workflow (FR-PROC-011). No other value changed.

| Status | Meaning |
|---|---|
| `DRAFT` | Being prepared. |
| `PENDING_APPROVAL` | Awaiting the `PEMILIK` decision. Formerly `SUBMITTED`. |
| `APPROVED` | Approved by `PEMILIK`; may become a PO. |
| `REJECTED` | Rejected with reason; terminal. |
| `CONVERTED_TO_PO` | A PO has been generated. |
| `CANCELLED` | Withdrawn with reason. |

### 9.6 Purchase order status

| Status | Meaning |
|---|---|
| `DRAFT` | Prepared, number not yet allocated. May only exist for an `APPROVED` request. |
| `ISSUED` | Numbered, snapshotted, sent to supplier. |
| `PARTIALLY_RECEIVED` | Some lines received; receipt outstanding. |
| `RECEIVED` | All lines fully received. |
| `CLOSED` | Received and financially settled to the extent required. |
| `CANCELLED` | Voided with reason; number retained. |

### 9.7 Supplier payable status

`RECORDED` (captured, unpaid) · `PARTIALLY_PAID` · `PAID` (fully paid and confirmed) · `DISPUTED` (under query; blocks closure) · `CANCELLED` (voided with reason).

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

### 9.14 Work session status (new) — owner: [`./workflows/TECHNICIAN_OPERATIONS.md`](./workflows/TECHNICIAN_OPERATIONS.md)

| Status | Meaning |
|---|---|
| `ACTIVE` | Session started; server timestamp recorded; no completion yet. |
| `COMPLETED` | Technician ended the session; server timestamp recorded. |
| `CORRECTED` | A timestamp or attribution was corrected by an authorised user; original values retained in the correction record. |
| `ABANDONED` | Closed by an authorised user because the technician never ended it; reason required, both timestamps attributed to the correction. |

**Only `ACTIVE` is non-terminal.** A session is never deleted. Multiple sessions per technician per project are normal.

### 9.15 Completion verification outcome (new) — owner: [`./workflows/TECHNICIAN_OPERATIONS.md`](./workflows/TECHNICIAN_OPERATIONS.md)

| Outcome | Effect |
|---|---|
| `VERIFIED` | Work order → `VERIFIED`; project may advance when all work orders are terminal. |
| `RETURNED_FOR_CORRECTION` | Work order → `IN_PROGRESS`; mandatory reason recorded; technician notified. |

---

## 10. Three-layer finance model

| Layer | Content | MVP | Rationale |
|---|---|---|---|
| **L1 — Operational finance and cash tracking** | Payments recorded/confirmed, expenses, project cost accumulation, cash movement visibility | **In scope** | What the owner actually needs to run the business |
| **L2 — AR/AP** | Invoices, payables, allocations, partial payments, ageing, outstanding balances | **In scope** | Directly resolves billing disputes and payment chasing |
| **L3 — Formal accounting** | Journals, chart of accounts, ledgers, trial balance, tax reporting, statutory statements | **[Out of Scope]** — **Q-FIN-01** | Requires accounting expertise and statutory rules; the operational layer can export to an accountant |

**[Open Question] Q-FIN-01 — Formal accounting depth.** Does the business (or its accountant) require statutory statements, PPN (VAT) reporting, or PPh withholding from this system? **Proposed default:** MVP stops at L1+L2 and provides a clean export (FR-DATA-006). **Decision trigger:** Phase 0 confirmation. **Rationale:** building a general ledger without an accountant's input produces statements nobody can file.

---

## 11. Document numbering — summary

Full standard: [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md). PRD-level obligations:

1. Every document type has its **own series** (FR-DOC-006).
2. The format is a **configurable template**, not an invoice-only hardcode. Initial invoice format: `WJC/INV/2026/10/00001` — prefix / type / year / month / 5-digit sequence.
3. Numbers are allocated **atomically on the server at formal issuance only**, never at draft creation, never by the client, never by `MAX()+1` (FR-DOC-004, BR-DOC-002).
4. Uniqueness is enforced by a **database constraint**, not application checks alone (NFR-DATA-002).
5. Issued numbers are **immutable and never reused**, including after void (BR-FIN-002, FR-FIN-011).
6. **Increment step ≠ void gap** (BR-FIN-006). A step of 5 yields 00001, 00006, 00011; it does not license reuse of a voided number.
7. **Two distinct risks** are analysed: internal duplication (solvable here) and conflict with external/client numbering rules (**not** solvable by an internal database alone).
8. Historical numbers are migrated verbatim (BR-DATA-001).

**[Open Question] Q-NUM-01 — External numbering constraints.** The confirmed conflict may be internal duplication, or a client-side system rejecting numbers that duplicate another vendor's. **The root cause remains unconfirmed.** **Proposed default:** an internal-consistency-guaranteed, configurable format with a company prefix, treating external constraints as configurable per client. **Decision trigger:** written confirmation from each affected client on permitted length/charset, whether a prefix is allowed, whether validation occurs against a national registry, whether historical numbers must be preserved, whether gaps are acceptable, and how cancellations must be represented. **A company prefix alone does not resolve external conflicts** — that claim is explicitly rejected in the numbering standard.

---

## 12. Security, auditability, backup and recovery

- **Least privilege by default.** Technicians have assignment-scoped access only (FR-TECH-009).
- **Action-and-resource permissions**, checked server-side (NFR-SEC-002).
- **Separation of duties proportional to team size.** Where one person must hold conflicting permissions, an override with mandatory reason and `PEMILIK` visibility is required (FR-AUD-004). An override never waives the purchase-approval requirement (BR-PROC-005).
- **Immutability of issued artefacts** (BR-FIN-001, BR-SYS-001, FR-DOC-008).
- **Append-only audit with transactional coupling** (FR-AUD-002, FR-AUD-003).
- **Private local storage first.** Files live on a private disk outside the webroot; S3-compatible storage is a future migration, not a dependency (FR-DOC-001, FR-DOC-009).
- **Backup:** automated daily, covering the database **and** the private files directory, with quarterly restore rehearsal (NFR-DATA-004, NFR-DATA-005).
- **Secrets** never in the repository or logs; stored credentials encrypted at rest (NFR-SEC-007, NFR-SEC-009).

**[Open Question] Q-OPS-01 — Hosting, backup location, retention, and access policy.** Not confirmed. **Proposed default:** a single small VPS or managed PHP host in a region close to Indonesia running the application and MySQL/MariaDB; daily encrypted database dumps plus a files-directory archive copied off-host; 30-day retention plus monthly archival; financial records retained at least 10 years subject to accountant confirmation. **Rationale:** keeps architecture and cost planning moving and is revisitable.

---

## 13. Assumptions, risks, dependencies

### 13.1 Assumptions

| ID | Assumption | Impact if false |
|---|---|---|
| A-01 | Users have smartphones with a modern browser and intermittent mobile data at sites. | Mobile-first web fails; native app or offline mode becomes necessary |
| A-02 | Indonesian-only UI is sufficient. **[Confirmed]** as a client requirement; risk is a later bilingual need. | i18n framework needed earlier |
| A-03 | Office staff can operate a web application with minimal training. | Adoption risk; heavier UX investment |
| A-04 | Historical spreadsheet data is recoverable and at least partially clean. | Migration scope grows (Q-DATA-01) |
| A-05 | One legal entity, one currency (IDR), one timezone (`Asia/Jakarta`). | Multi-entity/currency scope added |
| A-06 | Suppliers do not need system access. | Supplier portal scope added |
| A-07 | A single designated person owns data quality during pilot. | Migration and cleanup stall |
| A-08 | **TailAdmin Free Laravel has a release usable with Laravel 13, or can be reduced to a view layer.** Unverified offline. | UI foundation must be replaced; schedule impact (Q-UI-01) |
| A-09 | The `CVE-2026-114509` / 13.29.0 relationship is accurate. Unverified offline. | Minimum version may be wrong; release gate still applies |

### 13.2 Risks

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | Field adoption fails; technicians keep using WhatsApp | High | High | M3 target (<3 min), mobile-first design, WhatsApp preparation kept as a familiar bridge, pilot with 2–3 technicians |
| R-02 | Invoice conflict recurs with a client despite internal fix | Medium | High | Dual-risk analysis; per-client format configuration; Q-NUM-01 confirmation |
| R-03 | Material data quality is poor, so reservations are wrong | High | Medium | Explicit `UNKNOWN` state forces verification; reconciliation checkpoints (FR-MATL-013) |
| R-04 | Scope creep toward full ERP | Medium | High | Explicit exclusions §5.3; AGENTS.md scope boundary |
| R-05 | One person holds conflicting permissions and abuses them | Low | High | Override logging with reason + `PEMILIK` visibility (FR-AUD-004) |
| R-06 | Historical migration corrupts numbering | Medium | High | Verbatim preservation, pre-commit collision check (FR-DATA-004) |
| R-07 | Backup exists but restore was never tested | Medium | High | Quarterly restore rehearsal covering database and files (NFR-DATA-004) |
| R-08 | Small team cannot sustain documentation upkeep | Medium | Medium | AGENTS.md documentation duties; docs reviewed at each phase exit |
| R-09 | **TailAdmin incompatible with Laravel 13**, with pressure to downgrade | Medium | High | NFR-PLAT-001 forbids downgrade; view-layer path documented; gate AC-OPS-015 |
| R-10 | **The stated CVE fix version is wrong** | Low | High | Release gate runs `composer audit` and checks the advisory before every deployment (AC-OPS-014) |
| R-11 | Turbo re-initialisation bugs cause silent UI breakage | Medium | Medium | Idempotent init pattern (NFR-OPS-011) plus AC-OPS-009…012 |

### 13.3 Dependencies

| ID | Dependency | Blocks |
|---|---|---|
| D-01 | Phase 0 process mapping and open-question answers | All build phases |
| D-02 | Document numbering implemented and tested | Any issuance (Phase 2+) |
| D-03 | Audit logging implemented | Approvals, issuance, payments (Phase 1+) |
| D-04 | Client confirmation of numbering constraints (Q-NUM-01) | Final invoice format; not the internal mechanism |
| D-05 | Spreadsheet profiling (Q-DATA-01) | Migration (Phase 6) |
| D-06 | Hosting and backup decisions (Q-OPS-01) | Production deployment |
| D-07 | Accountant input (Q-FIN-01) | Any L3 accounting scope |
| D-08 | **TailAdmin compatibility and licence sign-off (Q-UI-01)** | Phase 1 exit; all UI work |
| D-09 | **Laravel 13.29.0 / CVE verification (Q-SEC-01)** | Any deployment (release gate) |

---

## 14. Open questions register

Rows are never deleted. Resolved questions keep their entry and record the resolution.

| ID | Question | Proposed default | Owner | Trigger | Status |
|---|---|---|---|---|---|
| Q-PROC-01 | Purchasing approval thresholds and mandatory-approval conditions | **Resolved 2026-10-25:** every purchase requires `PEMILIK` approval. No threshold, no auto-approval, no delegation, no below-threshold exception (FR-PROC-011, BR-PROC-005) | Owner | — | **Resolved** |
| Q-NUM-01 | Final invoice-numbering rules and client validation requirements | Internal-consistent configurable format with company prefix; external constraints per client | Owner + affected clients | Before Phase 5 invoice go-live | Open |
| Q-NUM-06 | **Numbering reset scope**: monthly, yearly, or continuous | Monthly, matching the initial format example; configurable per series | Owner | Before Phase 1 numbering build | Open |
| Q-NUM-07 | **Meaning and permitted range of the configurable "gap"/increment step** | Ambiguous. Proposed: the step is the increment between consecutive allocations (default 1). "Gap" is redefined as a voided-number hole, never reused. Confirm the business is not asking for something else | Owner + accountant | Before Phase 1 numbering build | Open |
| Q-NUM-08 | How starting numbers and step changes interact with existing allocations and historical imports | Future allocations only; counter never below the historical high-water mark; audited; never renumbers existing documents | Owner | Before Phase 1 numbering build | Open |
| Q-FIN-01 | Required financial reports and tax responsibilities | L1+L2 only; export for accountant | Owner + accountant | Phase 0 | Open |
| Q-FIN-02 | Which finance features are actually required: down payments, partial payments, supplier payments, expenses | All four are in MVP as scoped; confirm none is unnecessary | Owner | Phase 0 | Open |
| Q-FIN-03 | Invoice tax/discount fields and statutory reporting | Configurable rate and discount fields, no statutory format | Owner + accountant | Phase 0 | Open |
| Q-FIN-04 | Whether `PEMILIK` performs financial actions beyond approval, and which destructive actions are prohibited | May record and confirm payments and view all reports; prohibited: editing issued documents, deleting audit records, altering numbering history | Owner | Phase 0 | Open |
| Q-DATA-01 | Existing spreadsheet structures and historical data quality | Phase 0 profiling produces mapping spec | Admin | Phase 0 exit | Open |
| Q-PROC-02 | Supplier delivery and goods-receipt practices | Line-level partial receipt with discrepancy recording | Admin | Phase 3 | Open |
| Q-ROLE-01 | Whether the team-leader designation stays useful | Per-project designation, optional review step | Owner | Phase 0 | Open |
| Q-INV-01 | Required stock valuation and inventory-accounting depth | Project-scoped quantities only; no valuation | Owner | Phase 6+ | Open |
| Q-OPS-01 | Hosting, backup, retention, and access policies | Small VPS + MySQL/MariaDB, daily DB and files backup, 10-year financial retention | Owner | Phase 1 | Open |
| Q-BASE-01 | Baseline operational metrics | Phase 0 measurement | Admin | Phase 0 | Open |
| Q-UX-01 | Whether a native app is required | Mobile-first PWA; native out of scope | Owner | After pilot feedback | Open |
| Q-UI-01 | **TailAdmin release, licence terms, dependency set, Laravel 13 compatibility** | Verify exact package/repo, release tag, licence and attribution, declared `laravel/framework` constraint, Tailwind/Vite versions. If no 13-compatible release: vendor as view layer only | Technical lead | Phase 1 entry gate | Open |
| Q-SEC-01 | **Verification of CVE-2026-114509 and whether 13.29.0 is the fixed version** | Not verified offline. Confirm via official advisories, the GitHub Advisory Database, and `composer audit` before deployment | Technical lead | Before first deployment | Open |
| Q-MAIL-01 | Single SMTP profile or multiple | Single profile in MVP; multiple deferred | Owner | Phase 1 | Open |
| Q-MAIL-02 | Which email events need notifications | **[Proposed]** purchase decision, invoice issued, overdue reminder, completion verification — each requires business approval | Owner | Phase 4 | Open |
| Q-SYS-01 | Required company contact fields, and single or multiple bank accounts | Required: company name, address, phone. Optional: email, website, logo. Single bank account in MVP | Owner | Phase 1 | Open |
| Q-WA-01 | WhatsApp destination model: fixed per project/team, or manual selection | **Manual selection** by the technician, because group IDs are not reliably available without the Business API | Owner | Phase 4 | Open |
| Q-FILE-01 | Photo/file size, count, and retention limits | 5 MB per file, 10 files per report, retained with the project | Owner | Phase 4 | Open |
| Q-MAP-01 | Tile provider at production volume | OpenStreetMap public tiles for pilot; switch to a compliant provider or self-hosted tiles as volume grows | Technical lead | Phase 6 | Open |
| Q-PWA-01 | Must local drafts survive browser restart, logout, or device change? | No in MVP: drafts are device-local and may be lost; drafts are never treated as submitted records | Owner | Phase 4 | Open |

---

## 15. Acceptance criteria

All criteria are held in [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md), grouped by workflow, each referencing the FR/NFR IDs above and stating its test level. Requirement tables in §6 and §7 name their `AC-` references directly. Test approach: [`./implementation/TESTING_STRATEGY.md`](./implementation/TESTING_STRATEGY.md).

---

## 16. Document map

| Document | Owns |
|---|---|
| [`./architecture/ARCHITECTURE.md`](./architecture/ARCHITECTURE.md) | System structure, module boundaries, platform baseline, Turbo lifecycle, deployment |
| [`./architecture/DATA_MODEL.md`](./architecture/DATA_MODEL.md) | Entities, MySQL/MariaDB types, constraints, snapshots, movement ledger |
| [`./architecture/SECURITY_AND_ACCESS_CONTROL.md`](./architecture/SECURITY_AND_ACCESS_CONTROL.md) | Four roles, action×resource matrix, scoping, separation of duties, overrides |
| [`./workflows/PROJECT_LIFECYCLE.md`](./workflows/PROJECT_LIFECYCLE.md) | Project and work-order state machines, closure |
| [`./workflows/PROCUREMENT_AND_MATERIALS.md`](./workflows/PROCUREMENT_AND_MATERIALS.md) | Requirement → reservation → purchase → approval → receipt → issue → consumption |
| [`./workflows/TECHNICIAN_OPERATIONS.md`](./workflows/TECHNICIAN_OPERATIONS.md) | Scheduling, work sessions, daily reporting, evidence, completion verification, WhatsApp |
| [`./workflows/FINANCE_AND_DOCUMENTS.md`](./workflows/FINANCE_AND_DOCUMENTS.md) | Quotation → invoice → payment, payables, documents, corrections |
| [`./standards/DOCUMENT_NUMBERING.md`](./standards/DOCUMENT_NUMBERING.md) | Numbering template, atomic allocation, dual-risk analysis |
| [`./standards/AUDIT_LOGGING.md`](./standards/AUDIT_LOGGING.md) | Audit event model, immutability, retention |
| [`./implementation/IMPLEMENTATION_PLAN.md`](./implementation/IMPLEMENTATION_PLAN.md) | Phased roadmap with exit criteria and gates |
| [`./implementation/TESTING_STRATEGY.md`](./implementation/TESTING_STRATEGY.md) | Test levels, critical scenarios |
| [`./implementation/ACCEPTANCE_CRITERIA.md`](./implementation/ACCEPTANCE_CRITERIA.md) | `AC-` register |
| [`./decisions/`](./decisions/) | Architecture decision records |
