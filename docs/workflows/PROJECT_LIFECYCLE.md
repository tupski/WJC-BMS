# Project Lifecycle — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed |
| Version | 0.1 |
| Scope owner | This document owns the **project status** and **work-order status** state machines, project creation, scheduling, and closure |
| Requirement source | [`../PRD.md`](../PRD.md) §6.2, §9.1, §9.2, §8 |
| Related | [`./TECHNICIAN_OPERATIONS.md`](./TECHNICIAN_OPERATIONS.md), [`./PROCUREMENT_AND_MATERIALS.md`](./PROCUREMENT_AND_MATERIALS.md), [`./FINANCE_AND_DOCUMENTS.md`](./FINANCE_AND_DOCUMENTS.md) |

---

## 1. Core principle: three independent state machines

**[Confirmed]** Projects span multiple days and involve one or many technicians. This makes a binary completed/not-completed model unusable. The system therefore maintains three separate state machines that are **never collapsed into one another** (BR-PROJ-002):

| # | State machine | Owner document | Answers |
|---|---|---|---|
| 1 | **Project status** | This document §3 | Where is the engagement as a business commitment? |
| 2 | **Work-order status** | This document §5 (detail in [`./TECHNICIAN_OPERATIONS.md`](./TECHNICIAN_OPERATIONS.md)) | Where is the individual technician's task? |
| 3 | **Material readiness status** | [`./PROCUREMENT_AND_MATERIALS.md`](./PROCUREMENT_AND_MATERIALS.md) §7 | Can the work physically proceed? |

**Why they must stay separate:** a project can be `IN_PROGRESS` with materials `AWAITING_PROCUREMENT` (work started on the accessible portion); a project can have all work orders `VERIFIED` while materials remain `PARTIALLY_ISSUED` (leftover material not yet returned); material readiness can be `READY_TO_START` for a project still in `SCHEDULED` (everything staged, nobody on site yet). Any design that infers one from another produces false states.

**The only sanctioned coupling** is BR-PROJ-003: entering `IN_PROGRESS` while readiness is not `READY_TO_START` requires a recorded override with reason. This is deliberately narrow.

---

## 2. Project creation

### 2.1 Required at creation

| Field | Required | Notes |
|---|---|---|
| Customer | Yes | FR-PROJ-001 |
| Service category | Yes | FR-PROJ-002; drives no CCTV assumptions |
| Title | Yes | Human description of the work |
| Site location | Yes | Where technicians must go |
| Planned start date | Yes | May be revised later |
| Target completion date | Yes | Must be ≥ planned start |
| Owner of record (office user) | Yes | Accountability for the project |

### 2.2 Optional at creation — and stays optional

| Field | Optional because |
|---|---|
| Quotation reference | FR-PROJ-003 — many small jobs never get a formal quotation |
| Customer PO reference | FR-PROJ-004 — many customers issue no PO |
| Budget (revenue, material cost) | FR-PROJ-008 — the business may not budget small jobs |
| Material requirement lines | Some services require no materials |
| Technician assignments | Assigned after scoping |
| Target schedule detail | Known later for multi-day work |

**BR-PROJ-001 is the binding rule:** no step may be forced merely because the system supports it. A one-day, no-material, single-technician job must be creatable, executable, billable, and closable without ever touching quotation, customer PO, purchase request, or material screens. Acceptance test AC-PROJ-001 covers this explicitly.

### 2.3 Creation sequence

```
[Office] Create project (DRAFT)
   ├─ attach customer + service category
   ├─ optionally link quotation / customer PO
   └─ optionally record budget
        ↓
[Office] Define scope of work (description, site notes)
        ↓
[Office] (optional) Record material requirements  → see PROCUREMENT_AND_MATERIALS.md §3
        ↓
[Office] Confirm commercial basis → transition to PLANNED
```

`DRAFT` is intentionally permissive: it is a working area. Validation that matters (dates, category, customer) applies at `PLANNED`.

---

## 3. Project status state machine

Statuses are defined in [`../PRD.md`](../PRD.md) §9.1 and reused verbatim.

| From | To | Trigger / condition | Actor | Side effects |
|---|---|---|---|---|
| `DRAFT` | `PLANNED` | Required fields complete; commercial basis confirmed | `OPS_ADMIN`, `OWNER` | Project becomes visible in operational lists |
| `PLANNED` | `SCHEDULED` | ≥1 technician assigned AND planned dates set | `OPS_ADMIN`, `OWNER` | Assignment records created; technician sees project |
| `SCHEDULED` | `IN_PROGRESS` | Work has begun on site | `OPS_ADMIN`, `OWNER`, assigned `TEAM_LEADER` | `actual_start_date` set; readiness override required if not `READY_TO_START` |
| `IN_PROGRESS` | `ON_HOLD` | Blocker requiring suspension | `OPS_ADMIN`, `OWNER`, `TEAM_LEADER` | Blocker reason recorded (FR-TECH-006); open work orders → `BLOCKED` |
| `ON_HOLD` | `IN_PROGRESS` | Blocker resolved | `OPS_ADMIN`, `OWNER` | Blocker closure note recorded |
| `IN_PROGRESS` | `PENDING_COMPLETION_VERIFICATION` | All work orders `VERIFIED` or `CANCELLED` | **System-derived** | Not manually settable |
| `PENDING_COMPLETION_VERIFICATION` | `IN_PROGRESS` | Verification rejected / rework required | `OPS_ADMIN`, `OWNER` | Reason required; work order returned to `IN_PROGRESS` |
| `PENDING_COMPLETION_VERIFICATION` | `COMPLETED` | Office verifies completion | `OPS_ADMIN`, `OWNER` | `actual_completion_date` set; billing may proceed |
| `COMPLETED` | `CLOSED` | All closure conditions met (§6) | `OPS_ADMIN`, `OWNER` | Project becomes read-only except corrections |
| `DRAFT`/`PLANNED`/`SCHEDULED`/`IN_PROGRESS`/`ON_HOLD` | `CANCELLED` | Abandoned with reason | `OWNER`, `OPS_ADMIN` | Reason required; records retained (BR-PROJ-005) |
| `PENDING_COMPLETION_VERIFICATION`/`COMPLETED` | `CANCELLED` | **Not permitted** | — | Work already performed; use `CLOSED` with a note, or credit the invoice |

### 3.1 Transition diagram

```
                    ┌─────────┐
                    │  DRAFT  │
                    └────┬────┘
                         │ confirm scope
                    ┌────▼─────┐
                    │ PLANNED  │──── cancel ──┐
                    └────┬─────┘               │
                         │ assign + schedule   │
                   ┌─────▼──────┐               │
                   │ SCHEDULED  │──── cancel ──┤
                   └─────┬──────┘               │
                         │ work begins          │
                   ┌─────▼───────┐              │
        ┌─────────►│ IN_PROGRESS │──── cancel ──┤
        │          └──┬───────┬──┘              │
        │  blocker     │       │ all WOs verified│
        │  resolved    │       ▼                 │
   ┌────▼────┐    ┌────▼──────▼──────────┐       │
   │ ON_HOLD │    │ PENDING_COMPLETION_  │       │
   └────┬────┘    │ VERIFICATION         │       │
        │         └────┬─────────┬───────┘       │
        │              │ verify  │ rework        │
        │         ┌────▼────┐    └──► IN_PROGRESS│
        │         │COMPLETED│                    │
        │         └────┬────┘                    │
        │              │ closure conditions      │
        │         ┌────▼───┐                ┌────▼─────┐
        └────────►│ CLOSED │                │CANCELLED │
                  └────────┘                └──────────┘
```

### 3.2 Rejected transitions

The system must refuse, with the current status and the permitted next statuses named in the message (FR-PROJ-011):

- `DRAFT` → `IN_PROGRESS` (skipping planning and scheduling)
- `SCHEDULED` → `COMPLETED` (skipping execution and verification)
- `COMPLETED` → `IN_PROGRESS` (reopening completed work; use a new project or an explicit recorded correction)
- `CLOSED` → anything (closed is terminal)
- `CANCELLED` → anything (cancelled is terminal)
- Any manual set of `PENDING_COMPLETION_VERIFICATION`

**[Open Question] Q-PROJ-01 — Reopening a closed project.** Real businesses reopen closed jobs (a customer reports a fault within warranty). **Proposed default:** do not reopen; create a new project linked to the original with `related_project_id`, preserving the closed project's financial history. Rationale: reopening invalidates already-issued invoices and closed-period reporting. **Decision trigger:** Phase 0 — does the owner need to reopen, or is a linked follow-up project acceptable?

---

## 4. Scheduling and assignment

### 4.1 Assignment model

- A project may have **one or many** technicians (FR-PROJ-007). Both extremes are normal.
- At most **one** technician may be designated `TEAM_LEADER` for a project at a time — enforced by a partial unique index, not a UI rule.
- Assignment records are append-only in effect: unassigning sets `unassigned_at` and a reason, it never deletes the row.

### 4.2 Multi-day scheduling

| Concept | Model |
|---|---|
| Project planned start / target completion | Dates on the project |
| Per-day or per-phase detail | Optional schedule entries (date, technician, planned activity) |
| Actual work days | Derived from submitted daily reports (FR-TECH-004) |
| Schedule variance | Derived: actual vs target completion |

A project spanning five days does not create five projects, five invoices, or five work orders. It creates one project, one work order per technician (or per phase where a technician's work has distinct stages), and one daily report per technician per working day.

### 4.3 Revision

Changing planned start or target completion creates a `project_schedule_revisions` row (FR-PROJ-005) capturing previous values, new values, reason, actor, and timestamp. Nothing is overwritten. Reports can show "target moved twice, both with reasons".

---

## 5. Work-order status

Detail of the field workflow is in [`./TECHNICIAN_OPERATIONS.md`](./TECHNICIAN_OPERATIONS.md). The state machine is owned here because it interacts with project status.

| From | To | Condition | Actor |
|---|---|---|---|
| `DRAFT` | `ASSIGNED` | Technician assigned | `OPS_ADMIN`, `OWNER` |
| `ASSIGNED` | `ACCEPTED` | Technician acknowledges | Assigned technician |
| `ASSIGNED`/`ACCEPTED` | `IN_PROGRESS` | First daily report submitted or explicit start | Assigned technician |
| `IN_PROGRESS` | `BLOCKED` | Blocker recorded with reason code | Assigned technician, `TEAM_LEADER` |
| `BLOCKED` | `IN_PROGRESS` | Blocker resolved | Assigned technician, `OPS_ADMIN` |
| `IN_PROGRESS` | `COMPLETED_PENDING_VERIFICATION` | Technician submits completion | Assigned technician |
| `COMPLETED_PENDING_VERIFICATION` | `VERIFIED` | Office accepts completion | `OPS_ADMIN`, `OWNER`, project `TEAM_LEADER` |
| `COMPLETED_PENDING_VERIFICATION` | `IN_PROGRESS` | Rejected, rework required | Verifier, with reason |
| Any non-terminal | `CANCELLED` | Withdrawn with reason | `OPS_ADMIN`, `OWNER` |

**Terminal states:** `VERIFIED`, `CANCELLED`.

**BR-TECH-002:** the verifier may not be the technician who completed the work, unless an override with reason is recorded. In a 3-person office the verifier is usually `OPS_ADMIN`; on site it may be the `TEAM_LEADER`, which is acceptable because they are a different person from the completing technician.

---

## 6. Closure

### 6.1 Closure conditions (BR-PROJ-004)

All must hold; the system enumerates any that fail (FR-PROJ-010):

| # | Condition | Rationale |
|---|---|---|
| 1 | Every work order is `VERIFIED` or `CANCELLED` | No work in limbo |
| 2 | No material line is in an open issued state (all issued quantity consumed, returned, or written off) | Prevents untracked material loss |
| 3 | No invoice is `DRAFT` | Billing decided |
| 4 | Every issued invoice is `PAID`, `VOID`, or `CREDITED` — or explicitly accepted as outstanding with a note | Closure is about work completion, not payment; outstanding AR is allowed but must be acknowledged |
| 5 | Any closure override is recorded with reason | Auditability |

**Condition 4 is deliberately permissive.** Requiring full payment before closure would block projects for months over a slow-paying customer and push staff to fake payments. The system instead allows closure with acknowledged outstanding AR, which then appears in the receivables ageing report (FR-FIN-005).

### 6.2 Closure sequence

```
[System] Evaluate closure conditions
   ├─ unmet → refuse, list each unmet condition with a link
   └─ met → [Office] Confirm closure
                 ↓
            [System] status = CLOSED
                     • project becomes read-only
                     • remaining material reconciled (FR-MATL-013)
                     • final cost snapshot recorded
                     • audit event written
```

### 6.3 After closure

| Action | Permitted? |
|---|---|
| Edit project scope, dates, assignment | No |
| Add daily report | No |
| Issue an invoice for unbilled work | **Yes** — closure is about work, not billing completeness |
| Record a payment against an existing invoice | Yes |
| Issue a credit note | Yes |
| Correct a material movement (compensating entry) | Yes, with audit |
| Delete anything | Never |

**[Open Question] Q-PROJ-02 — Should closure block on unbilled work?** **Proposed default:** no. Warn if work orders exist with no corresponding invoice line, but allow closure. Rationale: forcing billing completeness at closure creates pressure to issue invoices prematurely; a warning plus a monthly unbilled-work report is a better control.

---

## 7. Cancellation

Cancelling a project is a business decision with financial consequences, so it is heavily constrained:

1. Requires a reason code and free-text reason (FR-PROJ-012).
2. Permitted only before work verification (§3).
3. Does **not** delete the project, its documents, its purchase orders, its payables, or its invoices (BR-PROJ-005).
4. Open work orders are cancelled with the same reason.
5. Reserved materials are released (not silently consumed).
6. Issued materials must still be returned or written off — cancellation does not make physical material disappear.
7. Outstanding payables to suppliers remain payable. Cancelling a project never cancels a supplier debt.
8. An audit event is written, and the project remains visible in reports marked `CANCELLED`.

**Common mistake this prevents:** treating project cancellation as a rollback. It is not. Money spent and material issued remain facts.

---

## 8. Interaction with materials and finance

| Event | Material effect | Finance effect |
|---|---|---|
| `PLANNED` | Requirements may be recorded | Quotation may be issued |
| `SCHEDULED` | Reservations may be created; purchasing may proceed | — |
| `IN_PROGRESS` | Materials issued as needed | — |
| `PENDING_COMPLETION_VERIFICATION` | Remaining material returned or written off | Invoice may be issued |
| `COMPLETED` | Reconciliation runs | Invoices issued, payments recorded |
| `CLOSED` | Read-only | Payments continue; corrections only |
| `CANCELLED` | Reservations released; issued material must be resolved | Outstanding invoices/payables persist |

**No step in this table is automatic across domains.** Issuing an invoice does not change project status; closing a project does not mark an invoice paid. Each domain acts on its own state machine, and the UI presents them side by side.

---

## 9. Reporting derived from lifecycle

| Report | Source |
|---|---|
| Projects by status | Project status + counts |
| Schedule variance | Target vs actual completion |
| Blocked work orders | Work-order status = `BLOCKED` |
| Projects awaiting verification | Status = `PENDING_COMPLETION_VERIFICATION` |
| Unbilled completed work | `COMPLETED` projects with work orders lacking invoice coverage |
| Cancelled project impact | Cancelled projects with spend or issued material |

---

## 10. Acceptance criteria references

| Behaviour | AC |
|---|---|
| Project creatable without quotation/PO/materials/multiple technicians | AC-PROJ-001 |
| Service category drives no CCTV assumptions | AC-PROJ-002 |
| Optional links truly optional | AC-PROJ-003, AC-PROJ-004 |
| Schedule revision recorded, not overwritten | AC-PROJ-005 |
| Multi-day progress not binary | AC-PROJ-006 |
| One-or-many technician assignment; single team leader | AC-PROJ-007 |
| Closure refused with enumerated unmet conditions | AC-PROJ-010 |
| Illegal transitions refused with permitted options named | AC-PROJ-011 |
| Cancellation retains all records | AC-PROJ-012 |
| Verification by another user, override when not | AC-TECH-008, AC-AUD-004 |

---

## 11. Cross-references

- Status definitions: [`../PRD.md`](../PRD.md) §9.1, §9.2
- Field execution detail: [`./TECHNICIAN_OPERATIONS.md`](./TECHNICIAN_OPERATIONS.md)
- Material readiness: [`./PROCUREMENT_AND_MATERIALS.md`](./PROCUREMENT_AND_MATERIALS.md)
- Billing after completion: [`./FINANCE_AND_DOCUMENTS.md`](./FINANCE_AND_DOCUMENTS.md)
- Who may transition: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) §5
