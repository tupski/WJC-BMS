# ADR-001 — Modular monolith architecture and technology stack

| Field | Value |
|---|---|
| Status | **Superseded by [ADR-008](./ADR-008-laravel-platform-baseline.md)** (2026-10-25) |
| Date | 2026-10-25 |
| Superseded on | 2026-10-25 |
| Deciders | Business owner, technical lead |
| Related | ADR-008 (replacement), ADR-003, [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) |

> **Supersession note.** The **architectural style** decided here — a modular monolith with enforced module boundaries — was **accepted** and carried forward unchanged. The **technology stack** was superseded by client instruction: Laravel 13.x, Blade, Hotwire Turbo, TailAdmin, MySQL/MariaDB, and local-first storage are now mandatory (ADR-008). The body below is retained **as history**; where it and ADR-008 disagree, ADR-008 governs.

---

## Context

The workspace was verified empty at the time: no code, no stack, no build or test commands, no conventions. The choice was open — and there was no evidence to justify an unusual one.

Confirmed operating context: ~3 office employees, ~15 technicians; projects spanning multiple days with teams from one to several; administration on spreadsheets, documents, and WhatsApp; a document-numbering conflict as the highest-severity known defect; no centralised inventory.

The two requirements that most constrain the architecture were **FR-DOC-004 / NFR-DATA-002** (atomic, unique number allocation under concurrency) and **FR-MATL-006** (reservations must never exceed verified stock). Both are transactional-correctness problems — trivial inside one ACID transaction, materially harder across process or service boundaries.

---

## Decision

Adopt a **modular monolith**: one deployable application, one relational database, one private file store, with enforced module boundaries.

### D1 — Architectural style (still current)

| Decision | Status |
|---|---|
| Single deployable application | **Accepted**, carried into ADR-008 |
| Enforced module boundaries with a CI dependency check | **Accepted**, carried into ADR-008 |
| One relational database | **Accepted**, carried into ADR-003 |
| Private file storage, never public | **Accepted**, carried into ADR-006 |
| Server-rendered UI with progressive enhancement | **Accepted**, carried into ADR-008 as Blade + Turbo |

### D2 — Technology stack as proposed at the time (superseded)

| Layer | Original proposal | Superseded by |
|---|---|---|
| Language / runtime | TypeScript on a Node.js LTS runtime | PHP on Laravel 13.x (ADR-008) |
| Database | PostgreSQL | MySQL or MariaDB (ADR-008, ADR-003) |
| Data access | Typed ORM with a raw-SQL escape hatch | Eloquent plus explicit transactions and targeted raw statements (ADR-008) |
| UI | Server-rendered pages, progressive enhancement | Blade + Hotwire Turbo + TailAdmin (ADR-008) |
| File storage | S3-compatible private object storage | Private local disk with an S3 migration path (ADR-006) |

**Nothing was ever installed under this proposal.** No manifest, lockfile, configuration, or source directory was created.

---

## Consequences

### Positive (still true of the accepted style)

- **Transactional correctness is native.** Number allocation, reservation checks, and audit coupling are ordinary transactions — no sagas, no outbox, no distributed compensation.
- **One deployment artefact.** A small team can operate it without a platform engineer.
- **One database to back up and restore.** Recovery is a single, testable procedure.
- **Debuggability.** A single request trace covers the whole operation.

### Negative

- **Boundaries erode without enforcement.** A modular monolith degrades into a monolith with folders unless a dependency check runs in CI (now AC-OPS-016).
- **One deploy unit** means a change anywhere redeploys everything. Acceptable at this scale.
- **Scaling is vertical first.** Acceptable for ~18 users.
- **A risk this record did not consider:** a mandatory UI template can conflict with a mandatory framework version. ADR-008 addresses it with an explicit compatibility gate, a view-layer adaptation path, and a prohibition on resolving the conflict by downgrading the framework.

### Neutral

- Extraction to services remains possible later if a module proves genuinely independent. It is not planned.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Microservices | **Rejected** — no operational capacity for service discovery, distributed tracing, and multiple deploys; converts the two highest-risk correctness problems into distributed-systems problems |
| Laravel (PHP) monolith | Recorded as viable but not chosen, on the grounds that end-to-end static typing makes boundary enforcement easier to verify. **The client has since mandated Laravel**, resolving the question in favour of this alternative |
| Django (Python) monolith | **Rejected** — weaker compile-time contract enforcement, moderate local hiring depth. Not revisited |
| SPA frontend + separate API | **Rejected** — doubles surface area, complicates mobile performance, no benefit for form-driven workflows. Turbo supplies the progressive-update behaviour without an SPA |
| Event-sourced / CQRS | **Rejected** — the audit log already provides history; adds reconstruction complexity and eventual consistency for no business gain |

---

## What survives this supersession

| Element | Survives? | Where it now lives |
|---|---|---|
| Modular monolith with enforced boundaries | Yes | ADR-008, ARCHITECTURE §2 |
| One relational database | Yes | ADR-003 |
| Immutable issued-document snapshots | Yes | ADR-003 |
| Append-only quantity ledger | Yes | ADR-003, DATA_MODEL §5 |
| Private file storage | Yes, local-first | ADR-006 |
| Server-rendered UI with progressive enhancement | Yes | ADR-008 (Blade + Turbo) |
| The original language, database, and storage choices | **No** | Superseded |

---

## Verification obligations (carried forward)

| # | Test | Proves |
|---|---|---|
| V1 | Illegal cross-module import fails the build | Boundaries are enforced, not nominal — AC-OPS-016 |
| V2 | Numbering, reservation, and audit guarantees hold inside single transactions | The monolith argument is real, not rhetorical |

---

## Cross-references

- Replacement: [`./ADR-008-laravel-platform-baseline.md`](./ADR-008-laravel-platform-baseline.md)
- Database: [`./ADR-003-single-relational-database.md`](./ADR-003-single-relational-database.md)
- Storage: [`./ADR-006-private-file-storage.md`](./ADR-006-private-file-storage.md)
- Architecture: [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md)
