# ADR-001 — Modular monolith architecture and technology stack

| Field | Value |
|---|---|
| Status | **Proposed** — not ratified, not installed |
| Date | 2026-10-25 |
| Deciders | Business owner, technical lead (both to be confirmed) |
| Supersedes | — |
| Related | ADR-003 (single database), [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) |

---

## Context

The workspace was verified empty: no code, no stack, no build or test commands, no conventions. There is no existing technology to remain consistent with, which means the choice is open — and also means there is no evidence to justify an unusual choice.

Confirmed operating context:

- ~3 office employees, ~15 technicians.
- Projects span multiple days; teams range from one to several technicians.
- Administration currently runs on spreadsheets, documents, and WhatsApp.
- The highest-severity known defect is a document-numbering conflict.
- No centralised inventory exists.

The two requirements that most constrain the architecture are:

1. **FR-DOC-004 / NFR-DATA-002** — atomic, unique document-number allocation under concurrency.
2. **FR-MATL-006** — reservations across projects must never exceed verified stock.

Both are transactional-correctness problems. Both are trivial inside a single ACID transaction and materially harder across process or service boundaries.

---

## Decision

Adopt a **modular monolith**: one deployable application, one relational database, one private object store, with enforced module boundaries.

Technology recommendation:

| Layer | Recommendation |
|---|---|
| Language | TypeScript, end to end |
| Runtime | Node.js (LTS) |
| Database | PostgreSQL |
| Data access | Typed ORM with explicit transaction control and raw-SQL escape hatch |
| UI | Server-rendered pages, mobile-first responsive, progressive enhancement |
| File storage | S3-compatible private object storage |
| PDF | Server-side HTML-to-PDF rendering |
| Deployment | Single-region, containerised, three environments |

**This is a proposal.** Nothing has been installed. No `package.json`, lockfile, configuration, or source directory exists in this repository.

---

## Consequences

### Positive

- **Transactional correctness is native.** Number allocation, reservation checks, and audit coupling are ordinary transactions. No sagas, no outbox, no distributed compensation.
- **One deployment artefact.** A small team can operate it without a platform engineer.
- **One database to back up and restore.** Recovery is a single, testable procedure.
- **Refactoring is cheap.** Module boundaries can move without network contracts.
- **Debuggability.** A single request trace covers the whole operation.

### Negative

- **Boundaries erode without enforcement.** A modular monolith degrades into a monolith with folders unless a dependency check runs in CI.
- **One deploy unit** means a change anywhere redeploys everything. Acceptable at this scale.
- **Scaling is vertical first.** Acceptable for ~18 users.
- **TypeScript ecosystem churn** requires disciplined dependency choices.
- **Server-rendered UI** makes rich interactions harder than an SPA would; mitigated by progressive enhancement where genuinely needed.

### Neutral

- Extraction to services remains possible later if a module proves genuinely independent. It is not planned.

---

## Alternatives considered

### Alternative A — Microservices

**Rejected.** No operational capacity for service discovery, distributed tracing, and multiple deploys. Critically, it converts the two highest-risk correctness problems into distributed-systems problems. A team of this size choosing microservices would spend its capacity on infrastructure rather than on the business problem.

### Alternative B — Laravel (PHP) monolith

**Viable.** Batteries-included, excellent for CRUD, strong PDF and templating support, widely available locally. Rejected primarily because end-to-end static typing across module boundaries makes the boundary enforcement this design depends on materially easier to verify. If the delivery team is already fluent in Laravel, this is a legitimate reversal — the architectural decision (monolith) matters more than the language.

### Alternative C — Django (Python) monolith

**Viable.** Excellent ORM, mature admin, good transaction control. Rejected on weaker compile-time contract enforcement and moderate local hiring depth relative to the alternatives.

### Alternative D — SPA frontend + separate API

**Rejected for MVP.** Doubles the surface area, complicates mobile performance on weak networks, and provides no benefit for form-driven workflows. Server-rendered pages with targeted enhancement meet the technician UX target.

### Alternative E — Event-sourced / CQRS

**Rejected.** The audit log already provides history. Event sourcing would add reconstruction complexity and eventual consistency for no business gain, and would make the numbering requirement harder rather than easier.

---

## Open questions

| ID | Question | Impact |
|---|---|---|
| Q-STACK-01 | Is the delivery team already fluent in one of the alternatives? | A team's existing fluency outweighs a marginal technical preference |
| Q-STACK-02 | Is there an organisational preference for a specific cloud provider? | Affects managed database and object storage choice |
| Q-STACK-03 | Is server-rendered UI acceptable to the owner, given no SPA-style interactions? | Affects UX expectations |

**Ratification trigger:** Phase 0 completion, when the delivery team is known. Until then, this ADR is a proposal and must not be treated as an installed decision.
