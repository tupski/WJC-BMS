# Technician Operations — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed |
| Version | 0.1 |
| Scope owner | This document owns scheduling, assignment, daily reporting, evidence capture, blockers, verification, and the field-facing experience |
| Requirement source | [`../PRD.md`](../PRD.md) §6.5, §9.2, §9.11 |
| Related | [`./PROJECT_LIFECYCLE.md`](./PROJECT_LIFECYCLE.md) (project and work-order state machines) |

---

## 1. Confirmed reality

**[Confirmed]:**

- ~15 technicians.
- The owner assigns technicians per project requirement — **one technician or a larger team**.
- Projects can span **multiple days**.
- Technicians currently report progress via **WhatsApp groups**.

**[Assumption]** A-01: technicians have smartphones with a modern browser and intermittent mobile data at customer sites. If false, the entire field-facing design fails and a native/offline approach becomes necessary. This is the single assumption most worth validating early (Phase 0).

### 1.1 What WhatsApp does well, and what it fails at

Understanding this determines what the replacement must preserve.

| WhatsApp does well | WhatsApp fails at |
|---|---|
| Zero training | No structure — a report is a paragraph |
| Instant, familiar | No history beyond scrollback |
| Photos are trivial | No link between a photo and a work order |
| Works on any phone | No status, no verification, no audit |
| Group visibility | No way to answer "which jobs are done?" |
| — | No material usage record |
| — | Office must re-type everything into spreadsheets |

**Design implication:** the replacement must be *at least as fast* as typing a WhatsApp message, or technicians will keep using WhatsApp and the office will keep transcribing. This is why M3 sets a hard target of **< 3 minutes on a phone** and why the reporting form is the most UX-critical screen in the product.

**A successful UI action is not proof the system works** — but a slow UI is proof the system will not be used.

---

## 2. Platform decision: mobile-first responsive web

**[Proposed]** Mobile-first responsive web for MVP. Native application **[Out of Scope]** (Q-UX-01).

### 2.1 Evaluation

| Criterion | Mobile-first web | Native app |
|---|---|---|
| Time to first working version | Low | High (two codebases or a cross-platform framework) |
| Distribution to 15 technicians | Send a link | App store, device compatibility, updates |
| Update cycle | Deploy once | Store review, version fragmentation |
| Photo capture | Browser file/camera input works on modern mobile browsers | Native, slightly better |
| Weak-signal behaviour | Degrades; can add retry/queue | Better offline support |
| Push notifications | Not in MVP | Native |
| Device hardware access | Sufficient (camera, GPS if ever needed) | Full |
| Cost to maintain | Low | Materially higher |

**Recommendation:** mobile-first responsive web, designed at 360 px width, with large touch targets (≥44 px), minimal page weight, and forms that survive a dropped connection by preserving entered data locally until submission succeeds.

**Justification for rejecting native:** the confirmed need is *reporting progress*, which is a form plus photos. A native app would be justified by offline-first requirements or hardware access — neither is confirmed. Building two platforms for a 15-person team would consume the entire MVP budget and delay the business benefits that motivate the project.

**[Open Question] Q-UX-01** — revisit after pilot. **Decision trigger:** measured evidence that web reporting is materially slower or unreliable on technicians' actual devices and networks.

### 2.2 Field UX constraints

| Constraint | Design response |
|---|---|
| 360 px viewport | Single-column layout; no horizontal scrolling; no data tables in field flows |
| Touch input | ≥44 px targets; no hover-dependent interactions |
| Weak/intermittent signal | Lightweight pages; form state preserved locally; explicit retry; no data loss on failure |
| Bright sunlight | High contrast; avoid subtle grey-on-grey |
| One-handed use | Primary actions within thumb reach |
| Indonesian language | All labels, buttons, and messages in Indonesian (NFR-OPS-007) |
| Limited data plan | Compress images client-side before upload; cap upload size |
| Gloves/wet hands | Large buttons; avoid precision interactions |

---

## 3. Technician and team model

### 3.1 Records

| Record | Content |
|---|---|
| Technician | Name, phone, active status, skills/categories where known, linked user account |
| Project assignment | Project, technician, role on project (`TECHNICIAN`/`TEAM_LEADER`), assigned by, assigned at |
| Work order | One per technician per project (or per phase where distinct), planned dates, scope, status |

### 3.2 One or many technicians

Both are normal (FR-PROJ-007):

- **One technician:** one assignment, one work order. No team leader needed.
- **Team of N:** N assignments, N work orders, at most one team leader.

Each technician gets their **own** work order even when working the same project on the same day. This matters because:
- Completion is per-person (one may finish, another may not).
- Daily reports are per-person (accountability).
- Material issue is per-person (custody).
- Verification is per-work-order (partial completion).

A shared work order for a team would make all four impossible.

### 3.3 Team leader

**[Proposed]** Designated per project, not a company-wide role (Q-ROLE-01).

| Capability | Team leader (on that project only) |
|---|---|
| See all work orders in the project | Yes |
| See team members' daily reports | Yes |
| Review/acknowledge reports | Yes (if the review step is enabled) |
| Verify completion of team members' work | Yes, if not their own work |
| Issue material to team members | No (office function) |
| See financial data | **No** |

**Rationale:** multi-day, multi-person projects need one accountable person on site. Without a designated lead, the office chases several people for one answer.

**[Open Question] Q-ROLE-01** — if the owner confirms all technicians report directly to the office, keep the lead designation for accountability but disable the review step. The system supports both without schema change.

---

## 4. Work-order lifecycle

Statuses defined in [`../PRD.md`](../PRD.md) §9.2, transitions in [`./PROJECT_LIFECYCLE.md`](./PROJECT_LIFECYCLE.md) §5.

```
DRAFT ──assign──► ASSIGNED ──ack──► ACCEPTED ──start──► IN_PROGRESS
                                                            │
                                              ┌─────────────┼─────────────┐
                                         blocker        complete       (work continues
                                              │             │          across days)
                                              ▼             ▼
                                          BLOCKED   COMPLETED_PENDING_
                                              │        VERIFICATION
                                       resolved│              │
                                              └──► IN_PROGRESS │
                                                               ├──verify──► VERIFIED
                                                               └──reject──► IN_PROGRESS
```

### 4.1 Multi-day behaviour

A five-day job does not produce five work orders. It produces:

- One work order per technician, `IN_PROGRESS` across all five days.
- One daily report per technician per working day.
- Progress accumulating from reports, not reset daily.

The work order stays `IN_PROGRESS` until the technician submits completion. Days passing without reports is a signal (see §7.3), not a status change.

### 4.2 Partial completion

FR-TECH-008: `COMPLETED_PENDING_VERIFICATION` is a real state, distinct from `VERIFIED`. This allows:

- One technician on a team to finish while others continue.
- Office verification to happen the next morning.
- Verification to reject and return work for rework with a reason.

**The project does not become `COMPLETED` when one work order is verified.** Project status only advances when *all* work orders are terminal (PROJECT_LIFECYCLE §3).

---

## 5. Daily work report

### 5.1 The core field interaction

This is the screen that determines whether the system is adopted.

**Required fields:**

| Field | Type | Notes |
|---|---|---|
| Date | Date, defaults to today | Business date, not timestamp |
| Project / work order | Pre-selected from assignment | Technician does not search |
| Work performed | Free text | Must be fast to type; no forced taxonomy |
| Progress indicator | Selection | See §5.3 |
| Material used | Optional, pre-filled from issued material | Technician confirms quantities |
| Photos | Optional, up to a configured limit | Camera capture or gallery |
| Blocker | Optional | Reason code + note |

**Design rules:**

1. **Pre-fill everything derivable.** Project, work order, date, issued materials — the technician confirms rather than enters.
2. **Free text for work performed.** Forcing a dropdown of activity types would produce "other" for everything and add friction.
3. **Photos must not block submission.** A failed photo upload must not lose the report. Submit the text; retry photos separately.
4. **Save locally until confirmed.** A dropped connection must not destroy ten minutes of typing.
5. **No more than one screen of required input.** Anything beyond that gets abandoned.

### 5.2 Report lifecycle

| Status | Meaning |
|---|---|
| `DRAFT` | Started, not submitted; local only |
| `SUBMITTED` | Received by server, visible to office |
| `REVIEWED` | Acknowledged (by office or team leader, if enabled) |
| `REJECTED` | Returned for correction with a reason |

**BR-TECH-003:** a submitted report is never edited. A correction is a **new version** that supersedes the original; the original remains retrievable. This preserves the record of what was actually reported at the time — which is precisely what WhatsApp scrollback cannot do reliably.

### 5.3 Progress indicator

FR-TECH-010 requires progress to be an **estimate**, not an authoritative measure.

| Approach | Assessment |
|---|---|
| Free percentage entry (0–100) | Easy to type, wildly inconsistent between technicians, and creates false precision |
| Milestone checklist per work order | More consistent, requires setup effort per project |
| Coarse bands (not started / quarter / half / mostly / done) | **Proposed** — low friction, honest about imprecision |
| Derived from completed checklist items | Best consistency, highest setup cost |

**[Proposed] default:** coarse bands mapped to an internal numeric range, with the raw band shown to users. The system displays progress as "perkiraan" (estimate) with a last-updated timestamp, and never presents it as the authoritative completion measure — that is the verification step.

**Rationale:** percentage entry produces numbers like 73% that imply precision nobody has. Bands are honest, faster to select, and aggregate acceptably for workload reporting. Milestone checklists can be added later for project types where they pay for themselves.

### 5.4 Evidence and photos

| Aspect | Rule |
|---|---|
| Storage | Private object storage; never public (FR-DOC-001) |
| Access | Short-lived signed URL after permission check |
| Limits | Configurable per-file size and per-report count, enforced **server-side** (FR-TECH-005) |
| Validation | Content-type sniffing, extension allow-list, image dimension sanity check |
| Metadata | Uploader, timestamp, content hash, linked to the report |
| Retention | Follows the project; not deleted while the project exists |
| Compression | Client-side downscale before upload to respect data plans |

**Why photos matter:** they are the evidence that work was done, and they are the single most valuable thing WhatsApp currently provides. Losing that would be a regression. Linking them to a work order — rather than leaving them in a chat — is the improvement.

### 5.5 Blockers and delays

| Reason code | Use |
|---|---|
| `MATERIAL_NOT_AVAILABLE` | Expected material absent |
| `SITE_NOT_READY` | Customer site inaccessible or unprepared |
| `CUSTOMER_REQUEST` | Customer paused or changed scope |
| `WEATHER` | Outdoor work prevented |
| `EQUIPMENT_FAULT` | Tool failure |
| `ACCESS_DENIED` | Cannot enter site |
| `OTHER` | With note |

A blocker sets the work order to `BLOCKED` and surfaces it to office staff immediately (FR-TECH-006). Blocked work orders appear in a dedicated office view and in the daily operational summary.

**Why this is a material improvement over WhatsApp:** a blocker buried in a group chat is discovered when someone reads the chat. A blocker in the system is a list item that persists until resolved.

---

## 6. Scheduling and rescheduling

### 6.1 Scheduling

| Concept | Behaviour |
|---|---|
| Planned start / target completion | On the project |
| Per-technician planned dates | On the work order |
| Multi-day span | Work order covers the span; daily reports mark actual work days |
| Conflict detection | **[Proposed]** warn when a technician is assigned to overlapping projects; do not block (small teams double up deliberately) |

**[Proposed]** conflict warning rather than hard block: with 15 technicians and urgent jobs, the owner sometimes knowingly double-books. A hard block would be overridden constantly and would train users to ignore warnings.

### 6.2 Rescheduling

FR-TECH-007: changing planned dates requires a reason and is recorded. The work order keeps its history; the previous dates are not overwritten.

### 6.3 Reassignment

Moving a work order to a different technician:

- Requires a reason.
- The new technician sees it; the old one loses access.
- **Material issued to the previous technician does not automatically transfer.** Custody is per-person. The office must record a return from the old technician and an issue to the new one, or an explicit custody transfer. Silently moving custody would break the material ledger's accountability.

This is a deliberate friction point. It reflects reality: material physically held by one person does not teleport when a task is reassigned.

---

## 7. Office oversight

### 7.1 Daily operational view

| Element | Purpose |
|---|---|
| Work orders active today | Who is where |
| Work orders with no report today | Early warning of silence |
| Blocked work orders | Needs action |
| Awaiting verification | Office work queue |
| Projects starting/ending this week | Planning |

### 7.2 Verification queue

FR-TECH-008: office reviews `COMPLETED_PENDING_VERIFICATION` work orders and either verifies or returns for rework with a reason.

Verification checks (guidance, not enforced fields): work performed matches scope, photos present, material accounted for, no unresolved blocker.

**BR-TECH-002:** the verifier must not be the technician who completed the work. If the only available verifier is that technician, an override with reason is required and appears in the owner's override report.

### 7.3 Silence detection

**[Proposed]** A work order `IN_PROGRESS` with no daily report for a configurable period (default: one working day) appears in an exception list.

**Rationale:** the current failure mode is not wrong reports — it is *no* reports, discovered days later. Silence detection converts an invisible problem into a visible one. It is a report, not an automatic status change; the office decides what it means.

---

## 8. Access scope

FR-TECH-009 and NFR-SEC-006 are absolute:

| A technician can | A technician cannot |
|---|---|
| See their own assigned projects and work orders | See other technicians' assignments |
| Submit their own daily reports | See any invoice, payment, or supplier data |
| Upload photos to their own reports | See company-wide reports |
| See materials issued to them | Approve purchases or issue documents |
| Record consumption/return of their own issued material | Change project status |
| Update their own contact details (if enabled) | See other technicians' personal data |

**Structural enforcement:** technician-facing queries are dedicated scoped functions, not general queries with an optional filter (SECURITY §4). A missing filter must be impossible, not merely unlikely.

**Why this matters beyond principle:** a technician seeing invoice values would learn the company's margins on their own work. That is a real, immediate commercial risk in a 15-person team, not a hypothetical privacy concern.

---

## 9. Reporting

| Report | Content | Requirement |
|---|---|---|
| Daily progress per project | All reports for a date, grouped by project | FR-RPT-001 |
| Daily progress per technician | That technician's reports | FR-RPT-001 |
| Workload | Assignments, scheduled days, open and blocked work orders | FR-RPT-006 |
| Completion status | Work orders by status per project | FR-RPT-002 |
| Silence exceptions | Work orders with no recent report | §7.3 |
| Material custody | Material issued per technician, outstanding | FR-RPT-003 |

Reports must be readable on a phone (FR-RPT-001) and exportable (FR-RPT-007).

---

## 10. Failure behaviours

Explicit, because field conditions are hostile:

| Failure | Required behaviour |
|---|---|
| Photo upload fails | Report submits; photo marked failed with retry; no data loss |
| Connection drops mid-form | Entered data preserved locally; resubmission offered on reconnect |
| Duplicate submission (double tap) | Idempotent; one report created |
| Report submitted for wrong date | New version supersedes; original retained (BR-TECH-003) |
| Technician unassigned after starting work | Reports already submitted remain; access to the work order ends |
| Material issue disputed by technician | Office records adjustment with reason; audit trail shows both |
| Work order verified then found incomplete | New work order created linked to the project; verified record stands |

**No failure may result in silent data loss.** That is the minimum standard, and it is the standard WhatsApp fails at.

---

## 11. Acceptance criteria references

| Behaviour | AC |
|---|---|
| Technician record linked to exactly one user | AC-TECH-001 |
| Team leader designation scoped to project | AC-TECH-002 |
| One work order per technician | AC-TECH-003 |
| Daily report in < 3 minutes on 360 px | AC-TECH-004 |
| Photo upload with server-side limits | AC-TECH-005 |
| Blocker recorded and surfaced | AC-TECH-006 |
| Reschedule/reassign recorded with reason | AC-TECH-007 |
| Partial completion and rejection for rework | AC-TECH-008 |
| Technician sees only own assignments | AC-TECH-009 |
| Progress presented as estimate | AC-TECH-010 |
| Multiple technicians on one project | AC-PROJ-007 |
| Multi-day reports accumulate | AC-PROJ-006 |

---

## 12. Cross-references

- Work-order state machine: [`./PROJECT_LIFECYCLE.md`](./PROJECT_LIFECYCLE.md) §5
- Material issue/return: [`./PROCUREMENT_AND_MATERIALS.md`](./PROCUREMENT_AND_MATERIALS.md) §6
- Permissions: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) §3, §4
- Mobile performance targets: [`../PRD.md`](../PRD.md) NFR-OPS-003, NFR-OPS-004
