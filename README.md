# WJ-CCTV Business Operations System

A centralised, auditable business operations platform for **WJ-CCTV.com** — covering project delivery, field technicians, procurement, project-based material control, document management, and operational finance.

**Current status: documentation phase. No application code exists.**

---

## 1. Purpose

WJ-CCTV.com provides CCTV-related services and other business services. Operations currently run on Excel/Google Sheets, Word, and WhatsApp, with no centralised inventory and no reliable audit trail.

This system replaces that with a single system of record that answers, with evidence:

- What was promised to the customer, and what was delivered?
- What materials were needed, what was actually available, what was purchased, and what was used?
- Who approved a purchase, and did the money actually move?
- What was invoiced, at what number, and what has been paid?
- Who changed what, and when?

The immediate business driver is a **confirmed invoice-numbering conflict** whose cause is unconfirmed. Eliminating internal duplicate numbering is a first-class requirement, not a side effect.

---

## 2. Current status

| Aspect | Status |
|---|---|
| Repository contents | Documentation only |
| Application code | **None** |
| Technology stack | **Not installed.** Recommended only — see [ADR-001](./docs/decisions/ADR-001-modular-monolith-and-stack.md) |
| Build / test / run commands | **None exist** |
| Dependencies | None |
| Configuration files | None (`package.json`, lockfiles, `tsconfig.json` etc. deliberately absent) |
| Git repository | Not initialised |
| Implementation | **Not started** — awaits authorisation |

**Repository verification (2026-10-25):** the workspace was confirmed empty before authoring — zero entries including hidden files, no `.git`, no pre-existing `AGENTS.md`, no `docs/`. There is therefore **no established technology stack, no codebase, no build or test commands, and no existing conventions.** Every technical statement in this documentation set is a proposal, not a description of an existing system.

---

## 3. Key capabilities

| Capability | Summary | Detail |
|---|---|---|
| Customer management | Customers, multiple contacts, service categories, duplicate warning | [PRD](./docs/PRD.md) §6.1 |
| Project management | Multi-day projects, one-to-many technician assignment, optional quotation/customer PO | [PRD](./docs/PRD.md) §6.2 |
| Material control | Requirement lines with explicit `UNKNOWN`/`VERIFIED` stock status, reservations, append-only movement ledger | [PRD](./docs/PRD.md) §6.4 |
| Procurement | Purchase request → approval → supplier PO → partial goods receipt → payable → payment | [PRD](./docs/PRD.md) §6.3 |
| Field operations | Mobile-first daily reports, photos, blockers, partial completion, verification | [PRD](./docs/PRD.md) §6.5 |
| Finance | Quotations, invoices, partial payments, AR/AP, expenses, project cost | [PRD](./docs/PRD.md) §6.6 |
| Document control | Server-side numbering, immutable issued snapshots, PDF history | [PRD](./docs/PRD.md) §6.6, §11 |
| Audit | Append-only, transactionally coupled, hash-chained | [PRD](./docs/PRD.md) §6.9 |
| Reporting | Daily progress, project dashboard, material status, AR/AP ageing | [PRD](./docs/PRD.md) §6.7 |

### Deliberately excluded from MVP

Full ERP, bank integration, native mobile app, microservices, warehouse management (bins/lots/serials), multi-currency, statutory accounting, customer/supplier portals, GPS tracking.

Rationale for each: [PRD §5.3](./docs/PRD.md).

---

## 4. Documentation index

### Entry points

| Document | Purpose |
|---|---|
| [`AGENTS.md`](./AGENTS.md) | Working agreement for coding agents: invariants, reading order, boundaries |
| [`docs/PRD.md`](./docs/PRD.md) | **Authoritative** scope, requirements, business rules, status vocabulary |
| [`docs/README.md`](./docs/README.md) | Documentation map and ownership |

### Architecture

| Document | Owns |
|---|---|
| [`docs/architecture/ARCHITECTURE.md`](./docs/architecture/ARCHITECTURE.md) | Module boundaries, layers, technology recommendation, deployment |
| [`docs/architecture/DATA_MODEL.md`](./docs/architecture/DATA_MODEL.md) | Entities, relationships, constraints, snapshots, movement ledger |
| [`docs/architecture/SECURITY_AND_ACCESS_CONTROL.md`](./docs/architecture/SECURITY_AND_ACCESS_CONTROL.md) | Roles, permission matrix, scoping, separation of duties, overrides |

### Workflows

| Document | Owns |
|---|---|
| [`docs/workflows/PROJECT_LIFECYCLE.md`](./docs/workflows/PROJECT_LIFECYCLE.md) | Project and work-order state machines, scheduling, closure |
| [`docs/workflows/PROCUREMENT_AND_MATERIALS.md`](./docs/workflows/PROCUREMENT_AND_MATERIALS.md) | Requirement → verification → reservation → purchase → receipt → issue → consumption |
| [`docs/workflows/TECHNICIAN_OPERATIONS.md`](./docs/workflows/TECHNICIAN_OPERATIONS.md) | Scheduling, daily reporting, evidence, blockers, verification |
| [`docs/workflows/FINANCE_AND_DOCUMENTS.md`](./docs/workflows/FINANCE_AND_DOCUMENTS.md) | Quotation → invoice → payment, payables, document history, corrections |

### Standards

| Document | Owns |
|---|---|
| [`docs/standards/DOCUMENT_NUMBERING.md`](./docs/standards/DOCUMENT_NUMBERING.md) | Numbering format, atomic allocation, dual-risk analysis |
| [`docs/standards/AUDIT_LOGGING.md`](./docs/standards/AUDIT_LOGGING.md) | Audited events, transactional coupling, immutability, retention |

### Implementation

| Document | Owns |
|---|---|
| [`docs/implementation/IMPLEMENTATION_PLAN.md`](./docs/implementation/IMPLEMENTATION_PLAN.md) | Phase 0–6 roadmap, dependencies, exit criteria |
| [`docs/implementation/TESTING_STRATEGY.md`](./docs/implementation/TESTING_STRATEGY.md) | Test levels, critical scenarios, verification philosophy |
| [`docs/implementation/ACCEPTANCE_CRITERIA.md`](./docs/implementation/ACCEPTANCE_CRITERIA.md) | The `AC-` register (109 criteria) |

### Decisions

| Document | Decision |
|---|---|
| [`docs/decisions/`](./docs/decisions/) | 7 ADRs: monolith + stack, numbering, single DB + snapshots, audit, authorisation, private storage, migration |

---

## 5. Architecture overview (proposed)

```
┌─────────────────────────────────────────────────────┐
│  Presentation   Indonesian UI, mobile-first responsive │
├─────────────────────────────────────────────────────┤
│  Application    Use cases · transaction boundary      │
│                 authorisation · audit · numbering     │
├─────────────────────────────────────────────────────┤
│  Domain         State machines · business rules       │
├─────────────────────────────────────────────────────┤
│  Infrastructure Repositories · storage · PDF · clock  │
└───────────────┬─────────────────────┬───────────────┘
                │                     │
      ┌─────────▼────────┐   ┌────────▼──────────┐
      │  PostgreSQL      │   │ Private object    │
      │  single database │   │ storage (S3)      │
      └──────────────────┘   └───────────────────┘
```

**Style:** modular monolith with enforced module boundaries — chosen because the two highest-risk requirements (atomic number allocation, reservation correctness) are trivial inside one transaction and materially harder across services.

**Modules:** `identity`, `audit`, `numbering`, `customer`, `catalog`, `supplier`, `project`, `workorder`, `material`, `procurement`, `finance`, `document`, `reporting`.

**Stack (proposed, not installed):** TypeScript · Node.js LTS · PostgreSQL · typed ORM with explicit transaction control · S3-compatible private storage · server-rendered mobile-first UI.

Detail: [`docs/architecture/ARCHITECTURE.md`](./docs/architecture/ARCHITECTURE.md) · Decision: [ADR-001](./docs/decisions/ADR-001-modular-monolith-and-stack.md)

---

## 6. Repository structure

```
wj-bms/
├── AGENTS.md                      # Coding agent working agreement
├── README.md                      # This file
└── docs/
    ├── README.md                  # Documentation map
    ├── PRD.md                     # Authoritative requirements
    ├── architecture/
    │   ├── ARCHITECTURE.md
    │   ├── DATA_MODEL.md
    │   └── SECURITY_AND_ACCESS_CONTROL.md
    ├── workflows/
    │   ├── PROJECT_LIFECYCLE.md
    │   ├── PROCUREMENT_AND_MATERIALS.md
    │   ├── TECHNICIAN_OPERATIONS.md
    │   └── FINANCE_AND_DOCUMENTS.md
    ├── standards/
    │   ├── DOCUMENT_NUMBERING.md
    │   └── AUDIT_LOGGING.md
    ├── implementation/
    │   ├── IMPLEMENTATION_PLAN.md
    │   ├── TESTING_STRATEGY.md
    │   └── ACCEPTANCE_CRITERIA.md
    └── decisions/
        ├── README.md
        ├── ADR-001-modular-monolith-and-stack.md
        ├── ADR-002-document-number-allocation.md
        ├── ADR-003-single-relational-database.md
        ├── ADR-004-audit-logging-approach.md
        ├── ADR-005-authorisation-model.md
        ├── ADR-006-private-file-storage.md
        └── ADR-007-spreadsheet-migration.md
```

---

## 7. Setup and development

**Pending — no application code yet.**

There is nothing to install, build, run, or configure. No `package.json`, lockfile, environment file, or source directory exists. Do not attempt to run project commands; none are defined.

When implementation is authorised, this section will be replaced with verified instructions. Until then, any setup instruction would be invented.

---

## 8. Testing

**Pending — no test framework yet.**

No test runner, no test files, and no CI configuration exist. The [testing strategy](./docs/implementation/TESTING_STRATEGY.md) defines *what* must be verified and at which level; it does not define runnable commands, because none exist.

**No test has been executed. No acceptance criterion has been verified.** The [acceptance register](./docs/implementation/ACCEPTANCE_CRITERIA.md) lists 109 defined criteria, all in `Defined` status.

---

## 9. Known limitations and open decisions

### Blocking decisions before build

| ID | Question | Impact |
|---|---|---|
| Q-NUM-01 | Final invoice numbering rules and client validation requirements (9 specific questions) | Blocks final invoice format |
| Q-PROC-01 | Purchasing approval thresholds | Blocks approval policy configuration |
| Q-FIN-01 | Required financial reports and tax obligations | Blocks any accounting scope |
| Q-DATA-01 | Existing spreadsheet structures and data quality | Blocks migration design |
| Q-OPS-01 | Hosting, backup, retention, access policy | Blocks production deployment |

Full register: [PRD §14](./docs/PRD.md).

### Known limitations of the design

1. **External numbering conflicts cannot be solved internally.** A company prefix reduces risk; it does not eliminate it. Nine client questions must be answered.
2. **The owner can do anything.** No technical control exists above the `OWNER` role; the system provides evidence, not prevention, at that level.
3. **Override makes conflicts detectable, not impossible.** In a 3-person office this is the only workable model, but it depends on the owner reviewing the override report.
4. **No delegation.** If the owner is unavailable, approvals wait.
5. **Pre-migration history is incomplete in the system.** Source spreadsheets remain the archive for earlier periods.
6. **Audit is tamper-evident, not tamper-proof.** Someone with full database access and the hashing key could rewrite the chain.

### Assumptions requiring validation

| ID | Assumption | Risk if false |
|---|---|---|
| A-01 | Technicians have smartphones with usable mobile data at sites | Field design fails; native/offline needed |
| A-02 | Indonesian-only UI is sufficient | i18n framework needed earlier |
| A-04 | Historical spreadsheet data is recoverable and usable | Migration scope grows |
| A-ROLE-02 | Technician access needs are non-uniform | Assignment scoping may be too restrictive |

---

## 10. Three properties that define this system

If nothing else from this documentation survives, these should:

1. **A document number is allocated once, atomically, on the server, and never reused.**
2. **An issued document never changes — corrections are new documents.**
3. **No state change commits without its audit record.**

Everything else is elaboration.

---

## 11. Where to start reading

| If you are… | Start here |
|---|---|
| The business owner | [PRD §2–§5](./docs/PRD.md), then [PRD §14](./docs/PRD.md) open questions |
| A developer joining | [`AGENTS.md`](./AGENTS.md), then [PRD](./docs/PRD.md), then [ARCHITECTURE](./docs/architecture/ARCHITECTURE.md) |
| Reviewing the numbering fix | [`docs/standards/DOCUMENT_NUMBERING.md`](./docs/standards/DOCUMENT_NUMBERING.md) |
| Reviewing the material logic | [`docs/workflows/PROCUREMENT_AND_MATERIALS.md`](./docs/workflows/PROCUREMENT_AND_MATERIALS.md) |
| Planning delivery | [`docs/implementation/IMPLEMENTATION_PLAN.md`](./docs/implementation/IMPLEMENTATION_PLAN.md) |
| Verifying completeness | [`docs/implementation/ACCEPTANCE_CRITERIA.md`](./docs/implementation/ACCEPTANCE_CRITERIA.md) |

---

## 12. Status of this documentation

| Aspect | Status |
|---|---|
| Language of documentation | English |
| Application UI language | **Indonesian** — proposed, pending confirmation (NFR-OPS-007) |
| Requirement IDs | 91 functional, 30 non-functional, all unique |
| Acceptance criteria | 109 defined, 0 executed |
| ADRs | 7, all `Proposed` |
| Implementation | **Not started — awaits authorisation** |
