# ADR-006 — Private object storage for documents and evidence

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-25 |
| Deciders | Technical lead |
| Related | [`../architecture/ARCHITECTURE.md`](../architecture/ARCHITECTURE.md) §7, ADR-003 |

---

## Context

The system stores:

- Issued document PDFs (quotations, POs, invoices, credit notes, receipts).
- Technician evidence photos attached to daily reports.
- Supplier invoices and delivery-note scans.
- Expense receipts.
- Customer PO scans.

**[Confirmed]** Technicians report progress via WhatsApp today, which means photos are already central to how the business evidences work. Losing that capability would be a regression.

Requirements:

- FR-DOC-001: private storage, no public readability, access only after a permission check.
- FR-DOC-003: issued PDFs stored immutably.
- FR-DOC-008: original bytes never replaced.
- FR-TECH-005: server-side size and type limits.
- FR-DOC-007: attachments inherit their owner's access rules.

Constraints: small team, no dedicated infrastructure engineer, files are business-critical evidence in disputes.

---

## Decision

### D1 — S3-compatible private object storage

Files live in a **private** bucket. No object is publicly readable. No permanent public URL exists anywhere in the system.

### D2 — Short-lived signed URLs after authorisation

Access flow: authenticate → authorise against the owning record → generate a signed URL valid for minutes → redirect. The signature is a capability issued after a check; it is not the check itself.

### D3 — Opaque object keys

Keys are random identifiers. They never encode customer names, invoice numbers, project names, or any business meaning. A leaked key reveals nothing.

### D4 — Immutable issued artefacts

Issued PDFs are written once. Regeneration produces a new derived artefact; it never overwrites the original. A SHA-256 hash is stored with the record for verification.

### D5 — Metadata in the relational database

The `attachments` table holds owner type/ID, uploader, timestamp, content type, byte size, hash, and soft-delete state. Object storage holds bytes only; it is not a metadata store.

### D6 — Server-side validation

Content-type sniffing, extension allow-list, and size limits enforced on the server. Client-side checks are convenience only.

### D7 — Deletion prohibited for issued-document files

Soft delete only, and only for non-issued attachments. Files belonging to issued documents cannot be deleted by any user.

---

## Consequences

### Positive

- **No accidental exposure.** A misconfigured bucket policy is the classic cause of leaked business documents; keeping objects private and serving through signed URLs removes the permanent-URL failure mode.
- **Access follows business rules.** A technician cannot fetch an invoice attachment because the authorisation check runs before signing.
- **Evidence survives.** Photos are preserved with the work order, not lost in chat scrollback.
- **Immutable artefacts support disputes.** A hash-verified original is defensible.
- **Offloads bytes from the database.** Backups of the relational database stay small and fast.

### Negative

- **Two systems to back up.** Database and object storage must both be covered; a restore that recovers rows but not files is incomplete (AC-DOC-001, B4).
- **Signed URL lifetime is a trade-off.** Too short breaks slow connections; too long weakens the capability. Minutes are the proposed default.
- **No built-in image transformation** in MVP; client-side downscaling handles data-plan concerns.
- **Storage cost grows with photo volume.** At ~15 technicians this is small but not zero; lifecycle rules can archive old evidence if needed.

### Neutral

- The storage provider is interchangeable because the interface is S3-compatible. This avoids provider lock-in for a non-differentiating component.

---

## Alternatives considered

### Alternative A — Files in the database (BLOB columns)

**Rejected.** Bloats backups, degrades restore time, complicates streaming, and makes the database do work it is poor at. Evidence photos are exactly the workload object storage exists for.

### Alternative B — Public bucket with unguessable URLs

**Rejected.** "Unguessable" is not "authorised". URLs leak through browser history, shared links, referrer headers, and screenshots. There is also no way to revoke access once a URL is known. This directly violates FR-DOC-001.

### Alternative C — Local filesystem on the application server

**Rejected.** Ties files to one server, complicates deployment and horizontal scaling, and makes backup/restore a manual, error-prone procedure.

### Alternative D — Third-party document management SaaS

**Rejected.** Introduces an external dependency, a second permission model, and a data-residency question for business documents — for functionality that is fundamentally simple.

### Alternative E — Permanent signed URLs with long expiry

**Rejected.** Equivalent to a public link in practice. Short expiry is what makes the capability revocable.

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Direct unauthenticated object URL | Fails — AC-DOC-001 |
| V2 | Signed URL after expiry | Fails — AC-DOC-001 |
| V3 | Technician requests an invoice attachment | 403 before signing — AC-DOC-007 |
| V4 | Oversized file upload | Rejected server-side — AC-TECH-005 |
| V5 | Disallowed content type | Rejected server-side — AC-TECH-005 |
| V6 | Re-download issued PDF after template change | Hash matches original — AC-DOC-008 |
| V7 | Attempt to delete an issued-document file | Refused — AC-DOC-008 |
| V8 | Restore object storage from backup | Files retrievable and hash-verified — B4 |

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-FILE-01 | Maximum photo size and count per report | 5 MB per file, 10 files per report (configurable) |
| Q-FILE-02 | Evidence retention after project closure | Retain with the project; never auto-delete |
| Q-FILE-03 | Is image compression applied server-side or client-side? | Client-side downscale before upload; server accepts as-is within limits |
| Q-FILE-04 | Storage region and data residency requirement | Same region as the database (Q-OPS-01) |
