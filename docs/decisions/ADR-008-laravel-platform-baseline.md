# ADR-008 — Laravel 13 platform baseline: Blade, Hotwire Turbo, TailAdmin, MySQL/MariaDB

| Field | Value |
|---|---|
| Status | **Accepted — client-mandated baseline, not yet installed** |
| Date | 2026-10-25 |
| Supersedes | [ADR-001](./ADR-001-modular-monolith-and-stack.md) (technology stack portion only; the modular-monolith style is carried forward) |
| Deciders | Client (business owner) — mandatory requirement |
| Related | ADR-003 (database), ADR-006 (storage), ADR-009 (allocation), [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) |

---

## Context

ADR-001 chose a TypeScript/Node.js stack on PostgreSQL with S3 storage. That was a **proposal** made against an empty repository with no client platform preference on record.

The client has since stated a **mandatory** platform baseline. This is not a technical preference to be weighed; it is a requirement, and the documentation must reconcile to it rather than argue with it.

**[Confirmed]** client requirements: **Laravel 13.x** (latest stable 13.x patch, never downgraded to 12 or earlier for a template) · server-rendered **Blade** · **Hotwire Turbo** · **TailAdmin Free Laravel** as UI foundation · **MySQL or MariaDB** · **Bahasa Indonesia (`id`)** default and primary · **`Asia/Jakarta`** business timezone · responsive **mobile-first** with **PWA** · **modular monolith** · minimum framework version **≥ 13.29.0**, stated as a security requirement (CVE-2026-114509).

The baseline also removes the previous stack's ability to rely on certain database features, which is why ADR-003 and ADR-009 were revised in the same pass.

---

## Decision

### D1 — Adopt the client baseline as a requirement

The platform items above are requirements (NFR-PLAT-001…009), not proposals. Any deviation requires written client authorisation and a new ADR.

### D2 — Preserve the modular-monolith architecture

The style from ADR-001 survives unchanged: one deployable application, enforced module boundaries, one database, one private file store. Laravel supports this with explicit module directories and a dependency check in CI (AC-OPS-016).

### D3 — Server-rendered Blade plus Hotwire Turbo, no SPA framework

Blade renders pages; Turbo Drive handles navigation and Turbo Frames/Streams handle partial updates. This satisfies the progressive-update requirement without a JavaScript SPA, keeping technician flows light on weak mobile networks.

### D4 — TailAdmin is a UI foundation under a gate, not an assumption

TailAdmin's release, licence, dependency set, and declared Laravel constraint **cannot be audited in this environment** (no network access). Using it is conditional on a **Phase 1 entry gate** (AC-OPS-015) producing a written compatibility and licence sign-off.

**Minimum adaptation path if no Laravel 13-compatible release exists:** treat TailAdmin as a **view layer only** — vendor its Blade components, layouts, and Tailwind assets; replace its service providers, routes, middleware, and auth scaffolding with our own; keep our auth, policies, and routing; pin the framework per this baseline.

**Forbidden responses to incompatibility:** silently downgrading Laravel below 13.x; substituting Filament or another admin panel; adopting a paid TailAdmin edition.

### D5 — Turbo requires idempotent client initialisation

Turbo replaces `<body>` without a full page load, so `DOMContentLoaded`-based initialisers run once and then stop, while naive re-running causes duplicate listeners. The pattern (ARCHITECTURE §3.2, enforced by AC-OPS-009…012): initialise on `turbo:load`/`turbo:frame-load`; be idempotent via `data-initialized`; tear down on `turbo:before-cache`; delegate global behaviours on `document`; reset transient UI state before caching; return **422 with a re-rendered form** for validation failures, never a redirect-with-flash.

### D6 — MySQL or MariaDB, with engine-specific mechanisms

The engine is the client's choice; the version floor is ours (MySQL 8.0.16+ / MariaDB 10.2.1+), because earlier versions parse `CHECK` constraints and ignore them. ADR-003 records the substitutions; ADR-009 rewrites number allocation.

### D7 — Minimum version 13.29.0 is a requirement; its CVE justification is unverified

The client mandates **≥ 13.29.0** citing **CVE-2026-114509**. This environment has **no network access**.

| Claim | Status |
|---|---|
| The client requires ≥ 13.29.0 | **[Confirmed]** — stated requirement |
| CVE-2026-114509 exists, with the nature the client describes | **[Assumption] / [Open Question]** — **not verified in this environment** |
| 13.29.0 is genuinely the fixed version | **[Assumption] / [Open Question]** — **not verified in this environment** |

**No advisory text, CVSS score, affected-component detail, or advisory URL is asserted anywhere in this set.** Inventing them would be worse than admitting the gap, because it would create false confidence in a security control.

**Where to verify** (named as places to check, without asserting contents): the Laravel security advisory channel, the GitHub Advisory Database, and `composer audit` against the resolved dependency set.

**Release gate (mandatory):** before any deployment, `composer audit` is run and the resolved framework version confirmed against the official advisory, with evidence recorded. It is an acceptance criterion (AC-OPS-014) and both a Phase 1 and Phase 6 gate — a control to execute, not a checkbox to tick.

**No security scan was performed in producing this documentation.** None could be.

### D8 — Language and locale are configuration, not convention

`id` is the default and primary locale; `Asia/Jakarta` is the business timezone; timestamps are stored UTC and rendered local; IDR is the currency. Status labels shown to users are Indonesian; the canonical status vocabulary stays as English identifiers so documents can cross-reference it unambiguously.

### D9 — PWA is online-first

Installability, manifest, icons, a service worker caching **only versioned static assets**, and safe cache versioning. No caching of authenticated HTML, JSON, or private files. Offline submission and synchronisation are **not** MVP.

---

## Consequences

### Positive

- **Client alignment.** The delivered system matches the platform the client chose, removing a class of rejection risk at acceptance.
- **Laravel fits this workload.** Auth, policies, queues, mail, the storage abstraction, migrations, and Blade all map onto requirements that would otherwise be hand-built.
- **Turbo gives SPA-like responsiveness without an SPA**, which matters on weak mobile networks.
- **The disk abstraction makes the storage decision reversible** (ADR-006).

### Negative

- **Laravel's upgrade cadence is a real obligation** — roughly an annual major upgrade. Accepted, but must be budgeted rather than discovered.
- **A template dependency can conflict with a framework version.** The specific risk the TailAdmin gate contains (R-09).
- **MySQL/MariaDB lacks several PostgreSQL primitives**, so allocation, conditional uniqueness, case-insensitive uniqueness, and timestamps all need documented workarounds. Each is a place where a future developer could reintroduce a bug by using a "cleaner" idiom that does not work here.
- **No static typing across module boundaries.** Mitigated by strict types, static analysis, and the module dependency check — not eliminated.
- **The CVE claim is unverified**, so the mandated floor may not remediate the issue the client is worried about (R-10).

### Neutral

- The previous stack was never installed, so no migration, porting, data conversion, or downtime is required. This is the cheapest possible moment to change platform.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Keep the TypeScript/Node.js stack from ADR-001 | **Not available** — client-mandated platform; re-litigating it delays delivery for no business benefit |
| Laravel 12 to match an available TailAdmin release | **Forbidden** — the framework carries the security requirement, the template does not. Adapt or replace the template |
| Filament instead of TailAdmin | **Rejected** — an admin-panel framework with its own routing, resource, and authorisation opinions would compete with the module boundaries and the policy layer, and contradicts a client instruction |
| JavaScript SPA with a Laravel API | **Rejected** — doubles surface area, complicates mobile performance, no benefit for form-driven workflows |
| Livewire instead of Turbo | **Rejected** — client specified Turbo; mixing both creates two competing update models |
| PostgreSQL as the engine | **Rejected by client instruction** — would have avoided workarounds, but the engine is a client decision |
| Defer CVE verification until after build | **Rejected** — cheap to verify, high risk; it is a Phase 1 entry gate and a per-deployment release gate |

---

## Verification obligations

| # | Test / record | Proves |
|---|---|---|
| V1 | Resolved framework version recorded from the manifest and lockfile | NFR-PLAT-001 — AC-OPS-014 |
| V2 | Resolved version ≥ 13.29.0 | NFR-PLAT-008 — AC-OPS-014 |
| V3 | `composer audit` output archived | Release gate executed — AC-OPS-014 |
| V4 | Advisory finding recorded against the client's CVE claim | Q-SEC-01 closed with evidence — AC-OPS-014 |
| V5 | TailAdmin package, tag, licence, constraint, and Laravel 13 status recorded in writing | Gate 1 — AC-OPS-015 |
| V6 | No SPA framework in the dependency set | NFR-PLAT-002 |
| V7 | Illegal cross-module import fails the build | NFR-PLAT-007 — AC-OPS-016 |
| V8 | Navigate away and back; components still work | NFR-OPS-011 — AC-OPS-009 |
| V9 | Five navigations then submit: exactly one handler fires | NFR-OPS-011 — AC-OPS-010 |
| V10 | Engine version assertion at boot | NFR-DATA-009 — AC-DATA-009 |

**V3 and V4 are the records that make the security claim honest.** Without them, "we are on a secure version" is an assertion nobody can check.

---

## Open questions

| ID | Question | Default | Owner |
|---|---|---|---|
| Q-UI-01 | TailAdmin release, licence terms including attribution, dependency set, declared framework constraint, Tailwind/Vite versions, Laravel 13 support | Vendor as a view layer if no 13-compatible release exists | Technical lead |
| Q-SEC-01 | Verification of CVE-2026-114509 and whether 13.29.0 is the fixed version | Verify via the official advisory and `composer audit` before the first deployment | Technical lead |
| Q-PLAT-01 | Which specific MySQL or MariaDB version is pinned? | Latest stable supported by the chosen host, at or above the floor | Technical lead |
| Q-PLAT-02 | Queue worker required in MVP, or is synchronous acceptable? | Queue worker from Phase 1, so SMTP test emails and PDF generation do not block requests | Technical lead |
| Q-PLAT-03 | Does the host support disk encryption at rest for the private files directory? | Required where available; otherwise state the limitation explicitly | Technical lead |

---

## What this ADR does not claim

- That Laravel 13 is installed, configured, or running.
- That TailAdmin is compatible, licensed for this use, or evaluated.
- That the CVE reference is accurate, or that 13.29.0 remediates it.
- That a security scan, dependency audit, or build was performed.

Each is either a gate to execute or a question to answer, and each is recorded as such.

---

## Cross-references

- Superseded record: [`./ADR-001-modular-monolith-and-stack.md`](./ADR-001-modular-monolith-and-stack.md)
- Database: [`./ADR-003-single-relational-database.md`](./ADR-003-single-relational-database.md)
- Allocation: [`./ADR-009-mysql-document-number-allocation.md`](./ADR-009-mysql-document-number-allocation.md)
- Storage: [`./ADR-006-private-file-storage.md`](./ADR-006-private-file-storage.md)
- Architecture: [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §3
- Requirements: [`../PRD.md`](../PRD.md) §2.4
