# Documentation Map

Navigation and ownership for the WJ-CCTV Business Operations System documentation set.

**Status:** documentation phase. No application code exists. Implementation awaits authorisation.

**Version 0.2 (2026-10-25):** the technology baseline, role model, purchasing approval rule, numbering configuration, and storage model were reconciled to client requirements. Requirements were updated in place; superseded decisions are retained as ADR history.

---

## 1. Where to start

| Reader | Path |
|---|---|
| Business owner signing off | [PRD](./PRD.md) §2–§5 → §14 open questions → [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) |
| Developer joining | [`../AGENTS.md`](../AGENTS.md) → [ADR-008](./decisions/ADR-008-laravel-platform-baseline.md) → [PRD](./PRD.md) → [ARCHITECTURE](./architecture/ARCHITECTURE.md) → relevant workflow |
| Reviewing the numbering defect | [DOCUMENT_NUMBERING](./standards/DOCUMENT_NUMBERING.md) → [ADR-009](./decisions/ADR-009-mysql-document-number-allocation.md) |
| Reviewing material logic | [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) |
| Reviewing the approval rule | [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) §8 |
| Planning delivery | [IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) |

---

## 2. Diagram rendering

**All diagrams in this documentation set are Mermaid.** They are written in ` ```mermaid ` fenced blocks and **require a Mermaid-capable renderer** — for example GitHub's Markdown viewer, VS Code with a Mermaid extension, GitLab, or a docs site that bundles Mermaid.

| Diagram type | Used for | Examples |
|---|---|---|
| `stateDiagram-v2` | Project, work-order, and work-session state machines | [PROJECT_LIFECYCLE](./workflows/PROJECT_LIFECYCLE.md) §3.2, §5.2; [TECHNICIAN_OPERATIONS](./workflows/TECHNICIAN_OPERATIONS.md) §3.3 |
| `sequenceDiagram` | Procurement approval, document issuance, session correction, completion verification, WhatsApp preparation, Turbo lifecycle | [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) §3.6; [FINANCE_AND_DOCUMENTS](./workflows/FINANCE_AND_DOCUMENTS.md) §4.1; [TECHNICIAN_OPERATIONS](./workflows/TECHNICIAN_OPERATIONS.md) §3.4, §6.1, §9.1; [ARCHITECTURE](./architecture/ARCHITECTURE.md) §3.2 |
| `erDiagram` | Entity relationships | [ARCHITECTURE](./architecture/ARCHITECTURE.md) §6 |
| `flowchart` | Process flows, dependency graphs, supersession maps | [PRD](./PRD.md) §2.3; [ARCHITECTURE](./architecture/ARCHITECTURE.md) §2.2; [IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) §11; [decisions/README](./decisions/README.md) |

**Two conventions:**

1. **Transition tables are authoritative, not the diagrams.** A state diagram cannot carry actor, guard, and side-effect columns, so every state machine has a transition table beside its diagram. Where the two disagree, the table governs.
2. **Rendering has not been verified.** The Mermaid syntax in this set was checked by reading only — no renderer was available while authoring. If a diagram fails to render, the table beside it is still correct.

ASCII/box-drawing diagrams were replaced with Mermaid blocks; no ASCII art remains in normative text.

---

## 3. Document ownership

Each document owns its subject. Where two documents touch the same topic, the owner is authoritative and the other links.

| Document | Owns | Does **not** own |
|---|---|---|
| [PRD](./PRD.md) | Scope, `FR-`/`NFR-` IDs, business rules `BR-`, canonical status vocabulary | Implementation detail |
| [ARCHITECTURE](./architecture/ARCHITECTURE.md) | Module boundaries, layers, platform baseline, Turbo lifecycle, deployment | Entity definitions |
| [DATA_MODEL](./architecture/DATA_MODEL.md) | Entities, columns, MySQL types, constraints, snapshots, ledger | Workflow sequences |
| [SECURITY_AND_ACCESS_CONTROL](./architecture/SECURITY_AND_ACCESS_CONTROL.md) | Roles, permission matrix, scoping, overrides | Workflow steps |
| [PROJECT_LIFECYCLE](./workflows/PROJECT_LIFECYCLE.md) | Project and work-order state machines, closure | Material logic |
| [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) | Requirements, reservations, ledger, purchasing, approval, receipts, issue/return | Invoice issuance |
| [TECHNICIAN_OPERATIONS](./workflows/TECHNICIAN_OPERATIONS.md) | Work sessions, scheduling, daily reports, evidence, verification, WhatsApp | Work-order state machine (defined in PROJECT_LIFECYCLE) |
| [FINANCE_AND_DOCUMENTS](./workflows/FINANCE_AND_DOCUMENTS.md) | Quotations, invoices, payments, payables, corrections | Numbering mechanism |
| [DOCUMENT_NUMBERING](./standards/DOCUMENT_NUMBERING.md) | Number format, atomic allocation, uniqueness, void rules | Invoice business rules |
| [AUDIT_LOGGING](./standards/AUDIT_LOGGING.md) | Audited events, transaction coupling, immutability, retention | Business history reporting |
| [IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) | Phasing, gates, dependencies, exit criteria | Requirements |
| [TESTING_STRATEGY](./implementation/TESTING_STRATEGY.md) | Test levels, critical scenarios | Criterion definitions |
| [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) | The `AC-` register | Test procedures |
| [decisions/](./decisions/) | Architectural decisions with alternatives and supersession history | Procedures (linked) |

---

## 4. Identifier conventions

| Prefix | Meaning | Defined in | Referenced by |
|---|---|---|---|
| `FR-<AREA>-NNN` | Functional requirement | [PRD](./PRD.md) §6 | Workflows, plan, AC |
| `NFR-<AREA>-NNN` | Non-functional requirement | [PRD](./PRD.md) §7 | Architecture, plan, AC |
| `BR-<AREA>-NNN` | Business rule | [PRD](./PRD.md) §8 | Workflows |
| `AC-<AREA>-NNN` | Acceptance criterion | [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) | PRD requirement tables |
| `ADR-NNN` | Architecture decision | [decisions/](./decisions/) | Architecture, standards |
| `Q-<AREA>-NN` | Open question | [PRD](./PRD.md) §14 | Wherever relevant |
| `A-NN` | Assumption | [PRD](./PRD.md) §13.1 | Wherever relevant |
| `M1`–`M12` | Measurable outcome | [PRD](./PRD.md) §3.2 | Plan, testing |

Areas: `CUST`, `PROJ`, `PROC`, `MATL`, `TECH`, `FIN`, `DOC`, `RPT`, `DATA`, `SEC`, `AUD`, `OPS`, `PLAT`, `SYS`, `WA`, `MAP`, `PWA`.

**IDs are never reused.** A withdrawn requirement keeps its number and is marked `WITHDRAWN`. A changed requirement is updated in place; if its meaning changes materially, the change is recorded in the document's revision note.

---

## 5. Classification labels

| Label | Meaning |
|---|---|
| `[Confirmed]` | Stated in the client brief; treat as fact |
| `[Proposed]` | Our recommendation, awaiting sign-off |
| `[Assumption]` | Believed true, not verified; must be validated |
| `[Open Question]` | Unresolved; owner and decision trigger named |
| `[Out of Scope]` | Explicitly excluded, with rationale |

An assumption is never silently promoted to a requirement.

**Two labels worth calling out**, because they carry the most risk in this set:

- **`[Confirmed]` minimum version ≥ 13.29.0** — the client's requirement. The **CVE identity and whether that version fixes it are `[Assumption]` / `[Open Question]`**, unverified offline (Q-SEC-01).
- **`[Confirmed]` TailAdmin is the UI foundation** — its **compatibility and licence are `[Open Question]`** (Q-UI-01), gated at Phase 1.

---

## 6. Reading paths by subject

### The numbering problem

[PRD §11](./PRD.md) → [DOCUMENT_NUMBERING](./standards/DOCUMENT_NUMBERING.md) → [ADR-009](./decisions/ADR-009-mysql-document-number-allocation.md) → [FINANCE_AND_DOCUMENTS §8](./workflows/FINANCE_AND_DOCUMENTS.md) → [TESTING_STRATEGY §4](./implementation/TESTING_STRATEGY.md)

### Materials and purchasing

[PRD §6.3–6.4](./PRD.md) → [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) → [DATA_MODEL §5–6](./architecture/DATA_MODEL.md) → [TESTING_STRATEGY §8–9](./implementation/TESTING_STRATEGY.md)

### Field operations and work sessions

[PRD §6.5](./PRD.md) → [TECHNICIAN_OPERATIONS](./workflows/TECHNICIAN_OPERATIONS.md) → [PROJECT_LIFECYCLE §5](./workflows/PROJECT_LIFECYCLE.md) → [TESTING_STRATEGY §10](./implementation/TESTING_STRATEGY.md)

### Security, roles, and audit

[SECURITY_AND_ACCESS_CONTROL](./architecture/SECURITY_AND_ACCESS_CONTROL.md) → [AUDIT_LOGGING](./standards/AUDIT_LOGGING.md) → [ADR-004](./decisions/ADR-004-audit-logging-approach.md), [ADR-005](./decisions/ADR-005-authorisation-model.md)

### Platform and compatibility gates

[ADR-008](./decisions/ADR-008-laravel-platform-baseline.md) → [ARCHITECTURE §3](./architecture/ARCHITECTURE.md) → [IMPLEMENTATION_PLAN §4.0](./implementation/IMPLEMENTATION_PLAN.md) → [TESTING_STRATEGY §14](./implementation/TESTING_STRATEGY.md)

### Settings, WhatsApp, maps, PWA

[PRD §6.10–6.13](./PRD.md) → [ARCHITECTURE §3.2, §7, §9.1](./architecture/ARCHITECTURE.md) → [TECHNICIAN_OPERATIONS §9–10](./workflows/TECHNICIAN_OPERATIONS.md) → [TESTING_STRATEGY §12–14](./implementation/TESTING_STRATEGY.md)

### Delivery planning

[IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) → [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) → [TESTING_STRATEGY](./implementation/TESTING_STRATEGY.md)

---

## 7. Consistency rules

These rules keep the set coherent and are enforced during review:

1. **One owner per subject.** No duplicate requirement blocks; cross-reference instead.
2. **Status names are verbatim.** Defined once in [PRD §9](./PRD.md), reused unchanged everywhere.
3. **All relative links resolve.** No link to a file that does not exist.
4. **Requirement IDs are unique and stable.** Duplicates are a defect.
5. **Every `AC-` reference points at a real entry** in [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md).
6. **Labels are used consistently** (§5).
7. **No empty sections, no placeholder-only content, no untestable language.**
8. **Repository facts are measured, not invented.** No branch, SHA, or remote claim without command output.
9. **Diagrams are Mermaid, and rendering is never claimed without rendering** (§2).
10. **Superseded decisions keep their body.** Status changes; history stays.

---

## 8. Maintaining this set

| Trigger | Action |
|---|---|
| New requirement | Add `FR-`/`NFR-` to PRD, add `AC-` entries, update the owning workflow |
| Changed requirement | Update in place; never renumber |
| New status value | Update PRD §9 and every referencing document |
| Architectural decision | New ADR |
| Superseding a decision | New ADR; old status → `Superseded by ADR-NNN`; body retained with a note |
| Resolved open question | Update PRD §14 with the resolution, keep the history row, update dependents |
| New open question | Add to PRD §14 with a proposed default and a decision trigger |

---

## 9. Current state

| Aspect | Value |
|---|---|
| Documents | 25 Markdown files (23 under `docs/`, plus root `README.md` and `AGENTS.md`) |
| Functional requirements | 96 (`FR-`), all unique |
| Non-functional requirements | 39 (`NFR-`), all unique |
| Business rules | 34 (`BR-`), all unique |
| Acceptance criteria | 141 (`AC-`), all unique and `Defined`, none executed |
| Architecture decisions | 9 — 2 superseded (history retained), 6 accepted or amended, 1 verified unchanged |
| Open questions | 27 in the PRD register, plus ADR-level questions |
| Mermaid diagrams | 21, syntax checked by reading only — **not rendered** |
| Application code | None |
| Implementation | Not started |

### 9.1 Document size, stated honestly

| Measure | Words |
|---|---|
| Baseline before this reconciliation (24 files) | 60,721 |
| Original 24 files, as reconciled | 69,775 — **+14.9%**, inside the brief's "+15%" guideline |
| Two new ADRs required by the brief (ADR-008 1,735 + ADR-009 1,690) | +3,425 |
| **Total now (26 files)** | **73,200 — +20.5%** |

The overage is **entirely** the two architecture decision records the reconciliation required: the platform baseline (ADR-008) and the MySQL allocation mechanism (ADR-009). No business rule, entity relationship, constraint, authorisation requirement, invariant, or implementation-critical detail was removed to hit a word target. Where content was cut, it was a fourth copy of a statement whose canonical home is elsewhere (e.g. `DATA_MODEL` §1.5 now points at ADR-003 for reasoning; `ADR-009` points at `DOCUMENT_NUMBERING` §5.1 for the SQL statement).
