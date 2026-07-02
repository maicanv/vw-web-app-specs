# Feature Specification: Improved Nesting (Arbitrary-Depth Field Groups)

**Feature Branch**: `005-improved-nesting`

**Created**: 2026-06-30

**Status**: Draft

**Input**: Confluence story "[DE] US: Improved Nesting" (https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/335347713)

## Summary

A Document Type author needs to define nested field groups of arbitrary depth — groups inside groups, and repeatable sections inside repeatable sections — so extraction and delivery can produce deeply nested integration payloads. The motivating target is a transport-order message (customer "DLG", example file `TYSON.json`) shaped `Order → CargoLines[] → Goods[] → Packing`, reaching ~3 real group levels (the depth cap is 5). Today Document Types allow exactly one level of grouping and forbid a group inside a group; every layer that reads the field tree (validation, extraction, review, delivery) assumes depth 2. This feature lifts that limit end-to-end and replaces the fixed delivery envelope with a per-route, author-written body template so the payload can match a customer's exact contract.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Author a deeply nested field tree (Priority: P1)

A Document Type author opens the Fields step and builds a tree where a repeatable group (`CargoLines`) contains another repeatable group (`Goods`), which in turn contains a single-object group (`Packing`) and value fields. They save, reopen, and edit it.

**Why this priority**: Nothing downstream (extraction, review, delivery) can capture parent→child nesting unless the schema itself can express it. This is the foundation slice.

**Independent Test**: Create a Document Type matching the full DLG tree (single-object + repeatable-inside-repeatable), save it, reload the editor, re-parent and delete a node, and confirm it persists and round-trips. Verifiable without any extraction running.

**Acceptance Scenarios**:

1. **Given** the Fields step, **When** an author nests a group inside another group (single-object or repeatable), **Then** it is accepted up to a depth of at least 5.
2. **Given** a tree already at maximum depth, **When** the author tries to nest one level deeper, **Then** the action is blocked with a clear message.
3. **Given** a saved nested Document Type, **When** the author reopens it and re-parents or deletes a node, **Then** the change validates, persists, and round-trips on the next edit.
4. **Given** a tree being saved, **When** the value (leaf) field count exceeds ~150, **Then** the save is rejected; structural group nodes are counted separately and are uncapped except by maximum depth.

---

### User Story 2 - Extract into the correct nested structure (Priority: P1)

A document with multiple cargo lines, each owning multiple goods with packing, is processed. Each good is stored against its own parent cargo line with no cross-contamination.

**Why this priority**: Correct per-item association is the core value and the hardest risk. Flat values cannot be reshaped back into the right parent at delivery time, so the nesting must be captured at extraction.

**Independent Test**: Feed a document with 2 cargo lines, each containing 2 goods with packing; assert each good is associated with the correct cargo line and absent optional groups are not invented.

**Acceptance Scenarios**:

1. **Given** a document with 2 cargo lines × 2 goods (each with packing), **When** extraction runs, **Then** every good is associated with its correct parent cargo line (zero cross-item leakage).
2. **Given** an optional nested group that is absent in the source, **When** extraction runs, **Then** no value is hallucinated for it.
3. **Given** a large manifest whose output is truncated/incomplete, **When** extraction finishes, **Then** the system retries once and, if still incomplete, flags the record for review rather than persisting partial data silently.

---

### User Story 3 - Review the nested result (Priority: P2)

A reviewer opens the record detail page and reads the full nested structure — groups within groups, arrays within arrays — with per-field confidence intact.

**Why this priority**: Human review is required before delivery (draft-first product principle); the reviewer must see the structure as extracted.

**Independent Test**: Open the record detail for a nested extraction and confirm each level renders read-correctly with confidence preserved on every leaf.

**Acceptance Scenarios**:

1. **Given** a nested extraction record, **When** the reviewer opens record detail, **Then** the full nested structure renders correctly at every level.
2. **Given** a rendered nested record, **When** the reviewer inspects a leaf field, **Then** its confidence score is shown.

---

### User Story 4 - Deliver via an author-written body template (Priority: P2)

In the Output Route creation dialog, when a body-carrying method (POST/PUT/PATCH) is selected, the author writes the customer's exact JSON shape using placeholders. On delivery the route renders that template into the final payload with nesting preserved.

**Why this priority**: The customer requires the payload in their exact shape; the legacy fixed `{general, email, data}` envelope cannot produce it. Delivery is the end of the value chain.

**Independent Test**: Configure an Output Route with a POST body template referencing metadata and extracted-data placeholders; deliver a nested record and assert the rendered payload deep-equals the target shape.

**Acceptance Scenarios**:

1. **Given** the Output Route dialog with a body-carrying method selected, **When** the author opens it, **Then** a JSON body template field is shown.
2. **Given** a template using metadata placeholders under a single `metadata` root (e.g. `{{ metadata.id }}`, `{{ metadata.email.received_at }}`), the whole extracted-data object (`{{ data }}`), and group-level references rendered with the `tojson` filter (`{{ data.CargoLines | tojson }}`), **When** a record is delivered, **Then** each placeholder resolves to its value/object and nesting is preserved end-to-end (each cargo line carries only its own goods; each good its own packing).
3. **Given** a body-carrying method is not selected, **When** the author configures the route, **Then** no body template field is required.

---

### User Story 5 - Field-usage feedback and save-time validation (Priority: P3)

The Output Route dialog shows a helper panel listing the offered metadata fields and all configured groups & fields, highlighting each when referenced in the body. Save is gated on template validity.

**Why this priority**: Quality-of-life and safety; it prevents silent mistakes but the route can function without it.

**Independent Test**: Reference some but not all configured fields in a template, attempt to save, and confirm an unreferenced-field warning; introduce a typo'd placeholder and confirm a blocking error.

**Acceptance Scenarios**:

1. **Given** the dialog, **When** the author views the helper panel, **Then** offered metadata fields (with their keys) and all configured groups & fields are listed, highlighted when referenced and un-highlighted when not.
2. **Given** a template with an invalid/unresolvable placeholder (typo, unknown key) or malformed JSON, **When** the author saves, **Then** the save is blocked with a hard error.
3. **Given** a template that leaves some configured fields unreferenced, **When** the author saves, **Then** a non-blocking warning is raised and the save proceeds.

---

### Edge Cases

- Depth exactly at the maximum vs. one level beyond — boundary must accept the former, block the latter.
- A wrong array boundary (e.g. miscounted cargo lines) corrupts every nested child; the structural-correctness gate must catch this, not just per-field accuracy.
- Confidence is per leaf field — a structurally wrong grouping can still score high, so confidence alone is not the gate.
- Large manifest output exceeding the token limit yields partial/invalid JSON → retry-once + review flag.
- The largest documents (the ones motivating this feature) are most likely to spuriously time out; the extraction timeout must accommodate them.
- An existing flat or single-level Document Type must behave exactly as before (no regression).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Authors MUST be able to nest a group (single-object or repeatable) inside another group, including repeatable-inside-repeatable, to a depth of at least 5.
- **FR-002**: The system MUST block nesting beyond the maximum depth with a clear, user-facing message.
- **FR-003**: The system MUST count value (leaf) fields separately from structural group nodes, reject a Document Type whose value-field count exceeds ~150, and leave group nodes uncapped except by maximum depth.
- **FR-004**: Repeatable groups MUST continue to honor min_items/max_items at every nesting level.
- **FR-005**: Creating or updating a Document Type with a nested tree MUST validate, persist, round-trip on edit, and reconcile re-parenting and deletion correctly.
- **FR-006**: Extraction MUST associate each child item with its correct parent item across all nesting levels, with zero cross-item leakage.
- **FR-007**: Extraction MUST NOT invent values for absent optional groups or fields.
- **FR-008**: On truncated/incomplete extraction output, the system MUST retry once and, if still incomplete, flag the record for review instead of persisting partial data silently.
- **FR-009**: The record detail page MUST render the full nested structure read-correctly at every level, preserving per-leaf confidence.
- **FR-010**: The Output Route creation dialog MUST show a JSON body template field when a body-carrying method (POST/PUT/PATCH) is selected, and not require one otherwise.
- **FR-011**: The body template (sandboxed Jinja2, `StrictUndefined`) MUST support placeholders for provided metadata variables under a single `metadata` root (e.g. `{{ metadata.id }}`, `{{ metadata.email.received_at }}`), the whole extracted-data object (`{{ data }}`), and group-level references into it rendered with the stock `tojson` filter (e.g. `{{ data.CargoLines | tojson }}`); scalar placeholders are author-quoted where JSON needs a string, containers use `| tojson` (never quoted); wrappers, metadata and constants live in the template, not a fixed data section.
- **FR-012**: On delivery the route MUST render the template into the final payload with nesting preserved end-to-end.
- **FR-013**: The dialog MUST show a helper panel listing offered metadata fields (with keys) and all configured groups & fields, highlighting each when referenced in the body.
- **FR-014**: On save, the system MUST treat an invalid/unresolvable placeholder or malformed JSON as a hard error that blocks saving, and unreferenced configured fields as a non-blocking warning.
- **FR-015**: Existing flat and single-level Document Types MUST produce byte-identical extraction and delivery output to before this feature.
- **FR-016**: The extraction timeout for this Document Type flow MUST accommodate the largest in-scope documents (≈10 cargo lines × 4 goods, observed ~36.5s) without spuriously timing out.

### Key Entities *(include if feature involves data)*

- **Document Type**: The author-defined extraction contract; owns a tree of fields.
- **Field / Group node**: A node in the Document Type tree. A value (leaf) field holds a typed scalar; a group node is either single-object (nested object) or repeatable (array of items) and may contain further group or value nodes. Self-referential parent→child relationship of arbitrary depth.
- **Extraction record**: The result of processing one document; holds extracted field values plus per-leaf confidence, structured to mirror the field tree including per-item nesting.
- **Output Route**: A delivery target; for body-carrying methods it owns an author-written JSON body template that references metadata and extracted-data placeholders.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An author can build and save a Document Type matching the full DLG shape (depth ≥ 3, repeatable-inside-repeatable) without hitting a depth or field-count limit.
- **SC-002**: On the fixture set, nested extraction achieves correct structural grouping (correct cargo-line count and correct goods-per-line) with zero cross-item leakage.
- **SC-003**: Zero invented values for absent optional groups across the fixture set.
- **SC-004**: A delivered payload for a nested record deep-equals the customer's target shape (each cargo line owns only its own goods; each good its own packing).
- **SC-005**: Existing flat and single-level Document Types produce byte-identical extraction and delivery output (regression-free).
- **SC-006**: The largest in-scope documents (≈10 cargo lines × 4 goods) deliver without spuriously timing out.

## Assumptions

- Maximum group depth is set to 5: the DLG data needs only 3, and depth-5 arrays-in-arrays were extracted at 100% structural correctness with zero cross-item leakage in the stress test, so 5 is safe headroom.
- Extraction is one-pass: a staged (outer-array-then-children) approach gave no structural benefit at +30–40% tokens and 6–11 calls.
- No pre-chunking of manifests unless they reach hundreds of items; at DLG scale (40 goods ≈ 8k output tokens) there is no truncation.
- The Document Type definition model is already recursive (self-referential parent→child); the schema can express arbitrary depth without a new structure — the work is removing the depth-2 assumption from each consumer and addressing per-item indices for nested arrays.
- The draft-first product principle holds: delivery still produces a draft/route output for human review, never an autonomous send.
- Uploading an example JSON to auto-generate the body template is **out of scope** (story item 9 is marked V2).
- Internal routing-template mechanics and the migration of per-item indexing are deferred to technical refinement, not specified here.

## Out of Scope

- Auto-generation of the Output Route body template from an uploaded example JSON (V2).
- Org/workspace setting cascade for Document Types (tracked separately).
