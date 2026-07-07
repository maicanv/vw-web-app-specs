# Feature Specification: Extraction Review — Confidence & Control Step

**Feature Branch**: `006-extraction-review`

**Created**: 2026-07-06

**Updated**: 2026-07-07 (source story revised; all open questions resolved, new scope added; Record UID and Post-review delivery method/UID selection dropped from scope — reviewed records reuse the route's normal delivery unchanged)

**Status**: Draft

**Input**: Confluence story "[DE] US: Extraction Review" (https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/341803023/DE+US+Extraction+Review). The source page now carries two stories — the Confidence & Control Step and the dependent Reviewer Assignment story — and all of its open questions are answered. This spec folds those answers into the requirements; see [Resolved Decisions](#resolved-decisions).

## What already works today (context, not in scope)

- Every extracted value carries a confidence score, and each record carries an overall confidence score. Both show in the records list and detail views.
- Records can carry a **Needs Review** status, exposed through list filters and status badges.
- A confidence threshold can be stored per document type (and per field). The system stores it but does not act on it yet.
- Records can be sent to downstream systems over REST with automatic or manual sending, retry, and re-send, plus a history of delivery attempts.

This spec covers the remaining scope: a per-document-type review configuration that drives flagging, holding flagged records, a dedicated review queue, value correction, re-send, a platform-wide audit surface, reviewer assignment with a dedicated Reviewer role, and a set of changes to existing delivery settings.

## Clarifications

### Session 2026-07-07

- Q: How do multiple enabled review rules combine when deciding if a record needs review? → A: OR — a record is flagged if any enabled rule matches.
- Q: How does the dedicated Reviewer role sit in the RBAC model? → A: One global Reviewer role; the per-document-type assignment grants and scopes it.
- Q: Which sends use the Post-review delivery method vs the route's normal delivery? → A: Superseded by A9 — the post-review method/UID feature was dropped from scope; all sends, reviewed or not, use the route's normal delivery.
- Q: Is Reject terminal or reversible? → A: Reversible — a rejected record can be brought back into review (e.g. via Select for review) and corrected/re-sent. Approving/rejecting a *correction* is a separate future-version flow, out of scope here.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Configure which records go to review (Priority: P1)

An admin editing a document type sees a new **Review Records** step that explains record review and offers a toggle to turn review on or off for that document type (on by default). When review is on, the admin selects which records the system routes to review by enabling any combination of rules: every record, records whose average confidence is below a chosen number, records from a chosen sender-domain list, records where a critical-marked field is below a chosen number, or records where any field is below a chosen number.

**Why this priority**: This configuration is what turns the stored threshold into behaviour. Every downstream story (flagging, holding, queue) reads from it, so it comes first.

**Independent Test**: Open a document type's Review Records step, toggle review on, enable the "average confidence below N" rule with N set, save, and verify the setting persists and is readable by the flagging step.

**Acceptance Scenarios**:

1. **Given** I create or edit a document type, **Then** an Review Records step explains record review and defaults the review toggle to on.
2. **Given** review is enabled, **Then** I can enable "all records go to review".
3. **Given** review is enabled, **Then** I can enable "records with average confidence below N go to review" and set N.
4. **Given** review is enabled, **Then** I can enable "records from a chosen sender-domain list go to review".
5. **Given** review is enabled, **Then** I can enable "records with a critical field below N go to review" and set N.
6. **Given** review is enabled, **Then** I can enable "records with any field below N go to review" and set N.
7. **Given** review is turned off for a document type, **Then** its records are never auto-flagged for review.

---

### User Story 2 - Flag records for review by the configured rules (Priority: P1)

After extraction, the system evaluates the document type's enabled review rules against the record and marks matching records as Needs Review, together with the reason the flag fired. Records the system could not score at all get the same treatment with a "confidence unavailable" reason.

**Why this priority**: This is the trigger for the review workflow. Without flagging, the queue, hold, and review actions have nothing to act on.

**Independent Test**: With a rule enabled on a document type, run an extraction that matches the rule and verify the record shows as Needs Review with the matching reason; run one that does not match and verify it stays unflagged.

**Acceptance Scenarios**:

1. **Given** a record has been extracted, **When** its overall (average) confidence is below the configured number, **Then** the record is marked Needs Review.
2. **Given** a record has been extracted, **When** a field's confidence is below the configured number and that rule is enabled, **Then** the record is marked Needs Review.
3. **Given** a record is marked Needs Review, **Then** the reviewer sees which rule triggered the flag, and the record appears in status indicators.
4. **Given** the system cannot compute confidence for a record, **Then** it marks the record Needs Review and shows a plain "confidence unavailable" message.

---

### User Story 3 - Hold flagged records before sending (control step) (Priority: P1)

A record flagged Needs Review is held: the system does not deliver it downstream until a person acts. The reviewer opens the record and chooses one of three actions: Confirm & Send, Correct, or Reject. Unflagged records with no blocking issues continue to send automatically where automatic REST output is configured.

**Why this priority**: The business goal is that wrong data never reaches downstream systems (e.g. an ERP). Flagging without a hold does not deliver that promise.

**Independent Test**: With a REST output configured for automatic sending, produce one flagged and one unflagged record. Verify the unflagged record sends and the flagged record waits until a reviewer confirms, corrects, or rejects it.

**Acceptance Scenarios**:

1. **Given** a record is Needs Review, **Then** the system does not send it and waits for a user action. The Needs Review status drives the hold, not the raw confidence score.
2. **Given** I am reviewing a held record, **Then** I can Confirm & Send, Correct, or Reject it.
3. **Given** a record is not Needs Review and has no blocking issues, **Then** it can be sent automatically when REST output is configured.
4. **Given** the system blocks a send, **Then** the UI explains why and offers Confirm / Correct / Reject.

---

### User Story 4 - Dedicated Needs Review queue in the sidebar (Priority: P2)

A new **Needs Review** entry in the document entry sidebar lists every record that needs review and shows a pending count on the entry itself. Because this dedicated surface exists, the "Needs review" filter is removed from the extraction record list. On a record that was not flagged (for example a false positive), a **Select for review** control next to the record adds it to the Needs Review list so a user can review and correct it.

**Why this priority**: Gives reviewers one place to work the backlog and a running count. Depends on the Needs Review status from Stories 2–3.

**Independent Test**: Produce several Needs-Review records and verify they all appear under the sidebar Needs Review entry with a correct pending count; verify the old list filter is gone; use Select for review on an unflagged record and verify it joins the queue.

**Acceptance Scenarios**:

1. **Given** records need review, **Then** the sidebar Needs Review entry lists them and shows the pending count.
2. **Given** the Needs Review section exists, **Then** the "Needs review" filter no longer appears in the extraction record list.
3. **Given** a record was not flagged, **When** I use Select for review on it, **Then** it is added to the Needs Review list.

---

### User Story 5 - Correct values on any record (Priority: P2)

A user can fix extracted values on any record regardless of its confidence, including false positives where the confidence looked fine and records that were already sent. Corrections are saved, the old and new value both remain visible (before → after), and the value is marked as manually edited. Its confidence score is kept and shown with the edited marker. An unsent corrected record becomes eligible to send.

**Why this priority**: Correction is one of the three review actions, and it also stands alone: users need it even for records the system never flagged.

**Independent Test**: Open any record (flagged or not), change a value, and verify the change persists, shows before → after, keeps its confidence with an edited marker, and (for unsent records) the record becomes sendable.

**Acceptance Scenarios**:

1. **Given** any record, including one already sent, **When** I correct one or more values, **Then** the changes are saved, and if the record has not been sent it becomes eligible to send.
2. **Given** I correct a value, **Then** both the original and corrected value show (before → after), the value keeps its confidence score, and it is marked as manually edited.

---

### User Story 6 - Re-send after correction (Priority: P2)

A user discovers an error in a record that was already delivered, corrects the values, and re-sends the updated payload to the configured endpoint(s), subject to the output route's resend policy. The system logs the re-send as a new delivery attempt in the existing delivery history.

**Why this priority**: Closes the loop on corrections for already-delivered data. Depends on Story 5 and the existing re-send mechanics.

**Independent Test**: Correct a value on a sent record whose route allows resend, trigger re-send, and verify the endpoint receives the corrected payload and the delivery history gains a new attempt. Repeat on a route that blocks resend and verify the re-send is disallowed.

**Acceptance Scenarios**:

1. **Given** a record was already sent, its route allows resend, and I later find an error, **When** I correct the values, **Then** I can re-send the updated payload to the configured endpoint(s), and the system logs a new delivery attempt.
2. **Given** a record's route blocks resend, **Then** I cannot re-send an already successfully delivered record.

---

### User Story 7 - Platform-wide audit section (Priority: P2)

A new **Audit** section under the Organisation area of the sidebar lets a user inspect what happened across subsystems, not only in the admin panel. The user filters by subsystem; **Extraction Record modification** is the first subsystem shown. For each audited action the section shows who acted, when, what changed (before → after), the action type (confirm, correct, reject, re-send), and whether the record was originally flagged for low confidence.

**Why this priority**: Accountability for a workflow that gates data flowing into financial/ERP systems. Meaningful once Stories 3–6 produce actions to audit.

**Independent Test**: Perform each review action on a record, open the Audit section, filter to Extraction Record modification, and verify actor, timestamp, action type, before → after values, and the original flag state show for each action.

**Acceptance Scenarios**:

1. **Given** a record is confirmed, corrected, rejected, or re-sent, **Then** the system records who did it, when, what changed (before → after), the action type, and whether the record was originally flagged for low confidence.
2. **Given** I am a product user (not an admin-panel user), **Then** I can view this history in the Audit section.
3. **Given** I open the Audit section, **Then** I can filter by subsystem, with Extraction Record modification available as the first subsystem.

---

### User Story 8 - Signal low confidence in the record UI (Priority: P3)

When a user views a record, the UI highlights the values that fall below the configured threshold, shows which threshold applied, and recommends what to do ("review these values before sending").

**Why this priority**: Guides the reviewer's attention inside a record. Valuable polish on top of Stories 2–3, but flagging and holding work without it.

**Independent Test**: Open a flagged record and verify low-confidence values are visually highlighted, the applied threshold is visible, and a recommendation is shown.

**Acceptance Scenarios**:

1. **Given** I view a record, **Then** the UI highlights low-confidence values against the configured threshold, exposes the applied threshold, and recommends an action.

---

### User Story 9 - Reviewer assignment with a dedicated Reviewer role (Priority: P3, dependent story)

An admin assigns reviewers to a document type, choosing from existing org/workspace members; assigned users hold a dedicated Reviewer role. Reviewers see the Needs-Review queue filtered to their assigned document types, with a pending count, and can Confirm / Correct / Reject those records. When several reviewers share a document type, the queue is shared: any of them can act, and the system places a hard lock on a record while one edits it, releasing after a 5-minute timeout. If a document type has no reviewers, workspace admins/managers see its Needs-Review records so none get stuck.

**Why this priority**: Depends on the Needs Review status and review actions from Stories 2–3. Without assignment, review still works — everyone with access sees the queue — so this ships last.

**Independent Test**: Assign a reviewer to a document type, produce a Needs-Review record for it, and verify the reviewer sees it in their queue with a pending count while a non-assigned member does not; open the record as one reviewer and verify a second reviewer is blocked by the lock until it is released or times out.

**Acceptance Scenarios**:

1. **Given** I manage a document type, **When** I open its config, **Then** I can add and remove reviewers, chosen from existing org/workspace members, and assignment grants them the Reviewer role for that document type.
2. **Given** I assign a reviewer, **Then** they can view and act (Confirm / Correct / Reject) on that document type's Needs-Review records.
3. **Given** I am a reviewer, **Then** I see the Needs-Review queue filtered to my assigned document types, with a pending count.
4. **Given** several reviewers share a document type, **Then** the queue is shared: any of them can act, and a hard lock holds a record while one of them edits it, releasing after a 5-minute timeout.
5. **Given** a document type has no reviewers assigned, **Then** workspace admins/managers see its Needs-Review records.

---

### Edge Cases

- Extraction produces no confidence score at all → record is flagged Needs Review with a "confidence unavailable" message (covered by US2).
- A record is corrected while a send/retry is already in flight → the delivery history must make clear which payload version each attempt carried.
- Two reviewers open the same record → the hard lock (US9) prevents colliding edits and releases after 5 minutes.
- The review configuration is changed after records were extracted → existing statuses are not retroactively recalculated; the new configuration applies to future extractions (assumption A4).
- A rejected record → excluded from sending while rejected; a user can bring it back into review (Select for review), correct it, and re-send. Every rejection and re-open is recorded in the audit trail.
- A reviewer is removed from a document type while holding a record lock → the lock must release so the record does not get stuck.
- A route with resend blocked → re-send is refused for an already successfully delivered record even after correction (US6).
- A document type with review disabled → no records are auto-flagged, but a user can still pull a record into review via Select for review (US4).

## Requirements *(mandatory)*

### Functional Requirements

#### Review configuration (document type)

- **FR-001**: The document type create/edit flow MUST include an Review Records step that explains record review and exposes an on/off toggle, defaulting to on.
- **FR-002**: When review is enabled, users MUST be able to enable any combination of these review rules: all records; average confidence below a set number; records from a set sender-domain list; a critical field below a set number; any field below a set number. Enabled rules combine with OR: a record is flagged Needs Review if any enabled rule matches.
- **FR-003**: When review is disabled for a document type, the system MUST NOT auto-flag its records.

#### Flagging

- **FR-004**: System MUST evaluate the document type's enabled review rules after extraction and mark matching records Needs Review.
- **FR-005**: System MUST store and display the reason a record was flagged (which rule triggered it).
- **FR-006**: System MUST mark a record Needs Review when it cannot compute confidence, with a plain "confidence unavailable" message.
- **FR-007**: System MUST calculate overall confidence as the average of field confidences.

#### Hold and review actions

- **FR-008**: System MUST NOT automatically send a Needs-Review record; the Needs Review status (not the raw score) drives the hold.
- **FR-009**: System MUST continue to send non-flagged records automatically where automatic REST output is configured.
- **FR-010**: Users MUST be able to Confirm & Send, Correct, or Reject a held record, and the UI MUST explain why a send was blocked.

#### Needs Review queue

- **FR-011**: The document entry sidebar MUST include a Needs Review entry that lists all records needing review and shows a pending count.
- **FR-012**: The "Needs review" filter MUST be removed from the extraction record list once the Needs Review section exists.
- **FR-013**: Users MUST be able to add an unflagged record to the Needs Review list via a Select for review control on the record.

#### Correction and re-send

- **FR-014**: Users MUST be able to correct values on any record at any time, independent of confidence and of sent status.
- **FR-015**: System MUST retain and display both original and corrected values (before → after), keep the value's confidence score, and mark corrected values as manually edited.
- **FR-016**: System MUST make an unsent, corrected record eligible to send.
- **FR-017**: Users MUST be able to re-send a corrected, already-sent record when its route allows resend; the system MUST log the re-send as a new delivery attempt.
- **FR-018**: System MUST refuse re-send of an already successfully delivered record when its route blocks resend.

#### Audit

- **FR-019**: System MUST record for every confirm/correct/reject/re-send: actor, timestamp, action type, changed values (before → after), and whether the record was originally flagged.
- **FR-020**: A platform-wide Audit section MUST exist under the Organisation area of the sidebar, viewable by product users (not only the admin panel).
- **FR-021**: The Audit section MUST let users filter by subsystem, with Extraction Record modification available as the first subsystem.

#### Signalling

- **FR-022**: Record detail UI MUST highlight values below the applied threshold, expose the threshold value, and recommend a review action.

#### Reviewer assignment

- **FR-023**: Admins MUST be able to add and remove reviewers on a document type, chosen from existing org/workspace members. A single dedicated Reviewer role exists in the role set; the per-document-type assignment grants that role and scopes it to the assigned document type (role = capability, assignment = scope).
- **FR-024**: Assigned reviewers MUST be able to view and act on their document types' Needs-Review records.
- **FR-025**: Reviewers MUST see the Needs-Review queue filtered to their assignments, with a pending count.
- **FR-026**: System MUST prevent colliding edits when multiple reviewers share a queue, using a hard lock that holds a record while one reviewer edits it and releases after a 5-minute timeout.
- **FR-027**: When a document type has no reviewers, its Needs-Review records MUST be visible to workspace admins/managers.

#### Changes to existing delivery settings

- **FR-028**: The "Repeat policy" setting MUST be renamed to "Resend policy" with options Allow and Block; Block MUST disallow re-sending an already successfully delivered record from the record detail delivery section.
- **FR-029**: The "Delivery Mode" policy MUST be removed; manual delivery is now expressed by configuring a document type so all its records go to review first.

### Key Entities

- **Extraction Record**: an extracted document instance; carries overall confidence, status (including Needs Review), a flag reason, and a delivery history. Gains: hold behaviour, review actions, audit history, and manual add-to-review.
- **Extracted Field Value**: a single extracted value with its own confidence; gains an original-vs-corrected pair, a kept confidence score, and a manually-edited marker. A field may be marked critical.
- **Review Configuration**: per-document-type settings (the Review Records step): the review on/off toggle plus the set of enabled review rules and their parameters (average threshold, domain list, critical-field threshold, any-field threshold).
- **Review Action**: a confirm, correct, reject, or re-send event with actor, timestamp, and before → after payload; the unit of the audit trail.
- **Reviewer Assignment**: a link between an org/workspace member and a document type that grants the Reviewer role, queue visibility, and review actions.
- **Record Lock**: a hard, temporary claim on a record while a reviewer edits it, releasing after 5 minutes.
- **Output Route delivery settings**: resend policy (Allow/Block). A reviewed (confirmed or corrected) record is sent with the exact same delivery method and UID handling as the route's normal flow — no separate post-review method or UID is introduced.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Zero records that match a document type's enabled review rules are delivered downstream without an explicit human confirm.
- **SC-002**: 100% of records the system cannot score are held for review instead of sent.
- **SC-003**: A reviewer can find, open, correct, and release a flagged record from the Needs Review section without leaving the product (no admin-panel round trip).
- **SC-004**: Every review action on a record is reconstructable from the in-product Audit section: actor, time, action, and value changes.
- **SC-005**: A corrected, re-sent record reaches the downstream endpoint with the corrected payload, and the delivery history distinguishes the attempts.
- **SC-006**: An assigned reviewer sees a new Needs-Review record in their queue without anyone forwarding it manually.
- **SC-007**: With resend blocked on a route, no already-delivered record can be re-sent from the product.

## Resolved Decisions

The source page's open questions are all answered; the answers are folded into the requirements above and recorded here for traceability.

- **D1 (overall confidence)**: Keep the average of field confidences. (FR-007)
- **D2 (flag trigger)**: Offer both "any field below N" and "critical fields below N" as user-selectable rules in the Review Records step, alongside all-records, average-threshold, and domain-list rules. (FR-002)
- **D3 (threshold location)**: Lives on the document type. (FR-001, FR-002)
- **D4 (corrected value)**: Keep the original confidence score and add a manually-edited marker; an explicit Confirm is still required (correction alone does not clear Needs Review). (FR-015)
- **D5 (routing axis)**: Assign reviewers by document type. (FR-023)
- **D6 (permission model)**: A dedicated Reviewer role. (FR-023)
- **D7 (lock behaviour)**: Hard lock with a 5-minute timeout. (FR-026)
- **D8 (round-robin)**: Deferred to a later version; out of scope here.
- **D9 (bulk-assign)**: Not for now; out of scope here.

## Assumptions

- **A1**: The existing confidence scoring, Needs Review status, records list/detail views, and REST delivery (send, retry, re-send, attempt history) are reused, not rebuilt.
- **A2**: The review configuration lives on the document type (per-field thresholds already stored may refine it later).
- **A3**: Changing a review configuration applies to future extractions only; existing record statuses are not recalculated retroactively.
- **A4**: "Reject" excludes a record from delivery while it is rejected; the record can be brought back into review and corrected/re-sent, and rejected data is retained for audit, not deleted. Approve/reject of an individual correction is a future-version flow, out of scope here.
- **A5**: Reviewers are existing org/workspace members; no external reviewer accounts.
- **A6**: The two stories ship in dependency order: confidence & control step first, reviewer assignment second.
- **A7**: Reviewer notifications (in-app/email) are not part of this revision of the story; they are out of scope until re-added.
- **A8**: The platform-wide Audit section is scoped in this feature to the Extraction Record modification subsystem; other subsystems are additive later.
- **A9**: Record UID selection and a distinct post-review delivery method/UID are dropped from this revision as too much work for now; a reviewed (confirmed or corrected) record is delivered with the exact same method and UID handling the route already uses for its normal flow. Revisit if downstream systems need a different identifier or method after review.
