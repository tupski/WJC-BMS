# ADR-006 — Private file storage: local-first with an S3 migration path

| Field | Value |
|---|---|
| Status | **Accepted (design) — amended in place 2026-10-25: local-first replaces S3-first** |
| Date | 2026-10-25 |
| Revision | Rev 2 — primary store changed; access model retained |
| Deciders | Technical lead |
| Related | [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §7, ADR-003, ADR-008 |

> **Supersession note.** Rev 1 chose an **S3-compatible private bucket** as the primary store and named "local filesystem on the application server" as a **rejected** alternative. The client has since specified **local-first storage**. Rev 2 reverses that specific choice and retains everything else: private by default, no public URLs, authorisation before access, opaque keys, immutable issued artefacts, metadata in the relational database, server-side validation.
>
> Rev 1's rejection reasoning was sound for a multi-server deployment. At this scale — one application server, ~18 users, modest file volume — it over-engineered the problem and added an external dependency. The mitigation for the original concern (server coupling) is now the **disk abstraction**, which keeps the S3 path open without requiring S3 today.

---

## Context

The system stores issued PDFs, technician evidence photos, supplier invoices and delivery-note scans, expense receipts, customer PO scans, and the company logo.

**[Confirmed]** Technicians report progress via WhatsApp today, so photos are already central to how the business evidences work; losing that capability would be a regression.

Requirements: FR-DOC-001 (private, access only after a permission check), FR-DOC-003 (immutable PDFs), FR-DOC-008 (original bytes never replaced), FR-DOC-009 (single disk abstraction), FR-TECH-005 (server-side limits), FR-DOC-007 (attachments inherit owner access), NFR-SEC-010 (authorised streamed downloads, no directory listing).

Constraints: small team, no dedicated infrastructure engineer, files are business-critical evidence in disputes, single-server deployment.

---

## Decision

### D1 — Private local filesystem outside the webroot

Files live on a **dedicated private disk** — e.g. `storage/app/private` — never inside the public document root. No object is publicly readable. No permanent public URL exists anywhere.

### D2 — One disk abstraction, addressed by name

All access goes through the storage facade with a named disk. Business logic and Blade templates never contain filesystem paths and never call `public_path()`. This is what makes the S3 migration a configuration change rather than a module rewrite (FR-DOC-009).

### D3 — Authorisation before access, streamed delivery

Authenticate → authorise against the owning record → stream the response with the correct `Content-Type` and `Content-Disposition`. Where a future driver supports it, temporary URLs are used instead; the authorisation check happens **before** signing, so a signed URL is a capability issued after a check, not the check itself.

### D4 — Opaque keys

Keys are random identifiers, never encoding customer names, invoice numbers, or business meaning.

### D5 — Immutable issued artefacts

Issued PDFs are written once. Regeneration produces a new derived artefact; it never overwrites the original. A SHA-256 hash is stored for verification.

### D6 — Metadata in the relational database

The `attachments` table holds owner type/ID, **disk name, relative path**, uploader, timestamp, content type, byte size, hash, dimensions, and soft-delete state. **No absolute URL and no absolute filesystem path is stored** — that is what allows the disk to be relocated.

### D7 — Server-side validation

MIME sniffing (never extension trust), size limit, image dimension limit, filename sanitisation, and path-traversal prevention, all enforced server-side. Client-side checks are convenience only.

### D8 — Deletion prohibited for issued-document files

Soft delete only, and only for non-issued attachments.

### D9 — Backup covers files as well as the database

A restore that recovers rows but not files is an incomplete restore (NFR-DATA-004/005, AC-DATA-004). The files directory is included in the daily backup and in every restore rehearsal.

### D10 — S3 migration is planned, not improvised

Moving later requires: adding the disk configuration, a verified copy-and-verify migration of existing objects, switching the disk name, and confirming hashes match. Because `disk` is stored per attachment, a partial migration is possible and auditable.

---

## Consequences

### Positive

- **No external dependency for MVP** — no bucket policy to misconfigure, no credentials to rotate, no egress cost.
- **No accidental exposure.** Files sit outside the webroot and are served only through an authorising route.
- **Simpler backup at this scale** — a database dump plus a directory archive is a procedure one person can run and verify.
- **Access follows business rules.** A technician cannot fetch an invoice attachment because the check runs before the file is opened.
- **Immutable artefacts support disputes.** A hash-verified original is defensible.
- **The S3 path stays open** through the disk abstraction, at no cost today.

### Negative

- **Files are tied to one server.** Horizontal scaling would require shared storage — exactly why the migration path exists. Accepted for a single-server deployment.
- **Backup discipline is now load-bearing.** A database-only backup policy silently loses every photo and every issued PDF. This is the highest-severity operational risk introduced by this decision, and why D9 is a decision rather than a suggestion.
- **Restore is a two-part procedure**, so it is easier to half-do. The rehearsal must verify files, not just rows.
- **Disk growth is unbounded** without a retention decision (Q-FILE-01).
- **No built-in image transformation**; client-side downscaling handles data-plan concerns.

### Neutral

- The storage driver is interchangeable because the interface is a single disk. Provider lock-in is avoided without paying for S3 today.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Files in the database (BLOB columns) | **Rejected** — bloats backups, degrades restore time, complicates streaming, makes the database do work it is poor at |
| Public directory with unguessable filenames | **Rejected** — "unguessable" is not "authorised"; URLs leak through history, shared links, referrers, and screenshots, and access cannot be revoked. Violates FR-DOC-001 |
| S3-compatible object storage as the primary store | **Superseded** — Rev 1's choice. Rejected now because it adds an external dependency, credential management, and egress cost for a single-server deployment with modest volume. Retained as the **documented migration target** |
| Third-party document management SaaS | **Rejected** — external dependency, a second permission model, and a data-residency question for functionality that is fundamentally simple |
| Hybrid: local for drafts, S3 for issued documents | **Rejected for MVP** — two storage paths doubles the authorisation surface and the backup procedure, and splits the immutability guarantee across two systems |
| Permanent signed URLs with long expiry | **Rejected** — equivalent to a public link in practice |

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Direct unauthenticated path to a stored file | Fails — AC-DOC-001 |
| V2 | Directory listing attempt | Impossible — AC-DOC-010 |
| V3 | Technician requests an invoice attachment | 403 before the file is opened — AC-DOC-007 |
| V4 | Oversized, wrong-type, or over-dimension upload | Rejected server-side — AC-TECH-005 |
| V5 | Filename containing `../` | Sanitised or rejected; no traversal — AC-DOC-001 |
| V6 | Re-download issued PDF after template change | Hash matches — AC-DOC-008 |
| V7 | Attempt to delete an issued-document file | Refused — AC-DOC-008 |
| V8 | **Restore files directory from backup** | Files retrievable and hash-verified — AC-DATA-004 |
| V9 | **Change the disk configuration** | No business-logic or Blade change required — AC-DOC-009 |
| V10 | Inspect attachment rows | No absolute path or URL stored — AC-DOC-009 |

**V8 is the test most likely to be skipped and most costly to skip.** A backup regime covering only the database leaves the business unable to produce any document it ever issued.

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-FILE-01 | Maximum photo size, count per report, and retention | 5 MB per file, 10 files per report, retained with the project |
| Q-FILE-02 | Evidence retention after project closure | Retain with the project; never auto-delete |
| Q-FILE-03 | Image compression server-side or client-side? | Client-side downscale; server accepts within limits |
| Q-FILE-04 | When does the S3 migration trigger? | When file volume exceeds local disk capacity, or a second application server is introduced |
| Q-FILE-05 | Is the files directory encrypted at rest? | Yes if the host supports it; otherwise rely on host-level disk encryption and state the limitation |

---

## Cross-references

- Architecture: [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §7
- Data model: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §8
- Document workflow: [`../workflows/FINANCE_AND_DOCUMENTS.md`](../workflows/FINANCE_AND_DOCUMENTS.md) §9
- Technician evidence: [`../workflows/TECHNICIAN_OPERATIONS.md`](../workflows/TECHNICIAN_OPERATIONS.md) §5.4
- Platform: [`./ADR-008-laravel-platform-baseline.md`](./ADR-008-laravel-platform-baseline.md)
