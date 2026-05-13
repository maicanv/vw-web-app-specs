# Feature Specification: Document Type Field Grouping

**Feature Branch**: `[TBD]`

**Jira**: [VWE-1496](https://frontiergroup.atlassian.net/browse/VWE-1496)

**Created**: 2026-05-12

**Status**: Draft

**Input**: User description: "https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/294420508/DE+US+Field+Grouping"

**Source**: [[DE] US: Field Grouping — Confluence](https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/294420508/DE+US+Field+Grouping)

**Blocks**: [DE] US: Data Output via REST API / MCP Server — this spec must ship first.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Define Field Groups on a Document Type (Priority: P1)

A user building a Document Type for structured documents (e.g. pickup-code manifests, invoices with line items) needs to model both header-level fields and one or more repeating sections of fields. Without this, extraction produces a flat wall of numbered fields that downstream systems cannot cleanly consume.

**Why this priority**: Core authoring capability — all other stories depend on groups existing. Without it, extraction and review features have nothing to display.

**Independent Test**: A user can open a Document Type wizard, define a repeatable group called "Pickup Codes" with a "Code" field inside it, save the Document Type, and confirm it is persisted.

**Acceptance Scenarios**:

1. **Given** I am on the Fields step of the Document Type wizard, **When** I click "Add Group", **Then** a group creation form appears with inputs for name, code, description, and kind (Repeatable / Single object).
2. **Given** I have created a Repeatable group, **When** I configure it, **Then** I can set a minimum and maximum number of items.
3. **Given** I have created a Single object group, **When** I configure it, **Then** I can mark it as optional.
4. **Given** I have existing header-level fields, **When** I select one, **Then** I can move it into a group; and vice versa — I can move a group field back to header level.
5. **Given** a Document Type has fields across header and groups, **When** I count them, **Then** the total cannot exceed 50 fields tree-wide (top-level + all group children).
6. **Given** I have a field in any context, **When** I view the field form, **Then** the description input is always visible (not collapsed).
7. **Given** a Document Type is active, **When** I try to rename a group code, delete a group, or remove a field from a group, **Then** the system prevents the action — same immutability rules as fields today.

---

### User Story 2 — Extract Grouped Fields from a Document (Priority: P1)

A user submits a document for extraction. The model must honour the group structure defined on the Document Type and return data shaped accordingly — repeatable groups as arrays, single object groups as nested objects.

**Why this priority**: Extraction is the core value — without correct grouped output, all downstream usage (review, API, output delivery) is broken.

**Independent Test**: Submit a real Mercitalia pickup-code manifest (5 pickup codes) against a Document Type with a repeatable "Pickup Codes" group; confirm the extraction result contains 5 items in that group.

**Acceptance Scenarios**:

1. **Given** a Document Type has a repeatable group, **When** extraction runs, **Then** the result contains one array item per occurrence found in the document.
2. **Given** a Document Type has a single object group, **When** extraction runs and the group is present, **Then** the result contains exactly one nested object for that group.
3. **Given** a Document Type has an optional single object group, **When** extraction runs and the group is absent from the document, **Then** the group is omitted from the result entirely.
4. **Given** a repeatable group has minimum items = 2 and extraction finds only 1, **When** the result is returned, **Then** a second item is padded with empty values to meet the minimum.
5. **Given** a flat-only Document Type with no groups, **When** extraction runs, **Then** the result is identical to today's flat output — no regressions.

**Extraction quality gate** — rollout blocked until fixture set passes:
- Repeatable group with multiple items (Mercitalia — 5 pickup codes).
- Repeatable group with single item (Hupac booking confirmation — still returned as an array).
- Same group repeated across multiple pages (Kombiverkehr delay notice — bilingual, same row on EN and DE page).
- Flat-only document — regression baseline.
- Single object group (sender block with name / address / VAT ID).

---

### User Story 3 — Review Grouped Extraction on the Record Detail Page (Priority: P2)

A reviewer opening an extracted record needs to see the grouped structure of the document — header fields first, then single object groups as cards, then repeatable groups as accordions — so they can verify and correct values in context.

**Why this priority**: Reviewers cannot meaningfully validate grouped data if the UI still shows a flat list. This directly affects review quality and user trust.

**Independent Test**: Open a record extracted against a Document Type with a repeatable "Invoice Lines" group; confirm the UI shows an accordion section with one tab per extracted line item, each with editable field-value pairs.

**Acceptance Scenarios**:

1. **Given** a record with header fields, **When** the detail page loads, **Then** header fields are shown first as a flat field-value grid with confidence chips — unchanged from today.
2. **Given** a record has a single object group, **When** the detail page loads, **Then** the group appears as a labeled card below header fields, with a group-level confidence chip and editable field-value pairs inside.
3. **Given** an optional single object group was not found in the document, **When** the detail page loads, **Then** the card still appears but shows "Not found in this document" with no fields listed.
4. **Given** a record has a repeatable group with 3 items, **When** the detail page loads, **Then** a labeled section shows item count, an accordion/tab strip (Item 1, Item 2, Item 3), and the first item is expanded by default.
5. **Given** a repeatable group item has a confidence score below threshold or a missing required field, **When** the accordion renders, **Then** that item's tab is visually flagged with a warning badge.
6. **Given** no items were extracted for a repeatable group, **When** the detail page loads, **Then** the section shows "No items were extracted for this group."
7. **Given** I expand a repeatable group item, **When** I edit a field value inline, **Then** the change is saved — same affordance as header-level field editing.

---

### User Story 4 — View Grouped Extraction in the History Detail (Document Entry Tab) (Priority: P3)

A user reviewing historical processing of a document needs to see the same grouped structure in the history view, but in a fully read-only mode.

**Why this priority**: Completes the review surface. Lower priority than the live record view because it is reference-only and does not block operational use.

**Independent Test**: Navigate to a document entry in the history tab; confirm groups are shown read-only, all repeatable group items are collapsed by default, and no edit controls are present.

**Acceptance Scenarios**:

1. **Given** I open the history detail view, **When** it loads, **Then** header fields are shown as a flat read-only grid — no change to existing behaviour.
2. **Given** the history view contains single object groups, **When** it loads, **Then** they appear as labeled cards with field-value pairs — no edit affordances.
3. **Given** the history view contains repeatable groups, **When** it loads, **Then** all items are collapsed by default; item count and warning badges are visible on the collapsed header.
4. **Given** any part of the history detail view, **When** I interact with it, **Then** no edit icons, save buttons, or inline edit controls are present anywhere.

---

### Edge Cases

- What happens when a repeatable group has 0 minimum items and the document has none? → Section shows "No items were extracted for this group."
- What happens when the same concept (e.g. ETA) exists at both header level and inside a group? → Both fields are defined independently; descriptions guide the model on which to populate. The wizard shows authoring guidance text.
- What happens when a document spans multiple pages and the same group appears on each page (bilingual)? → Fixture-covered by the Kombiverkehr test case; extraction must deduplicate or unify correctly.
- What happens when extracted items exceed the configured maximum? → Extra items are dropped.
- What happens when extraction encounters a structural error for a specific group (malformed model response, timeout, unmappable data)? → The group is treated as zero items / not found; the standard empty state is shown to the reviewer; the error is logged internally for observability.
- What happens when a Document Type reaches the 50-field tree-wide limit? → The UI disables adding more fields/groups and shows an informative message.
- What happens when a user tries to delete a group from an active Document Type? → Action is blocked with a clear error message citing the immutability rule.

---

## Requirements *(mandatory)*

### Functional Requirements

**Authoring**

- **FR-001**: Users MUST be able to add one or more named Groups to the Fields step of a Document Type, in addition to header-level fields.
- **FR-002**: Each Group MUST have: name, code (machine identifier), description, and kind (Repeatable or Single object).
- **FR-003**: Repeatable groups MUST support a configurable minimum and maximum item count (min default 0, max up to 200).
- **FR-004**: Single object groups MUST support an "optional" flag; if optional and absent, the group is omitted from output rather than returned as empty.
- **FR-005**: Users MUST be able to create, rename, reorder, and delete groups (subject to active Document Type immutability rules).
- **FR-006**: Users MUST be able to move existing fields into a group or back to header level.
- **FR-007**: A field MUST belong to at most one group; ungrouped fields are header-level.
- **FR-008**: Fields within a group are individually optional — the model fills what the document provides per item.
- **FR-009**: The field description input MUST always be visible during authoring (never collapsed or hidden).
- **FR-010**: The wizard MUST display authoring guidance text covering: disambiguation of same-concept fields at different levels, structural variability (table vs. inline text), and when to use repeatable vs. separate Document Types.
- **FR-011**: Total fields tree-wide (top-level + all group children) MUST NOT exceed 50 per Document Type.
- **FR-012**: Maximum items per repeatable group MUST NOT exceed 200.
- **FR-014**: Once a Document Type is active, group codes MUST NOT be changeable, groups MUST NOT be deletable, and fields inside groups MUST NOT be removable.
- **FR-013**: Group field codenames MUST be unique within a given Document Type; duplicates MUST be rejected at save time.

**Extraction**

- **FR-015**: The extraction model MUST receive header fields and each group as separate sections, each with name, description, and field list.
- **FR-016**: The extraction result for a repeatable group MUST be an array — one item per occurrence found in the document.
- **FR-017**: The extraction result for a single object group MUST be exactly one nested object, or absent if the group is optional and not found.
- **FR-018**: Unknown or extra items in extraction results MUST be dropped.
- **FR-019**: Missing items in repeatable groups MUST be padded to the configured minimum with empty values.

**Review — Record Detail Page**

- **FR-020**: The record detail page MUST render header fields first as a flat field-value grid with confidence chips — unchanged from today.
- **FR-021**: Single object groups MUST be rendered as labeled cards below header fields, with a group-level confidence chip (average of field confidence scores) and editable field-value pairs.
- **FR-022**: Optional single object groups not found in the document MUST still render as a card with a "Not found in this document" label.
- **FR-023**: Repeatable groups MUST be rendered as labeled sections with an accordion/tab strip; items collapsed by default except the first and any with warnings.
- **FR-024**: Repeatable group items with missing required fields or confidence below threshold MUST be visually flagged with a warning badge.
- **FR-025**: Field values inside group items MUST be editable inline — same affordance as header-level fields.
- **FR-026**: A repeatable group with no extracted items MUST display "No items were extracted for this group."

**Review — History Detail View**

- **FR-027**: The history detail view MUST follow the same grouped layout as the record detail page, but be fully read-only (no edit affordances anywhere).
- **FR-028**: In the history detail view, all repeatable group items MUST be collapsed by default; item count and warning badges remain visible on collapsed headers.

**API Contract**

- **FR-029**: The record detail endpoint's flat `field_values` list MUST be replaced by a unified `fields` list (polymorphic by `field_type`) — hard cutover, no versioned endpoint.
- **FR-030**: All in-product consumers (UI, internal tooling, API tests) MUST be fully migrated to the new shape before deploy; the deploy is a single coordinated release.
- **FR-031**: Release notes MUST call out this as a breaking change for external consumers of the record detail endpoint.

### Key Entities

- **Group**: Belongs to a Document Type. Attributes: name, code (immutable once active; unique within the Document Type), description, kind (Repeatable | Single object), optional (Single object only), min_items / max_items (Repeatable only). A group contains an ordered list of Fields.
- **Field**: Belongs to either header level or exactly one Group within a Document Type. Attributes: name, code, description, type (string / number / date / enum / currency / boolean), optional flag.
- **Document Type**: Top-level entity. Contains top-level Fields and Group fields (each group field's children are its member fields). Limit: 50 fields tree-wide.
- **Extraction Result**: Structured output with a single nested `fields` key — top-level value fields and group entries (repeatable_group / single_object_group) in one unified list.
- **Record**: A processed document instance. Links to a Document Type and stores the Extraction Result.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can define a Document Type with at least one repeatable group and one single object group in under 5 minutes, with no support intervention.
- **SC-002**: Extraction results for all five fixture cases (multiple items, single item, multi-page, flat-only, single object) pass with 100% structural correctness before rollout.
- **SC-003**: The record detail page renders grouped extraction results — including an accordion for repeatable groups — within the same load time as the current flat view (no perceptible regression).
- **SC-004**: Zero regressions on existing flat-only Document Types after the feature is deployed — all existing records continue to display and edit correctly.
- **SC-005**: The history detail view displays grouped results fully read-only; no edit affordances are reachable by any user interaction.
- **SC-006**: All in-product consumers of the record detail endpoint are migrated before release; zero broken integrations at deploy time.

---

## Clarifications

### Session 2026-05-12

- Q: Is the record detail endpoint change a hard cutover or a versioned approach? → A: Hard cutover — existing endpoint updated; all consumers migrate before deploy.
- Q: Must group codes be unique within a Document Type or across the workspace? → A: Unique within a Document Type only.
- Q: When extraction fails entirely for a group (structural error), what does the UI show? → A: Treat as zero items / not found — show standard empty state; log error internally.

---

## Assumptions

- Existing field types (string, number, date, enum, currency, boolean) are unchanged — groups are a structural concept independent of field type.
- Output payload generation (REST / MCP / file delivery) is out of scope for this story; it belongs to the Data Output story that this story unblocks.
- Group-level confidence thresholds, conditional groups, cross-group references, bulk-edit across items, and canonical group libraries are v2 and explicitly out of scope.
- The immutability rules for active Document Types already exist for fields; the same rules are extended to groups without changing the underlying mechanism.
- The UI wizard already has a multi-step flow for defining Document Types; the Fields step is extended, not replaced.
- Extraction quality gate must pass against the five specific real-world fixture documents before rollout is approved.
- The record detail page already supports inline editing and confidence chips for header-level fields; this capability is reused for group items without re-implementing it.
