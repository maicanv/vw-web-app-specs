# Feature Specification: Extraction Review — Confidence & Control Step

**Feature Branch**: `006-extraction-review`

**Created**: 2026-07-06

**Status**: Draft

**Input**: Confluence story "[DE] US: Extraction Review" (https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/341803023/DE+US+Extraction+Review). Per the requester, the open questions on the source page stay open in this spec and will be resolved later.

## What already works today (context, not in scope)

- Every extracted value carries a confidence score, and each record carries an overall confidence score. Both show in the records list and detail views.
- Records can carry a **Needs Review** status, exposed through list filters and status badges.
- A confidence threshold can be stored per document type (and per field). The system stores it but does not act on it yet.
- Records can be sent to downstream systems over REST with automatic or manual sending, retry, and re-send, plus a history of delivery attempts.

This spec covers the remaining scope: acting on the stored thresholds, holding flagged records, correcting values, a user-facing audit trail, and (as a dependent story) reviewer assignment.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Flag low-confidence records for review (Priority: P1)

A user configures a confidence threshold (e.g. 80%) on a document type. After extraction, the system compares the record's confidence against that threshold and marks records that fall below it as Needs Review, together with the reason the flag fired. Records the system could not score at all get the same treatment.

**Why this priority**: This is the trigger for everything else. Without automatic flagging, the review workflow has nothing to route, and the stored threshold setting remains dead configuration.

**Independent Test**: Configure a threshold on a document type, run an extraction that scores below it, and verify the record shows as Needs Review with a visible reason. Records above the threshold stay unflagged.

**Acceptance Scenarios**:

1. **Given** a record has been extracted, **When** its overall confidence is below the configured threshold, **Then** the record is marked Needs Review.
2. **Given** a record has been extracted, **When** a field's confidence is below the threshold, **Then** the record is marked Needs Review. *(Open question OQ-2: any field, or critical fields only.)*
3. **Given** a record is marked Needs Review, **Then** the reviewer sees which rule triggered the flag, and the record appears in list filters and status indicators.
4. **Given** I configure the confidence threshold on a document type, **Then** that value determines which of its records are marked Needs Review.
5. **Given** the system cannot compute confidence for a record, **Then** it marks the record Needs Review and shows a plain "confidence unavailable" message.

---

### User Story 2 - Hold flagged records before sending (control step) (Priority: P1)

A record flagged Needs Review is held: the system does not deliver it downstream until a person acts. The reviewer opens the record and chooses one of three actions: Confirm & Send, Correct, or Reject. Unflagged records with no blocking issues continue to send automatically where automatic REST output is configured.

**Why this priority**: The business goal of the feature is that wrong data never reaches downstream systems (e.g. an ERP). Flagging without a hold does not deliver that promise.

**Independent Test**: With a REST output configured for automatic sending, produce one flagged and one unflagged record. Verify the unflagged record sends and the flagged record waits until a reviewer confirms, corrects, or rejects it.

**Acceptance Scenarios**:

1. **Given** a record is Needs Review, **Then** the system does not send it and waits for a user action. The Needs Review status drives the hold, not the raw confidence score.
2. **Given** I am reviewing a held record, **Then** I can Confirm & Send, Correct, or Reject it.
3. **Given** a record is not Needs Review and has no blocking issues, **Then** it can be sent automatically when REST output is configured.
4. **Given** the system blocks a send, **Then** the UI explains why and offers Confirm / Correct / Reject.

---

### User Story 3 - Correct values on any record (Priority: P2)

A user can fix extracted values on any record regardless of its confidence, including false positives where the confidence looked fine and records that were already sent. Corrections are saved, the old and new value both remain visible (before → after), and the value is marked as manually edited. An unsent corrected record becomes eligible to send.

**Why this priority**: Correction is one of the three review actions, but it also stands alone: users need it even for records the system never flagged. It depends on nothing in Story 2 beyond the record detail view that already exists.

**Independent Test**: Open any record (flagged or not), change a value, and verify the change persists, shows before → after, is marked manually edited, and (for unsent records) the record becomes sendable.

**Acceptance Scenarios**:

1. **Given** any record, including one already sent, **When** I correct one or more values, **Then** the changes are saved, and if the record has not been sent it becomes eligible to send.
2. **Given** I correct a value, **Then** both the original and corrected value show (before → after) and the value is marked as manually edited.

---

### User Story 4 - Re-send after correction (Priority: P2)

A user discovers an error in a record that was already delivered, corrects the values, and re-sends the updated payload to the configured endpoint(s). The system logs the re-send as a new delivery attempt in the existing delivery history.

**Why this priority**: Closes the loop on corrections for already-delivered data. Depends on Story 3 (correction) and the existing re-send mechanics.

**Independent Test**: Correct a value on a sent record, trigger re-send, and verify the endpoint receives the corrected payload and the delivery history gains a new attempt.

**Acceptance Scenarios**:

1. **Given** a record was already sent and I later find an error, **When** I correct the values, **Then** I can re-send the updated payload to the configured endpoint(s), and the system logs a new delivery attempt.

---

### User Story 5 - Audit trail visible in the product (Priority: P2)

Every review action leaves a trace a user can inspect inside the product (not only the admin panel): who acted, when, what changed (before → after), the action type (confirm, correct, reject, re-send), and whether the record was originally flagged for low confidence.

**Why this priority**: Accountability for a workflow that gates data flowing into financial/ERP systems. Meaningful once Stories 2–4 produce actions to audit.

**Independent Test**: Perform each review action on a record and verify the record's audit view shows actor, timestamp, action type, before → after values, and the original flag state.

**Acceptance Scenarios**:

1. **Given** a record is confirmed, corrected, rejected, or re-sent, **Then** the system records who did it, when, what changed (before → after), the action type, and whether the record was originally flagged for low confidence.
2. **Given** I am a product user (not an admin-panel user), **Then** I can view this history in the product.

---

### User Story 6 - Signal low confidence in the UI (Priority: P3)

When a user views a record, the UI highlights the values that fall below the configured threshold, shows which threshold applied, and recommends what to do ("review these values before sending").

**Why this priority**: Guides the reviewer's attention inside a record. Valuable polish on top of Stories 1–2, but flagging and holding work without it.

**Independent Test**: Open a flagged record and verify low-confidence values are visually highlighted, the applied threshold is visible, and a recommendation is shown.

**Acceptance Scenarios**:

1. **Given** I view a record, **Then** the UI highlights low-confidence values against the configured threshold, exposes the applied threshold, and recommends an action.

---

### User Story 7 - Reviewer assignment (Priority: P3, dependent story)

An admin assigns reviewers to a document type (working assumption; the routing axis is still open — see OQ-5), choosing from existing org/workspace members. Reviewers see a Needs-Review queue in the existing records list filtered to their assigned document types, with a pending count, and can Confirm / Correct / Reject those records. Reviewers are notified in-app and by email. When several reviewers share a document type, the queue is shared: any of them can act, and the system locks a record while one edits it so edits do not collide. If a document type has no reviewers, workspace admins/managers see its Needs-Review records so none get stuck.

**Why this priority**: Depends on the Needs Review status and review actions from Stories 1–2. Without assignment, review still works — everyone with access sees the queue — so this ships last.

**Independent Test**: Assign a reviewer to a document type, produce a Needs-Review record for it, and verify the reviewer sees it in their queue with a pending count and receives a notification, while a non-assigned member does not see it in their queue.

**Acceptance Scenarios**:

1. **Given** I manage a document type, **When** I open its config, **Then** I can add and remove reviewers, chosen from existing org/workspace members.
2. **Given** I assign a reviewer, **Then** they can view and act (Confirm / Correct / Reject) on that document type's Needs-Review records.
3. **Given** I am a reviewer, **Then** I see a Needs-Review queue in the existing records list, filtered to my assigned document types, with a pending count.
4. **Given** several reviewers share a document type, **Then** the queue is shared: any of them can act, and the system locks a record while one of them edits it.
5. **Given** a record needs review, **Then** its assigned reviewers are notified in-app and by email.
6. **Given** a document type has no reviewers assigned, **Then** workspace admins/managers see its Needs-Review records.

---

### Edge Cases

- Extraction produces no confidence score at all → record is flagged Needs Review with a "confidence unavailable" message (covered by US1).
- A record is corrected while a send/retry is already in flight → the delivery history must make clear which payload version each attempt carried.
- Two reviewers open the same record → the lock in US7 prevents colliding edits; behaviour of the lock (hard vs. soft, timeout) is OQ-8.
- The threshold is changed after records were extracted → existing statuses are not retroactively recalculated; the new threshold applies to future extractions (assumption A4).
- A rejected record → excluded from sending permanently; rejection is recorded in the audit trail.
- A reviewer is removed from a document type while holding a record lock → the lock must release so the record does not get stuck.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST mark a record Needs Review when its overall confidence falls below the document type's configured threshold.
- **FR-002**: System MUST mark a record Needs Review when a field's confidence falls below the threshold (scope of "which fields" per OQ-2).
- **FR-003**: System MUST store and display the reason a record was flagged (which rule triggered it).
- **FR-004**: System MUST mark a record Needs Review when it cannot compute confidence, with a plain "confidence unavailable" message.
- **FR-005**: System MUST NOT automatically send a Needs-Review record; the Needs Review status (not the raw score) drives the hold.
- **FR-006**: System MUST continue to send non-flagged records automatically where automatic REST output is configured.
- **FR-007**: Users MUST be able to Confirm & Send, Correct, or Reject a held record, and the UI MUST explain why a send was blocked.
- **FR-008**: Users MUST be able to correct values on any record at any time, independent of confidence and of sent status.
- **FR-009**: System MUST retain and display both original and corrected values (before → after) and mark corrected values as manually edited.
- **FR-010**: System MUST make an unsent, corrected record eligible to send.
- **FR-011**: Users MUST be able to re-send a corrected, already-sent record; the system MUST log the re-send as a new delivery attempt.
- **FR-012**: System MUST record for every confirm/correct/reject/re-send: actor, timestamp, action type, changed values (before → after), and whether the record was originally flagged.
- **FR-013**: Users MUST be able to view that audit history in the product, not only in the admin panel.
- **FR-014**: Record detail UI MUST highlight values below the applied threshold, expose the threshold value, and recommend a review action.
- **FR-015**: Admins MUST be able to add and remove reviewers on a document type, chosen from existing org/workspace members (routing axis per OQ-5).
- **FR-016**: Assigned reviewers MUST be able to view and act on their document types' Needs-Review records.
- **FR-017**: Reviewers MUST see a Needs-Review queue in the existing records list, filtered to their assignments, with a pending count.
- **FR-018**: System MUST notify assigned reviewers in-app and by email when records need their review (cadence per OQ-7).
- **FR-019**: System MUST prevent colliding edits when multiple reviewers share a queue, by locking a record while one reviewer edits it (lock semantics per OQ-8).
- **FR-020**: When a document type has no reviewers, its Needs-Review records MUST be visible to workspace admins/managers.

### Key Entities

- **Extraction Record**: an extracted document instance; carries overall confidence, status (including Needs Review), a flag reason, and a delivery history. Gains: hold behaviour, review actions, audit history.
- **Extracted Field Value**: a single extracted value with its own confidence; gains an original-vs-corrected pair and a manually-edited marker.
- **Confidence Threshold**: per-document-type (and per-field) setting that already exists; gains enforcement.
- **Review Action**: a confirm, correct, reject, or re-send event with actor, timestamp, and before → after payload; the unit of the audit trail.
- **Reviewer Assignment**: a link between an org/workspace member and a document type (working assumption) granting queue visibility and review actions.
- **Record Lock**: a temporary claim on a record while a reviewer edits it.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Zero records below the configured threshold are delivered downstream without an explicit human confirm.
- **SC-002**: 100% of records the system cannot score are held for review instead of sent.
- **SC-003**: A reviewer can find, open, correct, and release a flagged record without leaving the product (no admin-panel round trip).
- **SC-004**: Every review action on a record is reconstructable from the in-product audit view: actor, time, action, and value changes.
- **SC-005**: A corrected, re-sent record reaches the downstream endpoint with the corrected payload, and the delivery history distinguishes the attempts.
- **SC-006**: An assigned reviewer sees a new Needs-Review record in their queue and receives a notification without anyone forwarding it manually.

## Open Questions *(kept open on purpose)*

Per the requester, these mirror the open questions on the Confluence page and stay unresolved in this spec. They will be settled with the team and folded back in later; none blocks drafting the plan, but OQ-2 and OQ-5 affect scope and should be settled before implementation of the affected stories.

- **OQ-1**: Overall confidence calculation: average of all fields (current behaviour), the lowest field, or a critical-weighted score? An average hides a single badly-extracted field.
- **OQ-2**: Needs Review trigger: does low confidence on **any** field flag review, or **critical fields only**? *(Affects FR-002.)*
- **OQ-3**: After a correction, what confidence does the value show: 100%, "not applicable", or the original score plus an edited marker? And does correcting clear Needs Review on its own, or is an explicit Confirm still required?
- **OQ-4**: ~~Where does the threshold setting live?~~ Resolved on the source page: in the document type, to keep it simple.
- **OQ-5**: Reviewer routing axis: by **document type** (working assumption in US7), by **data source/mailbox**, **both**, or a **workspace-wide pool**? *(Affects FR-015–FR-017.)*
- **OQ-6**: Permission model: a dedicated Reviewer role, or a review capability on existing roles? And who can assign reviewers?
- **OQ-7**: Email cadence: one email per record, or a batched digest?
- **OQ-8**: Lock behaviour: hard lock or soft "being reviewed" flag, and what timeout releases it?
- **OQ-9**: Round-robin auto-assignment: do we add it, and when? (Out of scope for this spec as written.)
- **OQ-10**: Can an admin bulk-assign reviewers to several document types at once?

## Assumptions

- **A1**: The existing confidence scoring, Needs Review status, records list/detail views, and REST delivery (send, retry, re-send, attempt history) are reused, not rebuilt.
- **A2**: The confidence threshold lives on the document type (resolved on the source page; per-field thresholds already stored may refine it later).
- **A3**: Reviewer assignment (US7) targets the document-type level as a working assumption until OQ-5 is decided.
- **A4**: Changing a threshold applies to future extractions only; existing record statuses are not recalculated retroactively.
- **A5**: "Reject" excludes a record from delivery; rejected data is retained for audit, not deleted.
- **A6**: Reviewers are existing org/workspace members; no external reviewer accounts.
- **A7**: The two stories ship in dependency order: confidence & control step first, reviewer assignment second.
