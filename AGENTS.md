# AGENTS.md — Working Agreement for Coding Agents

This file governs how automated coding agents work in this repository. It is intentionally short and imperative. It does not restate requirements; it points at the authoritative sources.

---

## 1. Current repository state

**This repository contains documentation only.**

Measured 2026-10-25: Git metadata present, branch `main`, clean tree, `origin` → `https://github.com/tupski/WJC-BMS.git`. The client states the project is hosted at GitHub `tupski/WJC-BMS` — confirmed as a client statement. **No network access was available**, so remote state and commit SHAs are unverifiable offline and must not be asserted.

There is **no application code, no installed technology stack, no `composer.json`/`package.json`/lockfile/`.env`/configuration file, and no build, lint, test, or run command.** Do not invent any of the above. Do not state that a command works, that a test passes, or that a framework is running.

**Historical note, for traceability only:** the documentation set was first authored against a workspace verified empty (no `.git`, no `docs/`). True then; not current status.

The **platform baseline is client-mandated**, not a proposal: Laravel 13.x, Blade, Hotwire Turbo, TailAdmin Free Laravel (gated), MySQL/MariaDB, `id` locale, `Asia/Jakarta`, PWA, modular monolith. See [`docs/decisions/ADR-008-laravel-platform-baseline.md`](./docs/decisions/ADR-008-laravel-platform-baseline.md). It has **not been installed**.

---

## 2. Authoritative documents and reading order

[`docs/PRD.md`](./docs/PRD.md) is the **single source of truth** for scope and functional requirements. It owns all `FR-`/`NFR-` identifiers, all business rules (`BR-`), and the canonical status vocabulary (PRD §9).

If any other document contradicts the PRD, **the PRD wins** and the contradiction must be raised, not silently resolved.

| Order | Document | Why |
|---|---|---|
| 1 | [`docs/PRD.md`](./docs/PRD.md) | Scope, requirements, statuses, business rules |
| 2 | [`docs/decisions/ADR-008-laravel-platform-baseline.md`](./docs/decisions/ADR-008-laravel-platform-baseline.md) | The mandatory platform and its gates |
| 3 | [`docs/architecture/ARCHITECTURE.md`](./docs/architecture/ARCHITECTURE.md) | Module boundaries, Turbo lifecycle, engine notes |
| 4 | [`docs/architecture/DATA_MODEL.md`](./docs/architecture/DATA_MODEL.md) | Entities, MySQL types, constraints, immutability |
| 5 | [`docs/architecture/SECURITY_AND_ACCESS_CONTROL.md`](./docs/architecture/SECURITY_AND_ACCESS_CONTROL.md) | Four roles, permissions, overrides |
| 6 | The relevant workflow document | The specific behaviour you are changing |
| 7 | The relevant standard | Numbering and/or audit invariants |
| 8 | [`docs/implementation/ACCEPTANCE_CRITERIA.md`](./docs/implementation/ACCEPTANCE_CRITERIA.md) | What "done" means |

**Before changing code:** read the workflow and the architecture section for that module, identify the `FR-`/`NFR-` IDs the change serves, identify the `AC-` entries that verify it, and if no requirement covers the change, **stop** (§4).

---

## 3. Scope and approval boundaries

**In scope:** only what is authorised in the current phase ([`docs/implementation/IMPLEMENTATION_PLAN.md`](./docs/implementation/IMPLEMENTATION_PLAN.md)). **Phase 1 cannot begin until both platform gates are signed off.**

**Out of scope without explicit written authorisation:** full ERP (manufacturing, HR, payroll, general ledger) · bank integration or payment initiation · native mobile application · microservices, event streaming, message brokers · multi-currency, multi-entity, multi-language UI · warehouse management (bins, lots, serials, cycle counting) · customer or supplier portals · real-time GPS tracking · WhatsApp Business API, automated posting, delivery receipts · offline submission and synchronisation · Google Maps API integration beyond deep links. Full list: PRD §5.3.

**Stop and ask when:** a change would add a feature not covered by an `FR-`/`NFR-` · would cross a module boundary (ARCHITECTURE §2.2) · would relax a constraint, invariant, or permission rule · would require **downgrading Laravel below 13.x**, substituting the UI framework, or adopting a paid template edition · a requirement is ambiguous and the PRD does not resolve it · an open question in PRD §14 blocks the work.

**Do not resolve an open question by inventing an answer and encoding it.** Record it, propose a default with rationale, and surface it.

---

## 4. Hard invariants

Non-negotiable. Violating one is a defect regardless of tests passing.

**Document numbering**

1. Never generate a document number client-side, in a template, in a report, or in a migration script.
2. Never use `MAX()+1`, read-then-write, or any non-atomic pattern.
3. Always allocate inside the issuing transaction, per [`docs/standards/DOCUMENT_NUMBERING.md`](./docs/standards/DOCUMENT_NUMBERING.md) §5 and ADR-009.
4. Never remove or weaken the uniqueness constraint — it is the only authoritative guarantee.
5. Never mutate or reuse an issued number, including after void. A configured increment step does **not** authorise reuse.
6. Never renumber historical imported documents.
7. Never lower a series counter below its high-water mark.
8. Never make a numbering configuration change retroactive.

**Issued documents**

9. Never mutate an issued document — no field, snapshot, number, or PDF bytes.
10. Corrections are new documents (credit note, replacement, void), never edits.
11. Never render an issued document from live master data — including company name, logo, and bank details. Always from its snapshot.

**Work sessions**

12. Never accept a client-supplied session timestamp. The server writes start and completion.
13. Never allow a session timestamp to be edited. Corrections are new, attributed records with a mandatory reason; originals remain.
14. Never treat ending a session or submitting completion as project completion. Only an authorised verification does that.

**Purchasing**

15. Never allow a PO to be issued without a recorded `PEMILIK` approval. No threshold, auto-approval, or delegation.
16. Never let an override waive purchase approval — it is a business control, not an authorisation conflict.
17. Never give `purchase.approve` to any role other than `PEMILIK`, including `SUPER_ADMIN`.

**Audit**

18. Never write an audit record outside the transaction of the change it describes.
19. Never update or delete an audit record.
20. Never let a state change commit without its audit record. If audit fails, the change fails.
21. Never disable audit logging to relieve performance or storage pressure.
22. Never write a secret into an audit record. Settings changes are audited by key, not value.

**Quantities**

23. Never add a writable quantity column that a user or endpoint can edit.
24. Quantity changes only via append-only movements; corrections are compensating entries.
25. Never treat an `UNKNOWN` stock status as available.
26. Never let a PO increase available stock. Only a posted goods receipt does.

**Storage**

27. Never store a file inside the webroot, and never generate a permanent public URL.
28. Never call `public_path()` from business logic or a Blade template, and never hard-code a filesystem path. Use the configured disk.
29. Never store an absolute path or URL in an attachment row — store disk plus relative path.
30. Never delete a file belonging to an issued document.

**Overrides**

31. An override may bypass an authorisation conflict. It may never bypass a data-integrity invariant, a session-timestamp rule, or the purchase-approval requirement. No override may produce a duplicate number, an over-reservation, an over-allocation, an edited issued document, or a backdated session.

**Data**

32. Never hard-delete referenced master data. Deactivate.
33. Never add a `CASCADE` delete on a business table. `ON DELETE RESTRICT` is the default.

**Engine**

34. Never rely on `RETURNING` on `UPDATE`, `ON CONFLICT`, `JSONB`, `citext`, partial unique indexes, deferrable FKs, or transactional DDL. None exist in the mandated engine (DATA_MODEL §1.5).
35. Never assume a `CHECK` constraint is enforced below MySQL 8.0.16 / MariaDB 10.2.1.

**Platform**

36. Never downgrade Laravel below 13.x to accommodate a template or package. Adapt or replace the template.
37. Never claim the CVE/13.29.0 relationship is verified unless you have run `composer audit`, checked the official advisory, and recorded the evidence.

---

## 5. Coding and architectural conventions

**Status: TBD.** No code exists, so no conventions have been established. Do not invent them.

Once implementation begins, this section must record the actual decisions: formatting, static analysis level, module structure, error handling, transaction patterns, test framework, and directory layout.

Until then, follow the architectural rules already decided:

| Rule | Source |
|---|---|
| Dependencies point downward only; domain never imports infrastructure | ARCHITECTURE §2.1 |
| One use case = one transaction | ARCHITECTURE §2.1 |
| Authorisation checked in the application layer before any write | ARCHITECTURE §4 |
| Cross-module reads go through public interfaces, never direct table access | ARCHITECTURE §2.2 |
| Illegal module imports must fail the build | ARCHITECTURE §2 |
| Permissions are data, not role-name branches | SECURITY §3 |
| Technician queries are structurally scoped, not filter-parameterised | SECURITY §4 |
| Turbo components initialise idempotently on `turbo:load`, never on `DOMContentLoaded` alone | ARCHITECTURE §3.2 |
| Validation failures return 422 with a re-rendered form, not a redirect-with-flash | ARCHITECTURE §3.2 |
| Persistent settings are read through a runtime resolver, not baked into `config:cache` | ARCHITECTURE §8.1 |

---

## 6. Testing and verification

**Status: TBD.** No test framework, runner, or command exists. Do not claim tests pass.

| Rule | Why |
|---|---|
| A successful UI action is **not** proof | Assert at the level where the property lives |
| Numbering, constraints, concurrency, audit, and immutability must be tested at **integration** level against a real database **at the pinned engine version** | Mocks and wrong engines prove nothing |
| Authorisation must be tested with **direct requests**, not by checking that a button is hidden | UI hiding is not a control |
| Turbo navigation and service-worker caching must be tested in a **real browser** | Server tests cannot see these failures |
| Every fixed defect gains a test that fails before the fix | Prevents recurrence |
| Failure behaviour is part of the requirement | Test the failure path, not just the happy path |
| Numbering, approval, and audit tests run on every build | They protect the highest-severity properties |
| Constraint tests re-run after every migration | Migrations can silently drop a constraint |

Before declaring work complete, state which `AC-` entries were verified and how. If none were executed, say so plainly. Full strategy: [`docs/implementation/TESTING_STRATEGY.md`](./docs/implementation/TESTING_STRATEGY.md).

---

## 7. Migration and data-integrity rules

1. Migrations are **versioned, forward-only, and rehearsed** on a copy before production (NFR-DATA-006).
2. Migrations must be **backward-compatible for at least one release** (expand → migrate → contract), because DDL commits implicitly and cannot be rolled back atomically (NFR-DATA-010).
3. Never write a destructive migration that runs automatically on deploy.
4. Never import data without a validation dry-run (FR-DATA-002).
5. Never renumber imported historical documents (BR-DATA-001).
6. Never resolve an import collision automatically. Report it; the business decides.
7. Never set a series counter below the historical high-water mark.
8. Data imports, exports, and rollbacks are audited.

---

## 8. Security and secrets

1. Never commit a secret, credential, key, token, or connection string (NFR-SEC-007).
2. Never log a secret, password, hash, token, authorization header, **or SMTP credential** (NFR-SEC-009).
3. Never return a secret in a response, and never render one into page source or client JavaScript.
4. Never display a stored credential in a form. The field is masked and write-only.
5. Never inspect or disclose environment values in documentation, commit messages, or output.
6. Never weaken authentication or authorisation to make a test or demo pass.
7. Never add a permission bypass outside the override mechanism.
8. Files are private. Never generate a permanent public URL for an attachment (FR-DOC-001).
9. Server-side validation only for uploads; client-side checks are convenience. Use MIME sniffing, never extension trust.

---

## 9. Documentation maintenance

| Trigger | Required action |
|---|---|
| Adding a requirement | New `FR-`/`NFR-` in PRD §6/§7, plus `AC-` entries |
| Changing a requirement | Update in place; never renumber. Mark withdrawn as `WITHDRAWN` |
| Changing a status value | Update PRD §9 **and** every document that references it |
| Making an architectural decision | New ADR in [`docs/decisions/`](./docs/decisions/) |
| Superseding a decision | New ADR; old status → `Superseded by ADR-NNN`; **keep the body as history** with a supersession note |
| Resolving an open question | Update the PRD register with the resolution and keep the history row |
| Discovering a new open question | Add to PRD §14 with a proposed default and a decision trigger |
| Changing a workflow step | Update the owning workflow, then check cross-references |

**Rules:** requirement IDs are stable and unique — never reuse a number, never silently change its meaning · no duplicate requirement blocks; cross-reference instead · all relative links must resolve · use the classification labels consistently (`[Confirmed]`, `[Proposed]`, `[Assumption]`, `[Open Question]`, `[Out of Scope]`) · never convert an assumption into a requirement silently · status names match verbatim across all documents · no untestable language — "user-friendly" must become an observable outcome · **diagrams are Mermaid** and require a Mermaid-capable renderer; never claim a diagram renders unless you rendered it · **never assert a repository fact you have not measured** — no branch, SHA, or remote state without command output.

---

## 10. Language

All documentation, identifiers, code, and commit messages in **English**. Application UI in **Indonesian (`id`)** — client-mandated (NFR-OPS-007). User-facing errors state what went wrong and what to do next, in Indonesian, with no stack traces (NFR-OPS-006). Business timezone **`Asia/Jakarta`**; timestamps stored **UTC** and rendered locally (NFR-OPS-009).

---

## 11. Reporting to the user

State exactly which files were created or modified · state which `AC-` criteria were verified and by what method · if nothing was executed, say so — **never imply verification that did not occur** · never claim a build, test, dependency audit, security scan, or diagram render occurred unless it did · surface any open question rather than resolving it silently · distinguish clearly between what is decided, what is proposed, and what is assumed.

---

## 12. Implementation status

**Implementation has not begun and awaits explicit authorisation.**

Do not scaffold, install dependencies, generate application code, or create configuration files until the business owner authorises implementation **and** both Phase 1 platform gates are signed off ([`docs/implementation/IMPLEMENTATION_PLAN.md`](./docs/implementation/IMPLEMENTATION_PLAN.md) §4.0).
