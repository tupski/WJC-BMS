# Documentation Map

Navigation and ownership for the WJ-CCTV Business Operations System documentation set.

**Status:** documentation phase. No application code exists. Implementation awaits authorisation.

---

## 1. Where to start

| Reader | Path |
|---|---|
| Business owner signing off | [PRD](./PRD.md) §2–§5 → §14 open questions → [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) |
| Developer joining | [`../AGENTS.md`](../AGENTS.md) → [PRD](./PRD.md) → [ARCHITECTURE](./architecture/ARCHITECTURE.md) → relevant workflow |
| Reviewing the numbering defect | [DOCUMENT_NUMBERING](./standards/DOCUMENT_NUMBERING.md) → [ADR-002](./decisions/ADR-002-document-number-allocation.md) |
| Reviewing material logic | [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) |
| Planning delivery | [IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) |

---

## 2. Document ownership

Each document owns its subject. Where two documents touch the same topic, the owner is authoritative and the other links.

| Document | Owns | Does **not** own |
|---|---|---|
| [PRD](./PRD.md) | Scope, `FR-`/`NFR-` IDs, business rules `BR-`, canonical status vocabulary | Implementation detail |
| [ARCHITECTURE](./architecture/ARCHITECTURE.md) | Module boundaries, layers, technology position, deployment | Entity definitions |
| [DATA_MODEL](./architecture/DATA_MODEL.md) | Entities, columns, constraints, snapshots, ledger | Workflow sequences |
| [SECURITY_AND_ACCESS_CONTROL](./architecture/SECURITY_AND_ACCESS_CONTROL.md) | Roles, permission matrix, scoping, overrides | Workflow steps |
| [PROJECT_LIFECYCLE](./workflows/PROJECT_LIFECYCLE.md) | Project and work-order state machines, closure | Material logic |
| [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) | Requirements, reservations, ledger, purchasing, receipts, issue/return | Invoice issuance |
| [TECHNICIAN_OPERATIONS](./workflows/TECHNICIAN_OPERATIONS.md) | Scheduling, daily reports, evidence, verification | Work-order state machine (defined in PROJECT_LIFECYCLE) |
| [FINANCE_AND_DOCUMENTS](./workflows/FINANCE_AND_DOCUMENTS.md) | Quotations, invoices, payments, payables, corrections | Numbering mechanism |
| [DOCUMENT_NUMBERING](./standards/DOCUMENT_NUMBERING.md) | Number format, atomic allocation, uniqueness, void rules | Invoice business rules |
| [AUDIT_LOGGING](./standards/AUDIT_LOGGING.md) | Audited events, transaction coupling, immutability, retention | Business history reporting |
| [IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) | Phasing, dependencies, exit criteria | Requirements |
| [TESTING_STRATEGY](./implementation/TESTING_STRATEGY.md) | Test levels, critical scenarios | Criterion definitions |
| [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) | The `AC-` register | Test procedures |
| [decisions/](./decisions/) | Architectural decisions with alternatives | Procedures (linked) |

---

## 3. Identifier conventions

| Prefix | Meaning | Defined in | Referenced by |
|---|---|---|---|
| `FR-<AREA>-NNN` | Functional requirement | [PRD](./PRD.md) §6 | Workflows, plan, AC |
| `NFR-<AREA>-NNN` | Non-functional requirement | [PRD](./PRD.md) §7 | Architecture, plan, AC |
| `BR-<AREA>-NNN` | Business rule | [PRD](./PRD.md) §8 | Workflows |
| `AC-<AREA>-NNN` | Acceptance criterion | [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) | PRD requirement tables |
| `ADR-NNN` | Architecture decision | [decisions/](./decisions/) | Architecture, standards |
| `Q-<AREA>-NN` | Open question | [PRD](./PRD.md) §14 | Wherever relevant |
| `A-NN` | Assumption | [PRD](./PRD.md) §13.1 | Wherever relevant |
| `M1`–`M9` | Measurable outcome | [PRD](./PRD.md) §3.2 | Plan, testing |

Areas: `CUST`, `PROJ`, `PROC`, `MATL`, `TECH`, `FIN`, `DOC`, `RPT`, `DATA`, `SEC`, `AUD`, `OPS`.

**IDs are never reused.** A withdrawn requirement keeps its number and is marked `WITHDRAWN`.

---

## 4. Classification labels

Used consistently wherever a claim appears:

| Label | Meaning |
|---|---|
| `[Confirmed]` | Stated in the client brief; treat as fact |
| `[Proposed]` | Our recommendation, awaiting sign-off |
| `[Assumption]` | Believed true, not verified; must be validated |
| `[Open Question]` | Unresolved; owner and decision trigger named |
| `[Out of Scope]` | Explicitly excluded, with rationale |

An assumption is never silently promoted to a requirement.

---

## 5. Reading paths by subject

### The numbering problem

[PRD §11](./PRD.md) → [DOCUMENT_NUMBERING](./standards/DOCUMENT_NUMBERING.md) → [ADR-002](./decisions/ADR-002-document-number-allocation.md) → [FINANCE_AND_DOCUMENTS §8](./workflows/FINANCE_AND_DOCUMENTS.md) → [TESTING_STRATEGY §4](./implementation/TESTING_STRATEGY.md)

### Materials and purchasing

[PRD §6.3–6.4](./PRD.md) → [PROCUREMENT_AND_MATERIALS](./workflows/PROCUREMENT_AND_MATERIALS.md) → [DATA_MODEL §5–6](./architecture/DATA_MODEL.md) → [TESTING_STRATEGY §8–9](./implementation/TESTING_STRATEGY.md)

### Field operations

[PRD §6.5](./PRD.md) → [TECHNICIAN_OPERATIONS](./workflows/TECHNICIAN_OPERATIONS.md) → [PROJECT_LIFECYCLE §5](./workflows/PROJECT_LIFECYCLE.md) → [TESTING_STRATEGY §10](./implementation/TESTING_STRATEGY.md)

### Security and audit

[SECURITY_AND_ACCESS_CONTROL](./architecture/SECURITY_AND_ACCESS_CONTROL.md) → [AUDIT_LOGGING](./standards/AUDIT_LOGGING.md) → [ADR-004](./decisions/ADR-004-audit-logging-approach.md), [ADR-005](./decisions/ADR-005-authorisation-model.md)

### Delivery planning

[IMPLEMENTATION_PLAN](./implementation/IMPLEMENTATION_PLAN.md) → [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md) → [TESTING_STRATEGY](./implementation/TESTING_STRATEGY.md)

---

## 6. Consistency rules

These rules keep the set coherent and are enforced during review:

1. **One owner per subject.** No duplicate requirement blocks; cross-reference instead.
2. **Status names are verbatim.** Defined once in [PRD §9](./PRD.md), reused unchanged everywhere.
3. **All relative links resolve.** No link to a file that does not exist.
4. **Requirement IDs are unique and stable.** Duplicates are a defect.
5. **Every `AC-` reference points at a real entry** in [ACCEPTANCE_CRITERIA](./implementation/ACCEPTANCE_CRITERIA.md).
6. **Labels are used consistently** (§4).
7. **No empty sections, no placeholder-only content, no untestable language.**
8. **Repository facts are not invented.** There is no code, so no document claims behaviour that exists.

---

## 7. Maintaining this set

| Trigger | Action |
|---|---|
| New requirement | Add `FR-`/`NFR-` to PRD, add `AC-` entries, update owning workflow |
| Changed requirement | Update in place; never renumber |
| New status value | Update PRD §9 and every referencing document |
| Architectural decision | New ADR |
| Resolved open question | Update PRD §14, affected ADR, and dependent documents |
| New open question | Add to PRD §14 with proposed default and decision trigger |

---

## 8. Current state

| Aspect | Value |
|---|---|
| Documents | 24 Markdown files (22 under `docs/`, plus root `README.md` and `AGENTS.md`) |
| Functional requirements | 91 (`FR-`), all unique |
| Non-functional requirements | 30 (`NFR-`), all unique |
| Business rules | 27 (`BR-`), all unique |
| Acceptance criteria | 109 (`AC-`), all unique and `Defined`, none executed |
| Architecture decisions | 7, all `Proposed` |
| Open questions | 11 in the PRD register, plus ADR-level questions |
| Application code | None |
| Implementation | Not started |
