---
description: "Task list for Improved Nesting (Arbitrary-Depth Field Groups) — VWE-1609"
---

# Tasks: Improved Nesting (Arbitrary-Depth Field Groups)

**Input**: Design documents from `/specs/005-improved-nesting/` + `technical_proposals/tp_vwe1609_improved_nesting.md` (source of truth)
**Prerequisites**: plan.md, spec.md, data-model.md, contracts/, research.md, quickstart.md, tp_vwe1609

**Tests**: Backend (Django) and vw-llm-app test tasks ARE included — the proposal §8 quality gate and spec Success Criteria (SC-002…SC-006) require them. No frontend (client) tests per project rule.

## Table of Contents

- [Format](#format-id-p-story-description)
- [Path Conventions](#path-conventions)
- [Phase 1: Setup](#phase-1-setup)
- [Phase 2: Foundational (Blocking Prerequisites)](#phase-2-foundational-blocking-prerequisites)
- [Phase 3: User Story 1 — Author a deeply nested field tree (P1)](#phase-3-user-story-1--author-a-deeply-nested-field-tree-p1--mvp)
- [Phase 4: User Story 2 — Extract into the correct nested structure (P1)](#phase-4-user-story-2--extract-into-the-correct-nested-structure-p1)
- [Phase 5: User Story 3 — Review the nested result (P2)](#phase-5-user-story-3--review-the-nested-result-p2)
- [Phase 6: User Story 4 — Deliver via an author-written body template (P2)](#phase-6-user-story-4--deliver-via-an-author-written-body-template-p2)
- [Phase 7: User Story 5 — Field-usage feedback and save-time validation (P3)](#phase-7-user-story-5--field-usage-feedback-and-save-time-validation-p3)
- [Phase 8: Polish & Cross-Cutting Concerns](#phase-8-polish--cross-cutting-concerns)
- [Dependencies & Execution Order](#dependencies--execution-order)
- [Parallel Examples](#parallel-examples)
- [Implementation Strategy](#implementation-strategy)
- [Notes](#notes)

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (US1–US5)
- Include exact file paths in descriptions

## Path Conventions

- **Django**: `backend/django/apps/document_entries/`
- **vw-llm-app**: `vw-llm-app/src/`
- **Client**: `client/src/app/documentEntry/`
- Backend tests run via `docker compose` from `backend/`; vw-llm-app tests via its own pytest.

---

## Phase 1: Setup

**Purpose**: Confirm shared prerequisites; no new top-level dirs.

- [ ] T001 Confirm `jinja2` (with `jinja2.sandbox.SandboxedEnvironment`) is available to the Django backend; add to `backend/pyproject.toml` (default/django group) only if not already resolvable, then `poetry lock` — do not add if it is already a transitive dependency.
- [ ] T002 [P] Verify Pydantic v2 forward-ref + `model_rebuild()` support is usable in `vw-llm-app` (already a dep) — no install expected; document the version in the task PR if a bump is needed.

**Checkpoint**: Template-render and recursive-model dependencies confirmed available.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The `ExtractedFieldValue` schema change is a shared prerequisite for extraction write-back (US2) and nested review render (US3). MUST complete before those stories.

**⚠️ CRITICAL**: US2 and US3 cannot begin until this phase is complete.

- [ ] T003 Add `group_item_path = JSONField(default=list)` to `ExtractedFieldValue` in `backend/django/apps/document_entries/models.py` (keep `group_item_index` transitionally; keep `group_field`). Per tp §3.2 / data-model.md.
- [ ] T004 Generate the migration via `docker compose exec django python manage.py makemigrations document_entries` (NEVER hand-write): (a) add `group_item_path`, (b) data-migrate existing rows — `[]` when no repeatable ancestor else `[group_item_index]`, (c) drop `group_item_index`. Lossless per tp §3.4 (no prod nested rows). Confirm Open-item #1 (no in-flight prod records) before dropping the column.
- [ ] T005 [P] Backend test: migration data-migration maps a legacy `group_item_index` row to `[old_index]` and a null-index row to `[]`, in `backend/django/tests/test_migrations/` (or the document_entries test module). Written before T004 is finalized.

**Checkpoint**: `group_item_path` exists and back-fills losslessly — extraction and review can now build on it.

---

## Phase 3: User Story 1 — Author a deeply nested field tree (P1) 🎯 MVP

**Goal**: Authors can nest groups inside groups (single-object + repeatable-in-repeatable) to depth 5, save, reopen, re-parent, and delete; leaf fields capped at 150, group nodes uncapped except by depth.

**Independent Test**: Create a Document Type matching the full DLG tree, save it, reload the editor, re-parent and delete a node, confirm it persists and round-trips. No extraction needed.

### Tests for User Story 1 ⚠️ (write first, ensure they FAIL)

- [ ] T006 [P] [US1] Serializer test: full TYSON/DLG tree (repeatable-in-repeatable + single-object) validates and round-trips on GET after create, in `backend/django/apps/document_entries/tests/` (per tp §8, contracts/document-type-fields.md).
- [ ] T007 [P] [US1] Serializer test: nesting depth > 5 → `400` with depth-limit message; depth == 5 accepted (boundary), same test module.
- [ ] T008 [P] [US1] Serializer test: leaf-field count > 150 → `400` (`MAX_FIELDS_ERR`); group nodes excluded from the cap; `min_items`/`max_items` validated at every level.
- [ ] T009 [P] [US1] Serializer test: `reconcile_fields` re-parents and deletes nested nodes on PATCH and round-trips (tree-wide codename uniqueness + active-type immutability unchanged).

### Implementation for User Story 1

- [ ] T010 [US1] In `backend/django/apps/document_entries/field_config.py`: add `MAX_GROUP_DEPTH = 5`; change the field cap to count **leaf** fields only, capped at 150 (replaces `MAX_FIELDS_PER_DOCUMENT_TYPE = 50` which counted the whole tree). Per tp §3.1.
- [ ] T011 [US1] In `backend/django/apps/document_entries/serializers.py`: remove the `NESTED_GROUP_ERR` rejection (~l.329); make `_validate_group_field` recurse — accept nested groups, enforce depth ≤ 5 (`400` clear message), leaf-count ≤ 150, group-node exclusion, `min_items`/`max_items` per level, and "value fields cannot have children".
- [ ] T012 [US1] In `backend/django/apps/document_entries/services.py`: make `reconcile_fields` (~l.190) recursive — create/update/delete/re-parent nodes at any depth so a saved nested tree round-trips.
- [ ] T013 [US1] In `client/src/app/documentEntry/types/documentEntry.ts`: make `DocumentTypeField` recursive (a group node's children may be group nodes).
- [ ] T014 [US1] In `client/src/app/documentEntry/FieldsStep.tsx`: replace the depth-2 client block with a depth-`MAX_GROUP_DEPTH` block (tooltip on the boundary); allow adding a group inside a group; reflect the 150 leaf-count limit.
- [ ] T015 [US1] In `client/src/app/documentEntry/RepeatableGroupSection.tsx` / `GroupCard.tsx`: recurse into child *groups* (re-enter the same components), not only leaf `FieldRow`s.

**Checkpoint**: An author can build, save, reopen, and edit the full DLG tree (SC-001).

---

## Phase 4: User Story 2 — Extract into the correct nested structure (P1)

**Goal**: A document with multiple cargo lines × goods × packing extracts with each child stored against its correct parent (zero cross-item leakage), absent optionals not invented, truncation → retry-once + review flag.

**Independent Test**: Feed a doc with 2 cargo lines × 2 goods (each with packing); assert each good associates with the correct cargo line and absent optional groups are not invented.

**Depends on**: Phase 2 (`group_item_path`).

### Tests for User Story 2 ⚠️ (write first, ensure they FAIL)

- [ ] T016 [P] [US2] vw-llm-app write-back test: `group_item_path` values correct across levels (`[0]`,`[1]`,`[0,0]`,`[0,1]`,`[1,0]`…); top-level + single-object-group children → `[]`. In `vw-llm-app` tests (per tp §8).
- [ ] T017 [P] [US2] vw-llm-app test: `min_items` padding / `max_items` capping / unknown-codename drop applied at **every** level, not just the first.
- [ ] T018 [P] [US2] vw-llm-app test: absent optional group → no hallucinated value (FR-007); truncated/invalid output → retry once then flag for review, no silent partial (FR-008).

### Implementation for User Story 2

- [ ] T019 [US2] In `vw-llm-app/src/prompts/templates.py`: render the nested `fields` tree inline to arbitrary depth (today stops at one group level) — repeatable group → array of items, single-object group → nested object (omit if optional + absent), recurse. Per tp §5.
- [ ] T020 [US2] In `vw-llm-app/src/actions/document_entry_side_effect_handler.py`: make `_ExtractionResult` / `_RepeatableGroupResult` / `_SingleObjectGroupResult` mutually recursive (Pydantic v2 forward refs + `model_rebuild()`); keep the `dict[str, Any]` per-subtree escape hatch so the `function_calling` tool schema doesn't explode at depth.
- [ ] T021 [US2] In the same handler: recursive write-back — walk the tree accumulating the index path down each repeatable ancestor, write `group_item_path` on each leaf; carry existing `min_items` padding / `max_items` capping / unknown-codename handling at every level.
- [ ] T022 [US2] Robustness (tp §5): one-pass extraction; on truncated/invalid output retry once then flag for review (no silent partial); raise the LLM timeout for this flow to **≥60s** (FR-016) in the vw-llm-app extraction config/call site.

**Checkpoint**: Nested extraction achieves correct structural grouping with zero cross-item leakage (SC-002, SC-003, SC-006).

---

## Phase 5: User Story 3 — Review the nested result (P2)

**Goal**: The record-detail page renders the full nested structure (groups-in-groups, arrays-in-arrays) with per-leaf confidence; a flat/single-level record stays byte-identical.

**Independent Test**: Open record detail for a nested extraction; confirm each level renders read-correctly with confidence on every leaf.

**Depends on**: Phase 2 (`group_item_path`). Reads trees authored via US1.

### Tests for User Story 3 ⚠️ (write first, ensure they FAIL)

- [ ] T023 [P] [US3] Backend serializer test: record-detail emits the nested `fields` tree recursively, grouping each repeatable group's items by the index at *that* group's depth in `group_item_path`; per-leaf confidence preserved; zero cross-item leakage (contracts/extraction-record-detail.md, tp §4.2).
- [ ] T024 [P] [US3] Backend regression test: a flat / single-level record's detail response is byte-identical to before (FR-015 / SC-005).

### Implementation for User Story 3

- [ ] T025 [US3] In `backend/django/apps/document_entries/serializers.py`: make the record-detail serializer emit the `fields` tree recursively to full depth (`DocumentTypeFieldSerializer` already nests via `source="children"`), grouping repeatable items by `group_item_path` depth-index, preserving per-leaf confidence.
- [ ] T026 [US3] In `client/src/app/documentEntry/types/documentEntry.ts`: make `ExtractedFieldEntry` recursive (a group entry's children may be group entries).
- [ ] T027 [US3] In `client/src/app/documentEntry/` record-detail components: recurse into nested groups/arrays, rendering per-leaf confidence at every level.

**Checkpoint**: Reviewer sees the full nested structure with confidence intact; flat records unchanged.

---

## Phase 6: User Story 4 — Deliver via an author-written body template (P2)

**Goal**: For POST/PUT/PATCH routes the author writes the customer's exact JSON shape with Jinja placeholders; on delivery the route renders it with nesting preserved end-to-end, replacing the fixed `{general, email, data}` envelope. Existing routes stay byte-identical.

**Independent Test**: Configure a POST route with a body template referencing metadata + extracted-data placeholders; deliver a nested record; assert the rendered payload deep-equals the target shape.

**Depends on**: US2 (nested extracted data) + US1 (field tree). Reuses US3-era recursive data build.

### Tests for User Story 4 ⚠️ (write first, ensure they FAIL)

- [ ] T028 [P] [US4] Backend render test: `render(template_str, context)` with `{metadata, data}` produces a payload that deep-equals the `TYSON.json` shape — each cargo line owns only its own goods, each good its own packing (SC-004); `{{ data.CargoLines | tojson }}` and quoted scalar placeholders resolve correctly.
- [ ] T029 [P] [US4] Backend regression test: existing flat/single-level routes produce byte-identical delivery output (FR-015 / SC-005).
- [ ] T030 [P] [US4] Backend test: `render()` uses `SandboxedEnvironment` + `StrictUndefined` — an unknown variable raises (SSTI/attribute-traversal guard, tp §6.1).

### Implementation for User Story 4

- [ ] T031 [US4] Add the body-template field to `OutputRoute` in `backend/django/apps/document_entries/models.py` (author-written Jinja, used for body-carrying methods). Per tp §3.3 — supersedes the fixed envelope for those routes; existing routes unaffected.
- [ ] T032 [US4] Generate the migration for the new `OutputRoute` field via `makemigrations document_entries` (NEVER hand-write).
- [ ] T033 [US4] Implement `render(template_str, context) -> str` in the VWE-1521 `TemporaryEndpointExecutor` (per tp §6 Decisions — NOT a new `document_entries` module): thin string+dict→string, no DE coupling, `SandboxedEnvironment` + `StrictUndefined`. `context = {"metadata": <dict §6.4>, "data": <nested dict>}`.
- [ ] T034 [US4] In `backend/django/apps/document_entries/services.py`: make `_build_data_section` (~l.761) recursive — value → scalar via `_apply_field_transform`; single-object group → dict (omit if optional + all descendants missing, made recursive); repeatable group → list grouped by `group_item_path` index; feed it as `data` into `render()`.
- [ ] T035 [US4] Build the `metadata` context root (tp §6.4): `metadata.id`, `metadata.processed_at`, `metadata.email.{id,received_at,from,to,cc,subject}` from the extraction record + source email; wire delivery to render the body template instead of the fixed envelope for body-carrying methods.
- [ ] T036 [US4] In `client/src/app/documentEntry/` Output Route dialog: show a body-template editor when a body-carrying method (POST/PUT/PATCH) is selected; not required otherwise. Preview/test reuse existing panels (VWE-1521 render path).

**Checkpoint**: A delivered nested payload deep-equals the customer target shape (SC-004); existing routes regression-free.

---

## Phase 7: User Story 5 — Field-usage feedback and save-time validation (P3)

**Goal**: The Output Route dialog shows a helper panel listing offered metadata vars + all configured groups & fields (highlighted when referenced); save is gated on template validity.

**Independent Test**: Reference some but not all configured fields → unreferenced-field warning; introduce a typo'd placeholder → blocking error.

**Depends on**: US4 (body template + render path).

### Tests for User Story 5 ⚠️ (write first, ensure they FAIL)

- [ ] T037 [P] [US5] Backend save-validation test (FR-014): `TemplateSyntaxError` / `UndefinedError` (unresolvable placeholder) → hard error blocks save; rendered output not JSON-parseable → non-blocking warning; valid template leaving fields unreferenced → non-blocking warning, save proceeds.
- [ ] T038 [P] [US5] Backend test: save-time render uses a mock context seeded with the full runtime key set (`metadata` §6.4 + `data` from configured fields) so `StrictUndefined` catches genuine unknowns but valid `{{ metadata.id }}` / `{{ metadata.email.received_at }}` pass (tp §6.2).

### Implementation for User Story 5

- [ ] T039 [US5] In `backend/django/apps/document_entries/serializers.py` (OutputRoute serializer): validate the body template at save via the shared mock-context builder — hard error on `TemplateSyntaxError`/`UndefinedError`; return a non-blocking warning for non-JSON output and for unreferenced configured fields.
- [ ] T040 [US5] Implement the referenced-variable detector: `jinja2.meta.find_undeclared_variables(env.parse(src))` + AST walk for attribute access on `data`/`metadata` (e.g. `data.CargoLines`, `metadata.email.subject`); a whole-object `{{ data }}` counts every nested field referenced (tp §6.3, Open-item #2: flag referenced node + descendants).
- [ ] T041 [US5] In `client/src/app/documentEntry/` Output Route dialog: render the helper panel (metadata vars + all groups & fields, highlighted when referenced); non-blocking JSON validation (render mock context → `JSON.parse` → warning if it fails); gate Save only on template validity (syntax / unresolvable placeholder), not JSON validity.

**Checkpoint**: All five stories independently functional.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T042 [P] Run the structural quality gate (tp §8 / VWE-1496 fixture-gate pattern): the DLG/TYSON fixture set must pass for correct cargo-line count, correct goods-per-line, zero cross-item leakage, zero invented optional groups — before rollout. Confidence is NOT the gate. Confirm the fixture set (Open-item #3).
- [ ] T043 [P] Run `quickstart.md` end-to-end walkthrough (author → extract → review → deliver) against a nested Document Type.
- [ ] T044 Backend lint/format: `ruff check . --fix && ruff format .` from `backend/`; vw-llm-app equivalent.
- [ ] T045 Full backend + vw-llm-app test suites green via docker compose from `backend/` and vw-llm-app pytest; no `TODO(0)` remaining.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: depends on Setup; BLOCKS US2 and US3 (both consume `group_item_path`).
- **US1 (Phase 3)**: depends on Setup only (field_config/serializer/reconcile + frontend authoring). MVP.
- **US2 (Phase 4)**: depends on Foundational (Phase 2).
- **US3 (Phase 5)**: depends on Foundational (Phase 2); reads US1-authored trees.
- **US4 (Phase 6)**: depends on US2 (nested data) + US1 (tree); reuses recursive data build.
- **US5 (Phase 7)**: depends on US4 (template + render path).
- **Polish (Phase 8)**: depends on all desired stories.

### Within Each Story

- Tests written first and FAIL before implementation (backend/llm only).
- Models/migration → services/serializers → endpoints → frontend.

### Parallel Opportunities

- All `[P]` test tasks within a story run in parallel.
- US1 (backend + frontend authoring) and US2 (vw-llm-app extraction) touch disjoint code and can run in parallel once Phase 2 is done — different developers.
- Frontend type change T013 (US1) unblocks T026/T027 (US3) frontend recursion.

---

## Parallel Examples

```bash
# US1 tests together (backend):
Task: T006 serializer round-trip of full DLG tree
Task: T007 depth>5 rejected / depth==5 accepted
Task: T008 150-leaf cap + per-level min/max_items
Task: T009 reconcile re-parent/delete round-trip

# US2 tests together (vw-llm-app):
Task: T016 group_item_path values across levels
Task: T017 per-level padding/capping/unknown-codename
Task: T018 absent-optional + truncation retry/flag
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1 Setup → Phase 3 US1.
2. **STOP and VALIDATE**: author + save + round-trip the full DLG tree (SC-001). Demo.

### Incremental Delivery

1. Setup + Foundational → foundation ready.
2. US1 (author tree) → MVP.
3. US2 (extraction) → nested capture, structural gate.
4. US3 (review render) → human review surface.
5. US4 (body template delivery) → customer-shape payload (SC-004).
6. US5 (helper panel + save validation) → quality-of-life.

### Parallel Team Strategy

After Phase 2: Dev A → US1 (Django + client authoring); Dev B → US2 (vw-llm-app); then US3 (Django + client review) once Phase 2 lands; US4/US5 follow US2.

---

## Notes

- `technical_proposals/tp_vwe1609_improved_nesting.md` is the canonical source for wire contracts and design decisions; sections cited inline per task.
- Never hand-write migrations — always `makemigrations` (project rule).
- Run backend tests via docker compose from `backend/` (not the local poetry env; not repo root).
- No frontend (client) tests unless explicitly requested.
- `render()` deliberately lives in the VWE-1521 `TemporaryEndpointExecutor`, kept dependency-light so it lifts into the generic `api_integrations` layer later (tp §6 Decisions).
- Open items to confirm before/at rollout: (1) no in-flight prod extraction records before dropping `group_item_index`; (2) helper-panel granularity for dotted `data.<group>` chains; (3) the exact fixture set for the structural quality gate.
