---

description: "Task list for Document Type Field Grouping (VWE-1496)"
---

# Tasks: Document Type Field Grouping (VWE-1496)

**Input**: `specs/001-field-grouping/` + `technical_proposals/tp_vwe1496_field_grouping.md`  
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Data model**: [data-model.md](data-model.md) | **Contract**: [contracts/record-detail-api.md](contracts/record-detail-api.md)

## Table of Contents

- [Format](#format-id-p-story-description)
- [Phase 1: Setup](#phase-1-setup)
- [Phase 2: Foundational (Blocking Prerequisites)](#phase-2-foundational-blocking-prerequisites)
- [Phase 3: US1 — Define Field Groups on a Document Type (P1)](#phase-3-us1--define-field-groups-on-a-document-type-p1)
- [Phase 4: US2 — Extract Grouped Fields from a Document (P1)](#phase-4-us2--extract-grouped-fields-from-a-document-p1)
- [Phase 5: US3 — Review Grouped Extraction on Record Detail Page (P2)](#phase-5-us3--review-grouped-extraction-on-record-detail-page-p2)
- [Phase 6: US4 — View Grouped Extraction in History Detail (P3)](#phase-6-us4--view-grouped-extraction-in-history-detail-p3)
- [Phase 7: Polish & Cross-Cutting Concerns](#phase-7-polish--cross-cutting-concerns)
- [Dependencies & Execution Order](#dependencies--execution-order)
- [Implementation Strategy](#implementation-strategy)
- [Notes](#notes)

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (US1–US4)
- Exact file paths are included in each task description

---

## Phase 1: Setup

**Purpose**: Verify brownfield context and confirm baseline before any changes

- [ ] T001 Confirm `backend/django/apps/document_entries/models.py` baseline — read DocumentTypeField, ExtractedFieldValue, and existing FieldConfig discriminated union to understand current structure
- [ ] T002 [P] Confirm `client/src/types/documentEntry.ts` baseline — read existing ExtractedFieldValue and ExtractionRecordDetail types to understand current shape

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: DB migration + model extension — MUST be complete before US1, US2, US3, US4 can be implemented

**⚠️ CRITICAL**: All user story phases depend on T003–T008 being complete

- [ ] T003 Add `parent_field` nullable self-FK (CASCADE, `related_name="children"`) to `DocumentTypeField` in `backend/django/apps/document_entries/models.py`
- [ ] T004 Add `group_item_index` nullable IntegerField to `ExtractedFieldValue` in `backend/django/apps/document_entries/models.py`
- [ ] T005 Add `RepeatableGroupConfig` (description, min_items, max_items) and `SingleObjectGroupConfig` (description, optional) variants to the `FieldConfig` discriminated union in `backend/django/apps/document_entries/models.py`
- [ ] T006 Create migration `backend/django/apps/document_entries/migrations/000N_add_field_grouping.py` — single migration adding `parent_field` self-FK on `DocumentTypeField` and `group_item_index` on `ExtractedFieldValue` (both nullable, backward-safe)
- [ ] T007 [P] Update `FieldConfig` TypeScript discriminated union in `client/src/types/documentEntry.ts` — add `RepeatableGroupConfig` and `SingleObjectGroupConfig` variants; add optional `fields: DocumentTypeField[]` to `DocumentTypeField`; add `RepeatableGroupEntry`, `SingleObjectGroupEntry`, `FieldsListEntry` union type; update `ExtractionRecordDetail` to replace `field_values` with `fields: FieldsListEntry[]` (see `contracts/record-detail-api.md` for exact shapes)
- [ ] T008 [P] Add group field factory traits in `backend/django/apps/document_entries/factories.py` — `DocumentTypeFieldFactory` traits for `repeatable_group` and `single_object_group` config; child field trait with `parent_field` set; `ExtractedFieldValue` trait with `group_item_index`

**Checkpoint**: Migration applied, model extended, FieldConfig union has group variants, TypeScript types updated — user story work can now begin

---

## Phase 3: US1 — Define Field Groups on a Document Type (P1)

**Goal**: Users can define repeatable and single-object groups in the Document Type wizard; the backend persists the nested field tree via the existing create/update endpoints

**Independent Test**: POST a Document Type with a repeatable group "Pickup Codes" containing "Code" and "ETA" child fields; confirm it persists with correct `parent_field` relationships; PATCH the same type to rename the group and add a child field; confirm atomic update

**Depends on**: Phase 2 complete (T003–T008)

### Backend — Serializer & View

- [ ] T009 [US1] Update `DocumentTypeFieldSerializer` in `backend/django/apps/document_entries/serializers.py` — accept nested `fields` array for group entries; validate: (a) field with non-null parent must use a value-type config (no groups-in-groups), (b) codename uniqueness across the whole tree per DocumentType, (c) 50-field tree-wide limit, (d) `max_items >= min_items` and `max_items <= 200` for repeatable groups; depth-2 rule enforced here
- [ ] T010 [US1] Update `DocumentTypeUpdateSerializer` in `backend/django/apps/document_entries/serializers.py` — reconcile the nested field tree atomically: match existing fields by `id`, create new ones without `id`, delete absent ones (blocked with 400 if DocumentType is active), re-parent fields by tree position; group field codename and `parent_field` are read-only when DocumentType is active (extend existing active-type immutability block)
- [ ] T011 [US1] Update `document_type_view_set.py` in `backend/django/apps/document_entries/document_type_view_set.py` — ensure create and update operations wrap the nested tree reconciliation in an atomic transaction; no new route needed (groups are fields)

### Frontend — FieldsStep

- [ ] T012 [US1] Update `FieldsStep.tsx` in `client/src/app/documentEntry/documentTypes/steps/FieldsStep.tsx` — add "Add Repeatable Group" and "Add Single Object Group" options to the field creation menu; group fields render as collapsible containers with their children as nested `FieldRow` entries; value fields inside a group use existing `FieldRow` component
- [ ] T013 [US1] Update `FieldsStep.tsx` group creation form — collect: name, codename, description, and (for repeatable) min_items / max_items; (for single object) optional flag; description input MUST render unconditionally (not behind a disclosure/toggle) — satisfies FR-009; confirm same is true for existing flat field forms
- [ ] T014 [US1] Update `FieldsStep.tsx` drag-drop behaviour — moving a value field into or out of a group container updates `parent_field`; moving a group into another group is blocked client-side (depth ≤ 2) with a tooltip error; uses existing `@hello-pangea/dnd` setup
- [ ] T015 [US1] Update `FieldsStep.tsx` limits and immutability — "Add Field" and "Add child field" buttons disabled when 50-field tree-wide total is reached; group edit/delete controls show inline error when DocumentType is active; reuses existing active-type guard pattern
- [ ] T016 [US1] Add authoring guidance text (FR-010) to `FieldsStep.tsx` — collapsible Mantine `Alert` above the field list, shown by default; content: disambiguation of same-concept fields at different levels, structural variability (table vs. inline), when to use repeatable vs. separate Document Types

**Checkpoint**: US1 fully functional — Document Types with nested group fields can be created and updated; immutability and limits enforced

---

## Phase 4: US2 — Extract Grouped Fields from a Document (P1)

**Goal**: The extraction model receives a unified nested field list, returns grouped results, and the side effect handler writes `ExtractedFieldValue` rows with correct `group_item_index`

**Independent Test**: Submit Mercitalia pickup-code manifest against a DocumentType with a `pickup_codes` repeatable group; confirm 5 `ExtractedFieldValue` rows exist for each child field codename with `group_item_index` 0–4

**Depends on**: Phase 2 complete (T003–T008); US1 backend (T009–T011) needed to create test DocumentTypes with groups

### vw-llm-app

- [ ] T017 [US2] Replace flat `_ExtractionResult` Pydantic model with recursive model in `vw-llm-app/actions/document_entry_side_effect_handler.py` — add `_FieldValue`, `_GroupItem`, `_RepeatableGroupResult`, `_SingleObjectGroupResult`, and updated `_ExtractionResult` with a single `fields: dict[str, Any]` key (see §4.2 of `technical_proposals/tp_vwe1496_field_grouping.md` for exact model definitions); confirm Pydantic major version and use correct recursion syntax (v1: `update_forward_refs()`, v2: `model_rebuild()`)
- [ ] T018 [US2] Update `_extract()` call in `vw-llm-app/actions/document_entry_side_effect_handler.py` — pass unified nested `doc_type["fields"]` list (group entries carry their children inline with `type`, `description`, and `fields` keys) instead of separate header fields and groups sections; match shape in §4.1 of TP
- [ ] T019 [US2] Update extraction prompt template in `vw-llm-app/prompts/templates.py` — render nested `fields` inline (not two separate sections); instruct model to return repeatable groups as `items` arrays and single object groups as nested `fields` dicts (or omit if optional and absent)
- [ ] T020 [US2] Update side effect handler write-back loop in `vw-llm-app/actions/document_entry_side_effect_handler.py` — walk the DocumentType field tree recursively: (a) top-level value fields: one `ExtractedFieldValue` per codename, `group_item_index=NULL`; (b) repeatable group: for each item at index `i`, one row per child field with `group_item_index=i`; pad to `min_items` with empty rows; cap at `max_items`; (c) single object group: one row per child field with `group_item_index=NULL`; if optional and absent, write no rows; (d) child field absent from LLM response for an item: write one `ExtractedFieldValue` row with `raw_value=None, display_value=None` (best-effort — do not skip the row, as the record detail serializer iterates the schema tree to build the response); (e) unknown codenames at any level: drop and append to `needs_review_reason`; (f) structural errors on a subtree: treat as absent, log to Sentry, append to `needs_review_reason`, continue for remaining fields

### Extraction Quality Gate

- [ ] T021 [US2] Run extraction quality gate — all five fixture documents must pass with 100% structural correctness before rollout approval:
  - Mercitalia pickup-code manifest: 5 items in `pickup_codes` repeatable group
  - Hupac booking confirmation: 1 item in `bookings` repeatable group (returned as array)
  - Kombiverkehr delay notice (bilingual): items from both pages returned, duplicates accepted
  - Flat-only document: no group entries, value entries populated — regression baseline
  - Sender block document: 1 nested object in `sender` single object group

**Checkpoint**: US2 fully functional — extraction produces correct grouped `ExtractedFieldValue` rows; all five fixture cases pass

---

## Phase 5: US3 — Review Grouped Extraction on Record Detail Page (P2)

**Goal**: The record detail page renders grouped extraction results — header fields, single object group cards, repeatable group accordions — with inline editing parity to today's flat view

**Independent Test**: Open a record extracted against a DocumentType with a repeatable "Invoice Lines" group; confirm accordion with one tab per extracted line item; edit a field in item 2; confirm save

**Depends on**: Phase 2 (T003–T008), US2 backend (T017–T020) to have real grouped extraction data

### Backend — Record Detail Serializer

- [ ] T022 [US3] Update `ExtractionRecordDetailSerializer` in `backend/django/apps/document_entries/serializers.py` — remove `field_values` field; add unified `fields` serializer method: (a) single query `ExtractedFieldValue.objects.filter(extraction_record=...).select_related("document_type_field__parent_field")`; (b) bucket rows by `document_type_field.parent_field_id`; (c) iterate DocumentType field tree in `display_order` order; (d) top-level value fields → existing shape; (e) repeatable group field → group metadata + `items` (list of lists, ordered by `group_item_index` then child `display_order`); (f) single object group field → group metadata + `fields` list + `not_found` bool; (g) groups with no extracted rows still appear (repeatable: `items: []`; absent optional single object: `fields: null, not_found: true`)

### Frontend — New Components

- [ ] T023 [P] [US3] Create `GroupCard.tsx` in `client/src/app/documentEntry/components/GroupCard.tsx` — renders a single object group entry; header: group name + group-level `ConfidenceBar` (arithmetic mean of non-null child confidences, computed client-side); body: child field-value pairs via existing `FieldRow` component; when `not_found: true`: show "Not found in this document" label instead of field list; edit affordances hidden in read-only mode or when `not_found`
- [ ] T024 [P] [US3] Create `RepeatableGroupSection.tsx` in `client/src/app/documentEntry/components/RepeatableGroupSection.tsx` — renders a repeatable group entry as Mantine `Accordion`; section header: group name + item count + warning badge if any item has a child below confidence threshold or missing critical child; each accordion item labelled "Item 1", "Item 2", …; each item has its own `ConfidenceBar`; items with warnings are flagged visually; first item + warned items expanded by default; read-only mode: all collapsed by default; each item body renders `FieldRow` entries; empty state: "No items were extracted for this group"

### Frontend — Record Detail Page

- [ ] T025 [US3] Update `ExtractionRecordDetailPage.tsx` in `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx` — replace flat `field_values` table with ordered render of unified `fields` list: `FieldsListEntry` items dispatched by `field_type` — value entries use existing `FieldRow`, repeatable entries use `RepeatableGroupSection`, single-object entries use `GroupCard`; order preserved from API response (DocumentType `display_order`)

**Checkpoint**: US3 fully functional — grouped extraction is reviewable and editable on the record detail page; no regressions on flat-only records

---

## Phase 6: US4 — View Grouped Extraction in History Detail (P3)

**Goal**: The history detail view (DocumentEntryRenderer) renders grouped results fully read-only — same layout as the record detail page but no edit affordances

**Independent Test**: Navigate to a document entry in the history tab with a repeatable group; confirm all items are collapsed by default; confirm no edit icons or save buttons exist anywhere

**Depends on**: Phase 5 complete (T022–T025) — reuses `GroupCard` and `RepeatableGroupSection` in read-only mode

- [ ] T026 [US4] Update `DocumentEntryRenderer.tsx` in `client/src/app/application/history/actionRenderers/DocumentEntryRenderer.tsx` — replace flat `ExtractionEntry` render with unified `fields` list render; pass `readOnly={true}` to `GroupCard` and `RepeatableGroupSection`; all accordion items collapsed by default; no edit icons, save buttons, or inline edit controls anywhere; reuses components created in T023 and T024

**Checkpoint**: US4 fully functional — history view displays grouped results read-only; item count and warning badges visible on collapsed headers

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Tests, quality gate verification, migration checklist, and release prep

- [ ] T027 [P] Add Django model + serializer tests in `backend/django/tests/test_apps/test_document_entries/test_document_type_viewset.py` — cover: codename uniqueness tree-wide; 50-field tree-wide limit; depth-2 enforcement (group inside group rejected); CASCADE deletes children when parent deleted (draft only); active-type immutability (codename read-only, `parent_field` read-only, delete blocked, re-parent blocked); repeatable group config validation (`max_items >= min_items`, `max_items <= 200`); nested reconciliation atomicity (any validation failure rejects entire PATCH)
- [ ] T028 [P] Add Django record detail tests in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py` — cover: unified `fields` list shape; repeatable group items ordered by `group_item_index`; `not_found: true` for absent optional single object; `items: []` for repeatable with no extracted rows; flat-only records produce no group entries; `field_values` key absent from all responses; include a `django_assert_num_queries` assertion to verify the record detail endpoint issues at most 1 `ExtractedFieldValue` query regardless of group count (SC-003 load parity)
- [ ] T029 [P] Add Vitest component tests for `GroupCard.tsx` — renders child rows; `not_found` empty state; edit affordances absent in read-only mode
- [ ] T030 [P] Add Vitest component tests for `RepeatableGroupSection.tsx` — item count in section header; first item expanded by default; warning badge on items with low confidence; all collapsed in read-only mode; empty state when no items
- [ ] T031 [P] Add Vitest tests for `ExtractionRecordDetailPage.tsx` — unified `fields` list renders in order; `GroupCard` for single-object entries; `RepeatableGroupSection` for repeatable entries; no runtime error when `field_values` absent
- [ ] T032 Verify migration checklist in `specs/001-field-grouping/contracts/record-detail-api.md` — check off all items before deploy: `ExtractionRecordDetailSerializer` updated, TypeScript type updated, `ExtractionRecordDetailPage.tsx` updated, `DocumentEntryRenderer.tsx` updated, all Django API tests updated, release notes drafted

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Setup — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Foundational — backend (T009–T011) and frontend (T012–T016) can run in parallel once Phase 2 is done
- **US2 (Phase 4)**: Depends on Foundational + US1 backend (T009–T011 needed to create test DocumentTypes with groups)
- **US3 (Phase 5)**: Depends on Foundational + US2 backend (T017–T020) — needs real grouped extraction data to test serializer and UI
- **US4 (Phase 6)**: Depends on Phase 5 — reuses `GroupCard` and `RepeatableGroupSection`
- **Polish (Phase 7)**: Depends on all stories complete

### Within Each Phase — Task Order

- **Phase 2**: T003 → T004 → T005 → T006 (sequential, same file); T007 and T008 can run in parallel after T005
- **Phase 3**: T009 → T010 → T011 (serializer before view); T012–T016 all depend on T007 (TS types) and can run in order
- **Phase 4**: T017 → T018 → T019 → T020 (sequential, same file + template); T021 after T020
- **Phase 5**: T022 (backend) can run in parallel with T023 + T024 (new components); T025 after T022 + T023 + T024
- **Phase 6**: T026 after T023 + T024 (needs the components)
- **Phase 7**: T027–T031 all [P] after their respective phases; T032 after all

### Parallel Opportunities

- T007 (TS types) and T008 (factories) can run in parallel after T005
- T009–T011 (backend serializer/view) and T012–T016 (frontend FieldsStep) can run in parallel after Phase 2
- T023 (GroupCard) and T024 (RepeatableGroupSection) can run in parallel after T007
- T027–T031 (tests) can all run in parallel

---

## Implementation Strategy

### MVP (US1 + US2 only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (migration, model, TS types)
3. Complete Phase 3: US1 — Document Type authoring with groups
4. Complete Phase 4: US2 — Extraction pipeline + quality gate
5. **STOP and VALIDATE**: Five fixture documents pass; Document Types with groups create/update correctly
6. Deploy US1+US2 behind feature flag if needed

### Incremental Delivery

1. Phase 1+2 → Foundation ready
2. Phase 3 (US1) → Authors can define grouped Document Types
3. Phase 4 (US2) → Extraction produces correct grouped data; quality gate passes
4. Phase 5 (US3) → Reviewers can view and edit grouped extraction results
5. Phase 6 (US4) → History view shows grouped results read-only
6. Phase 7 → Tests + release prep → Ship

### Parallel Team Strategy

With two developers after Phase 2:

- Developer A: US1 backend (T009–T011) + US2 vw-llm-app (T017–T021)
- Developer B: US1 frontend (T012–T016) + US3 components (T023, T024) + US3 page (T025)

---

## Notes

- `[P]` = different files, no blocking dependencies on incomplete tasks — safe to run in parallel
- `[Story]` label maps each task to a specific user story for traceability
- TP reference: `technical_proposals/tp_vwe1496_field_grouping.md` for detailed JSON examples and Pydantic model definitions
- `contracts/record-detail-api.md` contains the migration checklist for T032 — verify before any deploy
- Quality gate (T021) is a hard rollout gate — all five fixture cases must pass before Phase 5 is unblocked for production deploy
- Pydantic version check is required before T017 — recursion syntax differs between v1 and v2
- CASCADE on `parent_field` fires only on draft DocumentTypes — active-type immutability (T010) is the guard
- The `display_order` field on `DocumentTypeField` controls ordering within the parent context (top-level among top-level; children among their siblings) — no changes needed to `display_order` itself
