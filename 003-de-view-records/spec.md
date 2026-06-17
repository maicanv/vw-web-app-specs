# Feature Specification: Document Entry — View Records

**Feature Branch**: `003-de-view-records`

**Created**: 2026-06-16

**Status**: Draft

## Clarifications

### Session 2026-06-16

- Q: Is the detail view read-only or does this user story include editing extracted values? → A: Read-only. The detail view is for viewing and validation only. Editing/correction of extracted values is out of scope for this user story and will be addressed in a dedicated future story.
- Q: Is filtering by Document Type field values (FR-014) in scope for this story? → A: Deferred. Per-field value filtering (e.g., "Invoice Total > 5000") is a planned enhancement but out of scope for this user story due to complexity and sprint size.
- Q: Should search cover extracted field values (FR-015) or sender/subject only? → A: Sender and email subject only. Search within extracted field values is deferred to a future story alongside per-field filtering.
- Q: Who within a workspace can see records? → A: All workspace members with the Document Entry subscription feature enabled. No per-role or per-user record restriction.
- Q: What happens when a source attachment file is no longer available? → A: The source entry is shown with a "file unavailable" notice; extracted field values remain fully visible; inline view and download are disabled.

**Source**: [Confluence — [DE] US: View records](https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/325877781/DE+US+View+records)

---

## User Scenarios & Testing

### User Story 1 — Browse Extraction Records (Priority: P1)

As a user with the Document Entry feature enabled, I want to see all extraction records generated from my incoming emails so that I can track what data has been extracted and its status.

**Why this priority**: The list view is the entry point for all record-related workflows. Without it, no other record functionality is accessible.

**Independent Test**: A user can navigate to Document Entry > Records, see a paginated list of records with correct metadata columns, and verify the page is inaccessible (hidden or upsell shown) when the feature is not part of their subscription.

**Acceptance Scenarios**:

1. **Given** the user's subscription includes Document Entry, **when** they navigate to Document Entry > Records in the side menu, **then** they see a list of records with the following columns: received date/time, document type, email subject, sender, processing status, delivery status, and overall confidence indicator.
2. **Given** the user's subscription does not include Document Entry, **when** they navigate to the Document Entry section, **then** the Records sub-section is either hidden from the side menu or replaced with an upsell prompt.
3. **Given** no records exist, **when** the user opens the Records page, **then** an appropriate empty-state message is shown.

---

### User Story 2 — Filter, Search, and Sort Records (Priority: P2)

As a user, I want to narrow down the list of records using filters, search, and sorting so that I can quickly locate specific records without scrolling through the full list.

**Why this priority**: Filtering and search become essential as the number of records grows; however, a basic list is still useful without them.

**Independent Test**: A user can apply a filter (e.g., processing status = "Needs review") and see only matching records; remove the filter and see all records; search by sender keyword and see only matching results; sort by date descending and verify order.

**Acceptance Scenarios**:

1. **Given** the Records page is open, **when** the user applies a Document Type filter, **then** only records of that type are shown.
2. **Given** the Records page is open, **when** the user filters by processing status (Extracted / Needs review / Failed / Sent), **then** only records matching the selected status are shown.
3. **Given** the Records page is open, **when** the user filters by delivery status (Pending / Warning / Success / Error), **then** only records matching that status are shown.
4. **Given** the Records page is open, **when** the user sets a confidence threshold filter (e.g., "below 80%"), **then** only records with an overall confidence below that threshold are shown.
5. **Given** the Records page is open, **when** the user applies a date range filter, **then** only records received within that range are shown.
6. **Given** the Records page is open, **when** the user searches by sender email or subject keywords, **then** only records matching the search query are shown.
7. **Given** the Records page is open, **when** the user sorts by date & time, **then** records reorder accordingly (ascending or descending).

---

### User Story 3 — Inspect a Record Detail (Priority: P1)

As a user, I want to open a specific record and see the full extracted data side-by-side with the source document so that I can validate accuracy and understand what will be sent downstream. This view is read-only; editing extracted values is out of scope for this user story.

**Why this priority**: The detail view is where the primary value of Document Entry is delivered — users validate, correct, and approve extracted data before it flows to downstream systems.

**Independent Test**: A user can click any record in the list, see the detail view with all email metadata, source attachments (viewable and downloadable), extracted field values with confidence scores, and highlighted issues (missing critical fields, low-confidence values).

**Acceptance Scenarios**:

1. **Given** the user clicks a record, **when** the detail view opens, **then** they can see: email subject, sender, received date/time, document type, action trace ID, and a link to the related history item.
2. **Given** the detail view is open, **when** the record was processed from one or more attachments (PDF, Word, Excel), **then** the extraction sources are listed and each source can be viewed inline and downloaded.
3. **Given** the detail view is open, **when** the extraction sources are displayed, **then** the source content (e.g., PDF or email body) is shown side-by-side with the extracted field values for direct visual comparison.
4. **Given** the detail view is open, **when** extracted field values are displayed, **then** each value shows its data-level confidence score.
5. **Given** a record has missing values for fields marked as critical, **when** the detail view is open, **then** those missing fields are visually highlighted.
6. **Given** a record has extracted values with confidence below the configured threshold, **when** the detail view is open, **then** those values are visually highlighted as low-confidence.

---

### Edge Cases

- If a source attachment file is no longer available (e.g., deleted from storage), the detail view still renders with full extracted field data; the source entry shows a "file unavailable" notice and inline view / download are disabled.
- How does the list behave when there are thousands of records — is pagination or infinite scroll used?
- What is shown in the confidence column when confidence cannot be computed (e.g., extraction failed)?
- What does the detail view show when a record was processed from the email body only (no attachments)?
- How are records that belong to multiple document types (if ever possible) represented?

---

## Requirements

### Functional Requirements

- **FR-001**: The Records page MUST only be accessible to users whose subscription includes the Document Entry feature; all others see the section hidden or an upsell prompt. All workspace members with this feature enabled have equal access to all records in the workspace — no per-role restriction applies.
- **FR-002**: The Records list MUST display each record with: received date/time, document type, email subject, sender, processing status, delivery status, and overall confidence indicator.
- **FR-003**: Processing status MUST support at least four values: Extracted, Needs review, Failed, Sent.
- **FR-004**: Delivery status MUST support at least four values: Pending, Warning, Success, Error.
- **FR-005**: The Records list MUST support filtering by: document type, processing status, delivery status, confidence threshold, and date range.
- **FR-006**: The Records list MUST support keyword search by sender and email subject.
- **FR-007**: The Records list MUST support sorting by received date & time (ascending and descending).
- **FR-008**: The Record detail view MUST display email metadata: subject, sender, received date/time, document type, action trace ID, and a link to the associated history item.
- **FR-009**: The Record detail view MUST list all extraction sources (email body and/or attachments); each source file MUST be viewable inline and downloadable.
- **FR-010**: The Record detail view MUST present extraction source content side-by-side with extracted field values to enable visual comparison.
- **FR-011**: Each extracted field value in the detail view MUST display its individual confidence score.
- **FR-012**: The system MUST highlight missing values for fields configured as critical on the associated document type.
- **FR-013**: The system MUST highlight extracted values whose confidence score falls below the configured threshold.
- **FR-014**: ~~Filtering by Document Type field values~~ — **Deferred**. Per-field value filtering (e.g., "show invoices where Supplier = X") is out of scope for this story; planned as a future enhancement.
- **FR-015**: ~~Search by extracted field values~~ — **Deferred**. Search is limited to sender email and subject keywords (FR-006). Full-text search across extracted field values is out of scope for this story.
- **FR-016**: When a source attachment file is unavailable, the detail view MUST still display all extracted field values. The extraction source entry MUST show a "file unavailable" notice; inline preview and download MUST be disabled for that source.

### Key Entities

- **Extraction Record**: Represents one processing result for a single email. Carries processing status, delivery status, overall confidence, and links to the source email and document type.
- **Extracted Field Value**: A single field's extracted value from a record, including the field name, raw extracted value, normalised value, and per-field confidence score.
- **Document Type**: The template that defines the expected fields and which fields are marked critical, used to interpret and validate a record's extracted data.
- **Extraction Source**: The source artefact used for extraction — either the email body or an attached file (PDF, Word, Excel). Carries a reference to the stored file for viewing and download.

---

## Success Criteria

### Measurable Outcomes

- **SC-001**: A user can open the Records list and locate a specific record by filtering and/or searching in under 30 seconds, even when the list contains 500+ records.
- **SC-002**: The Records list loads within 2 seconds for a workspace with up to 1,000 records.
- **SC-003**: A user can open a record detail view and visually compare the source document to the extracted values without leaving the page.
- **SC-004**: 100% of records with missing critical fields are visually flagged in the detail view.
- **SC-005**: 100% of records with any value below the confidence threshold are visually flagged in the detail view.
- **SC-006**: Users without the Document Entry subscription feature cannot access the Records section (0% unauthorised access).

---

## Assumptions

- Extraction records are already being generated by the Document Entry action pipeline (VWE-1388/1389); this feature adds the viewing surface only — no changes to how records are created.
- The confidence threshold used for highlighting is configured at the document type or workspace level (not per-user).
- Sorting is initially limited to date & time; additional sort columns are out of scope for this user story.
- The side-by-side source comparison displays a rendered/preview version of the attachment where technically feasible (e.g., PDF preview); a download fallback is always available.
- The Records page is scoped to the current workspace; cross-workspace record views are out of scope. All workspace members with Document Entry enabled see all records equally — no per-role or per-user visibility restriction.
- Pagination strategy (page-based vs. infinite scroll) is an implementation detail; the spec requires that large lists remain performant and navigable.
- Filtering by extracted field values (per-field filtering within a document type) is deferred to a future story due to backend indexing and dynamic UI complexity.
- Search within extracted field values is deferred to a future story; current search scope is sender and email subject only (FR-006).
- The detail view is read-only. Users can validate extracted data but cannot edit it within this user story. Correction capability is explicitly deferred to a future story.
