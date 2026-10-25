# AGENTS.md — Working Agreement for Coding Agents

This file governs how automated coding agents work in this repository. It is intentionally short and imperative. It does not restate requirements; it points at the authoritative sources.

---

## 1. Current repository state

**This repository contains documentation only.**

Verified on 2026-10-25: the workspace was empty before authoring — zero entries including hidden files, no `.git`, no `AGENTS.md`, no `docs/`. There is:

- **No application code.**
- **No technology stack installed.**
- **No `package.json`, lockfile, `tsconfig.json`, or configuration file.**
- **No build, lint, test, or run commands.**

Do not invent any of the above. Do not state that a command works, that a test passes, or that a framework is in use. If you need a command that does not exist, say so explicitly.

The recommended stack is a **proposal** ([`docs/decisions/ADR-001-modular-monolith-and-stack.md`](./docs/decisions/ADR-001-modular-monolith-and-stack.md)). It has not been installed and is not binding until ratified.

---

## 2. Authoritative documents and reading order

[`docs/PRD.md`](./docs/PRD.md) is the **single source of truth** for scope and functional requirements. It owns:

- All `FR-` and `NFR-` identifiers.
- All business rules (`BR-`).
- The canonical status vocabulary (PRD §9).

If any other document contradicts the PRD, **the PRD wins** and the contradiction must be raised, not silently resolved.

### Required reading order

| Order | Document | Why |
|---|---|---|
| 1 | [`docs/PRD.md`](./docs/PRD.md) | Scope, requirements, statuses, business rules |
| 2 | [`docs/architecture/ARCHITECTURE.md`](./docs/architecture/ARCHITECTURE.md) | Module boundaries and technology position |
| 3 | [`docs/architecture/DATA_MODEL.md`](./docs/architecture/DATA_MODEL.md) | Entities, constraints, immutability |
| 4 | [`docs/architecture/SECURITY_AND_ACCESS_CONTROL.md`](./docs/architecture/SECURITY_AND_ACCESS_CONTROL.md) | Roles, permissions, overrides |
| 5 | The relevant workflow document | The specific behaviour you are changing |
| 6 | The relevant standard | Numbering and/or audit invariants |
| 7 | [`docs/implementation/ACCEPTANCE_CRITERIA.md`](./docs/implementation/ACCEPTANCE_CRITERIA.md) | What "done" means |

### Before changing code

1. Read the workflow document for the area you are touching.
2. Read the architecture section covering that module.
3. Identify the `FR-`/`NFR-` IDs your change serves.
4. Identify the `AC-` entries that will verify it.
5. If no requirement covers the change, **stop** — see §4.

---

## 3. Scope and approval boundaries

### In scope

Only what is authorised in the current phase ([`docs/implementation/IMPLEMENTATION_PLAN.md`](./docs/implementation/IMPLEMENTATION_PLAN.md)).

### Out of scope without explicit written authorisation

- Full ERP functionality (manufacturing, HR, payroll, general ledger).
- Bank integration or payment initiation.
- Native mobile application.
- Microservices, event streaming, message brokers.
- Multi-currency, multi-entity, multi-language UI.
- Warehouse management (bins, lots, serials, cycle counting).
- Customer or supplier portals.
- Real-time GPS tracking.

Full list: PRD §5.3.

### Stop and ask when

- A change would add a feature not covered by an `FR-`/`NFR-`.
- A change would cross a module boundary defined in ARCHITECTURE §2.3.
- A change would relax a constraint, invariant, or permission rule.
- A requirement is ambiguous and the PRD does not resolve it.
- An open question in PRD §14 blocks the work.

**Do not resolve an open question by inventing an answer and encoding it.** Record it, propose a default with rationale, and surface it.

---

## 4. Hard invariants

These are non-negotiable. Violating one is a defect regardless of tests passing.

### Document numbering

1. **Never generate a document number client-side**, in a template, in a report, or in a migration script.
2. **Never use `MAX()+1`, read-then-write, or any non-atomic pattern.**
3. **Always allocate inside the issuing transaction**, using the mechanism in [`docs/standards/DOCUMENT_NUMBERING.md`](./docs/standards/DOCUMENT_NUMBERING.md) §5.
4. **Never remove or weaken the uniqueness constraint.** It is the only authoritative guarantee.
5. **Never mutate or reuse an issued number**, including after void.
6. **Never renumber historical imported documents.**

### Issued documents

7. **Never mutate an issued document.** No field, no snapshot, no number, no PDF bytes.
8. **Corrections are new documents** (credit note, replacement, void) — never edits.
9. **Never render an issued document from live master data.** Always from its snapshot.

### Audit

10. **Never write an audit record outside the transaction of the change it describes.**
11. **Never update or delete an audit record.**
12. **Never let a state change commit without its audit record.** If audit fails, the change fails.
13. **Never disable audit logging to relieve performance or storage pressure.**

### Quantities

14. **Never add a writable quantity column** that a user or endpoint can edit.
15. **Quantity changes only via append-only movements.** Corrections are compensating entries.
16. **Never treat an `UNKNOWN` stock status as available.**

### Overrides

17. **An override may bypass an authorisation conflict. It may never bypass a data-integrity invariant.** No override may produce a duplicate number, an over-reservation, an over-allocation, or an edited issued document.

### Data

18. **Never hard-delete referenced master data.** Deactivate.
19. **Never add a `CASCADE` delete on a business table.**

---

## 5. Coding and architectural conventions

**Status: TBD.** No code exists, so no conventions have been established. Do not invent them.

Once implementation begins, this section must be updated with the actual decisions: language settings, formatting, linting, module structure, error handling, transaction patterns, test framework, and directory layout.

Until then, follow the architectural rules already decided:

| Rule | Source |
|---|---|
| Dependencies point downward only; domain never imports infrastructure | ARCHITECTURE §2.2 |
| One use case = one transaction | ARCHITECTURE §2.2 |
| Authorisation checked in the application layer before any write | ARCHITECTURE §4 |
| Cross-module reads go through public interfaces, never direct table access | ARCHITECTURE §2.3 |
| Illegal module imports must fail CI | ARCHITECTURE §2.1 |
| Permissions are data, not `if role === ...` branches | SECURITY §3 |
| Technician queries are structurally scoped, not filter-parameterised | SECURITY §4 |

---

## 6. Testing and verification requirements

**Status: TBD.** No test framework, runner, or command exists. Do not claim tests pass.

Requirements that hold regardless of tooling:

| Rule | Why |
|---|---|
| A successful UI action is **not** proof | Assert at the level where the property lives |
| Numbering, constraints, concurrency, audit, immutability must be tested at **integration** level against a real database | Mocks prove nothing about these |
| Authorisation must be tested with **direct requests**, not by checking that a button is hidden | UI hiding is not a control |
| Every fixed defect gains a test that fails before the fix | Prevents recurrence |
| Failure behaviour is part of the requirement | Test the failure path, not just the happy path |
| Numbering and audit tests run on every build | They protect the highest-severity properties |
| Constraint tests re-run after every migration | Migrations can silently drop constraints |

Before declaring work complete, state which `AC-` entries were verified and how. If none were executed, say so plainly.

Full strategy: [`docs/implementation/TESTING_STRATEGY.md`](./docs/implementation/TESTING_STRATEGY.md).

---

## 7. Migration and data-integrity rules

1. Migrations are **versioned, forward-only, and rehearsed** on a copy before production (NFR-DATA-006).
2. Migrations must be **backward-compatible for at least one release** (expand → migrate → contract) so rollback is possible.
3. **Never write a destructive migration** that runs automatically on deploy.
4. **Never import data without a validation dry-run** (FR-DATA-002).
5. **Never renumber imported historical documents** (BR-DATA-001).
6. **Never resolve an import collision automatically.** Report it; the business decides.
7. **Never set a series counter below the historical high-water mark** (DOCUMENT_NUMBERING §9).
8. Data imports, exports, and rollbacks are audited.

---

## 8. Security and secrets

1. **Never commit a secret**, credential, key, token, or connection string. Use environment variables or a secret store (NFR-SEC-007).
2. **Never log a secret**, password, hash, token, or authorization header.
3. **Never return a secret in an API response.**
4. **Never inspect or disclose environment values** in documentation, commit messages, or output.
5. **Never weaken authentication or authorisation** to make a test or a demo pass.
6. **Never add a permission bypass** outside the override mechanism.
7. Files are private. **Never generate a permanent public URL** for an attachment (FR-DOC-001).
8. Server-side validation only for uploads; client-side checks are convenience.

---

## 9. Documentation maintenance

Documentation is part of the deliverable, not an afterthought.

| Trigger | Required action |
|---|---|
| Adding a requirement | New `FR-`/`NFR-` in PRD §6/§7, plus `AC-` entries |
| Changing a requirement | Update in place; never renumber. Mark withdrawn as `WITHDRAWN` |
| Changing a status value | Update PRD §9 **and** every document that references it |
| Making an architectural decision | New ADR in [`docs/decisions/`](./docs/decisions/) |
| Resolving an open question | Update the PRD register, the ADR if applicable, and any dependent document |
| Discovering a new open question | Add it to PRD §14 with a proposed default and a decision trigger |
| Changing a workflow step | Update the owning workflow document, then check cross-references |

### Documentation rules

- **Requirement IDs are stable and unique.** Never reuse a number.
- **No duplicate requirement blocks.** Cross-reference instead.
- **All relative links must resolve.** Do not link to files that do not exist.
- **Use classification labels consistently:** `[Confirmed]`, `[Proposed]`, `[Assumption]`, `[Open Question]`, `[Out of Scope]`.
- **Never convert an assumption into a requirement silently.** If an assumption is validated, update its label and record the evidence.
- **Status names must match verbatim** across all documents. Define once (PRD §9), reuse everywhere.
- **No untestable language.** "User-friendly" must become an observable outcome.

---

## 10. Language

- All documentation, identifiers, code, and commit messages: **English**.
- Application UI: **Indonesian** (NFR-OPS-007). This is a planning assumption pending confirmation.
- User-facing error messages must state what went wrong and what to do next, in Indonesian, with no stack traces (NFR-OPS-006).

---

## 11. Reporting to the user

When reporting work:

- State exactly which files were created or modified.
- State which `AC-` criteria were verified and by what method.
- If nothing was executed, say so — **never imply verification that did not occur**.
- Surface any open question encountered rather than resolving it silently.
- Distinguish clearly between what is decided, what is proposed, and what is assumed.

---

## 12. Implementation status

**Implementation has not begun and awaits explicit authorisation.**

This repository is a documentation foundation. Do not scaffold, install dependencies, generate application code, or create configuration files until the business owner authorises implementation and Phase 0 ([`docs/implementation/IMPLEMENTATION_PLAN.md`](./docs/implementation/IMPLEMENTATION_PLAN.md) §3) is complete.
