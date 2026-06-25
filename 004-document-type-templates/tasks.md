# Tasks: Document Type Templates

**Input**: Design documents from `/specs/004-document-type-templates/`
**Prerequisites**: plan.md ✓, spec.md ✓, research.md ✓, data-model.md ✓, contracts/ ✓
**Source of truth**: `technical_proposals/tp_vwe1584_document_type_templates.md`

**Organization**: Aligned to the proposal's Phases A → B → C. Phase A + B must land before the frontend picker (Phase C) can be implemented.

## Table of Contents

- [Phase 1: Foundational — Model, Admin, Seed (Phase A)](#phase-1-foundational--model-admin-seed-phase-a)
- [Phase 2: Read-only API (Phase B)](#phase-2-read-only-api-phase-b)
- [Phase 3: US1 — Create-flow Picker (Phase C)](#phase-3-us1--create-flow-picker-phase-c)
- [Phase 4: US2 — Superadmin Django Admin (Phase A continuation)](#phase-4-us2--superadmin-django-admin-phase-a-continuation)
- [Phase 5: Polish & Cross-Cutting Concerns](#phase-5-polish--cross-cutting-concerns)
- [Dependencies & Execution Order](#dependencies--execution-order)
- [Implementation Strategy](#implementation-strategy)

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (US1 = picker, US2 = admin management)

---

## Phase 1: Foundational — Model, Admin, Seed (Phase A)

**Purpose**: `DocumentTypeTemplate` model, `TemplatePayload`/`TemplateField` Pydantic schema, Django admin, seed data.
Blocks all phases; independently demoable — superadmin creates/deactivates templates in admin.

**Independent Test**: In Django admin, create a global template "Test Invoice" with a 3-field payload, confirm it saves. Deactivate it, confirm `is_active=False` in the DB.

- [x] T001 Add `TemplatePayload`, `TemplateField`, and `TemplateFieldChild` Pydantic models in `backend/django/apps/document_entries/template_schema.py` — `TemplatePayload` = `{description: str, instructions: str, fields: list[TemplateField]}`; reuse `FieldConfig` union from `field_config.py`, `CODENAME_REGEX`, `CODENAME_MAX_LENGTH`, `MAX_FIELDS_PER_DOCUMENT_TYPE` constants; enforce codename uniqueness across full tree, ≤50 fields, depth-2 only (children cannot have children)
- [x] T002 Add `DocumentTypeTemplate` model to `backend/django/apps/document_entries/models.py` — fields: `organisation` (nullable FK → Organisation, CASCADE), `name` (varchar 80), `blurb` (varchar 300), `icon` (varchar 50, blank), `payload` (`SchemaField(TemplatePayload)`, holds `{description, instructions, fields}`), `is_active` (bool, default True); inherits `BaseModel` (hashid, created_at, updated_at). Add a `field_count` property/helper = total tree size of `payload.fields` (top-level + all children) reused by both admin and serializer
- [x] T003 Generate and review migration for `DocumentTypeTemplate` in `backend/django/apps/document_entries/migrations/`
- [x] T004 Register `DocumentTypeTemplateAdmin` in `backend/django/apps/document_entries/admin.py` — list display: name, organisation, is_active, field_count, created_at; list filter: is_active, organisation; search: name; `field_count` read-only method reuses the model's total-tree count (top-level + children), not `len(payload.fields)`
- [x] T005 Add `DocumentTypeTemplateFactory` to `backend/django/apps/document_entries/factories.py` — builds a valid `payload` (description, instructions, 1–3 valid TemplateFields); params for `organisation=None` (global) and `is_active=True`
- [x] T006 Implemented as the idempotent management command `seed_document_type_templates` (NOT a data migration — `.claude/rules/backend.md` forbids backfills in migrations) — seeds 8 global templates (Invoice, Purchase Order, Delivery Note, Remittance Advice, Credit Note, Contract, HR Onboarding Form, Bill of Lading) using `get_or_create(name=..., organisation=None)`; each has a realistic `payload` (description, instructions, representative `fields` list reusing `FieldConfig` shapes)
- [x] T007 [P] Write backend tests for `TemplatePayload`/`TemplateField` schema validation in `backend/django/tests/test_apps/test_document_entries/test_document_type_template_schema.py` — cover: invalid codename rejected, duplicate codename across tree rejected, >50 fields rejected, child-with-children rejected, valid group with children accepted

**Checkpoint**: Phase A complete — superadmin can create, edit, and deactivate templates in Django admin; seed data populates the DB.

---

## Phase 2: Read-only API (Phase B)

**Purpose**: List + retrieve endpoints with org-visibility scoping, single serializer (full `payload` inline). Blocks the frontend picker.

**Independent Test**: Hit `GET /api/v1/document-entries/document-type-templates/` as an editor-role user; confirm only active global + own-org templates are returned, each with full `payload`. Hit with a cross-org id; confirm `404`.

- [x] T008 Add a single `DocumentTypeTemplateSerializer` to `backend/django/apps/document_entries/serializers.py` — serves both list + retrieve (no list/detail split, per proposal §3/§5): id, name, blurb, icon, `field_count`, full `payload` (description, instructions, fields nested tree); `field_count = SerializerMethodField` reusing the model's total-tree count (top-level + all children); IDs hashid-encoded
- [x] T009 Create `DocumentTypeTemplateViewSet` (ReadOnlyModelViewSet) in `backend/django/apps/document_entries/document_type_template_view_set.py` — custom `get_queryset`: `is_active=True AND (organisation__isnull=True OR organisation=request.organisation)`, ordered by `name`; the same `DocumentTypeTemplateSerializer` for both actions; return `404` (not `403`) for hidden/cross-org ids
- [x] T010 Add access policy for templates in `backend/django/common/access_policy/` (either a new `document_entry_template_access_policy.py` or extend the existing policy) — gate `list` and `retrieve` on `document_types:edit`; `403` for authenticated users lacking the permission, `401` for unauthenticated
- [x] T011 Register the template viewset in `backend/django/apps/document_entries/urls.py` — router path `document-type-templates/` under the existing `document-entries` namespace
- [x] T012 [P] Write backend tests for the viewset in `backend/django/tests/test_apps/test_document_entries/test_document_type_template_viewset.py` — cover: org isolation (no cross-org leak), global templates visible to all orgs, inactive templates excluded, list payload shape (full `payload` inline, with `field_count`), retrieve payload shape (same object), `404` for hidden/cross-org id, `403` without `document_types:edit` permission

**Checkpoint**: Phase B complete — API returns active org-visible templates; cross-org and inactive templates are hidden.

---

## Phase 3: US1 — Create-flow Picker (Phase C)

**Goal**: Template picker modal before the creation wizard; wizard pre-fill from selected template.

**Independent Test**: Click "New Document Type", select the "Invoice" template, proceed — confirm wizard opens with description/instructions/fields pre-filled and the name field is blank. Select "Start from scratch" — confirm wizard opens blank.

- [x] T013 Add `DocumentTypeTemplate` TypeScript type to `client/src/types/documentEntry.ts` — single shape (id, name, blurb, icon, field_count, `payload: { description, instructions, fields }` where `fields` maps to `DocumentTypeFieldFormValues[]`); no separate list/detail types
- [x] T014 (Implemented as inline `useApiQuery` per frontend rule — no extracted hook file.) API query for templates — list (`useDocumentTypeTemplates`, returns full payloads) and a retrieve hook (`useDocumentTypeTemplate(id)`) for the deep-link / refresh case only, using the project's `useApiQuery` convention; place alongside other document-type queries in the existing query files
- [x] T015 [P] Create `client/src/app/documentEntry/documentTypes/templates/templateIcon.tsx` — small map from icon identifier string to a Tabler icon component; export a `TemplateIcon` component that falls back to a generic document icon for unknown identifiers
- [x] T016 [P] Create `client/src/app/documentEntry/documentTypes/templates/TemplateFieldPreview.tsx` — renders the `payload.fields` tree for a selected template: top-level fields (name + type badge), group children indented below their parent group; shows an empty/placeholder state when "Start from scratch" is active
- [x] T017 Create `client/src/app/documentEntry/documentTypes/templates/TemplatePickerModal.tsx` — two-panel layout (Mantine): left panel = card grid (all templates at once, no pagination, sized for ~20 cards) + "Start from scratch" card pre-selected; right panel = persistent `TemplateFieldPreview`; on card select: render the preview from the already-loaded list item's `payload` (**no per-card fetch**); on "Continue": navigate to `/document-entries/types/new` passing `location.state.templateId` (or nothing for scratch); on "Cancel": close modal
- [x] T018 Update `client/src/app/documentEntry/documentTypes/DocumentTypesListPage.tsx` — wire both "New Document Type" entry points (header button + table toolbar) to open `TemplatePickerModal` instead of navigating directly to the create route
- [x] T019 Update `client/src/app/documentEntry/documentTypes/DocumentTypeCreatePage.tsx` — on mount read `location.state.templateId`; if present, pre-fill from the list cache (the picker already loaded the full `payload`), falling back to the retrieve hook on deep-link / refresh; spread `payload.description`, `payload.instructions`, and `payload.fields` into form initial values; name field always starts blank regardless of template; if no state → blank form (existing behavior unchanged)

**Checkpoint**: Phase C complete — picker opens, templates are browsable with side-panel preview, wizard is pre-filled correctly, name stays blank.

---

## Phase 4: US2 — Superadmin Django Admin (Phase A continuation)

**Note**: The core model + basic admin landed in Phase 1 (T004). These tasks complete the admin UX to the level needed for superadmins to manage and monitor templates comfortably.

**Goal**: Superadmins can create, edit, scope, deactivate, and browse templates — including JSON field authoring — without needing raw DB access.

**Independent Test**: Create a global template "Test Invoice" with 3 fields via Django admin. Confirm it appears in the picker. Deactivate it. Confirm it no longer appears.

- [x] T020 [US2] Enhance `DocumentTypeTemplateAdmin` in `backend/django/apps/document_entries/admin.py` — add `fieldsets` grouping (Identity: name/blurb/icon; Visibility: organisation/is_active; Content: payload); add `readonly_fields = ("created_at", "updated_at", "field_count")`; ensure the `payload` JSON textarea has `help_text` describing the `{description, instructions, fields}` shape and linking to the `TemplatePayload` schema; superadmin-only by inheriting `SuperAdminMixin` or setting `def has_module_perms` / `def has_change_permission` to require `is_superuser`

**Checkpoint**: US2 complete — superadmins can fully manage templates from Django admin.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [x] T021 [P] Run full backend test suite via `docker compose exec django pytest apps/document_entries/` from `backend/` and fix any failures
- [x] T022 [P] Run frontend type-check (`npm run type-check` from `client/`) and lint; fix any errors introduced by new types and components
- [ ] T023 Walk through `specs/004-document-type-templates/quickstart.md` end-to-end and verify all acceptance scenarios pass

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Foundational — A)**: No dependencies. Start immediately.
- **Phase 2 (API — B)**: Requires Phase 1 complete (model must exist for queryset/serializer).
- **Phase 3 (Picker — C / US1)**: Requires Phase 2 complete (API must be available for frontend queries).
- **Phase 4 (Admin — US2)**: T020 depends on T002/T004 from Phase 1 (model and basic admin must exist). Can be done alongside Phase 2 or 3 as long as Phase 1 is done.
- **Phase 5 (Polish)**: Requires all prior phases complete.

### Within Phase Dependencies

- T002 depends on T001 (model imports `TemplateField` schema)
- T003 depends on T002 (migration from model)
- T004 depends on T002 (admin references model)
- T006 depends on T002 + T003 (data migration after model migration)
- T008 depends on T001 + T002 (serializer uses schema and model)
- T009 depends on T002 + T008 (viewset uses model + serializer)
- T010 depends on T009 (policy applied to viewset)
- T011 depends on T009 + T010 (URL registration)
- T014 depends on T013 (hooks use TS types)
- T017 depends on T015 + T016 + T014 (modal uses icon, preview, and the list query)
- T018 depends on T017 (list page opens the modal)
- T019 depends on T013 + T014 (create page uses types and the list cache / retrieve hook)

### Parallel Opportunities

Within Phase 1: T001 can start immediately; T004, T005 can start in parallel after T002+T003; T007 can start in parallel after T001.

Within Phase 2: T008, T012 can start together after Phase 1; T009 after T008; T010+T011 after T009.

Within Phase 3: T013+T015+T016 can start together after Phase 2; T014 after T013; T017 after T015+T016+T014; T018+T019 after T017.

---

## Parallel Example: Phase 3 Kick-off

```
# Once Phase 2 is done, launch these three in parallel:
Task T013: Add TS types to client/src/types/documentEntry.ts
Task T015: Create templateIcon.tsx
Task T016: Create TemplateFieldPreview.tsx

# Then, once T013 is done:
Task T014: Add API query hooks

# Then, once T014 + T015 + T016 are done:
Task T017: Create TemplatePickerModal.tsx

# Finally, T018 and T019 can run in parallel after T017:
Task T018: Update DocumentTypesListPage.tsx
Task T019: Update DocumentTypeCreatePage.tsx
```

---

## Implementation Strategy

### MVP First

1. Complete Phase 1 (model + admin + seed) — demoable with admin UI
2. Complete Phase 2 (API) — backend complete; independently testable
3. Complete Phase 3 US1 (picker) — **full user value delivered**
4. **STOP and VALIDATE**: walk quickstart.md, confirm acceptance scenarios
5. Complete Phase 4 (admin polish) — improves superadmin DX
6. Phase 5 polish + deploy

### Key Constraints (from proposal)

- **`OrgQuerySetMixin` cannot be reused** for the template viewset — use a custom one-line `get_queryset` with the nullable-org predicate.
- **`404`, not `403`** for hidden/cross-org template ids — never confirm existence.
- **Name stays blank** in the wizard regardless of template — enforced in `DocumentTypeCreatePage`, never from the API.
- **One serializer, no list/detail split** — the proposal deliberately uses a single `DocumentTypeTemplateSerializer` that returns the full `payload` inline for both list and retrieve. The list already carries every payload, so the picker renders the side-panel preview and pre-fills the wizard with **no per-card detail fetch**; the retrieve endpoint exists only for deep-link / refresh.
- **Single opaque `payload`** — store `{description, instructions, fields}` as one `SchemaField(TemplatePayload)`, not three separate columns; it maps 1:1 onto `DocumentTypeFormValues` (minus name).
- **Seed migration must be idempotent** — use `get_or_create(name=..., organisation=None)`.
