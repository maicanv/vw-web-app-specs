# Implementation Plan: Document Type Templates

**Branch**: `VWE-1584-story-document-entry-templates` | **Date**: 2026-06-24 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `specs/004-document-type-templates/spec.md`

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Phased Delivery](#phased-delivery)
- [Complexity Tracking](#complexity-tracking)

## Summary

Add a library of pre-configured Document Type templates that pre-fill the creation wizard, so users start common document types (Invoice, PO, Contract, …) without manual setup. A new `DocumentTypeTemplate` model stores name, blurb, icon, description, instructions, and a Pydantic-validated `fields` JSON blob (reusing the existing `FieldConfig` union). Templates are global (org-null) or org-scoped, deactivatable, and managed by superadmins in Django admin. A read-only API serves active, org-visible templates to a **create-flow picker modal**; selecting a card shows a **persistent side panel** with the full field tree (group children indented) so the user can preview before committing, then proceeding navigates into the existing wizard with values pre-filled (name always blank).

**Scope note**: The "Load from template" edit flow (Replace/Merge into an existing Document Type) is **out of scope** — templates apply at create time only.

## Technical Context

**Language/Version**: Python 3.12 (Django 4.2, DRF), TypeScript 5 (React 18)
**Primary Dependencies**: DRF, `django-pydantic-field` (`SchemaField`), drf-access-policy, Mantine UI, TanStack Query, React Router
**Storage**: PostgreSQL (one new table `document_entries_documenttypetemplate`)
**Testing**: pytest (backend, run via `docker compose exec django pytest` from `backend/`), Vitest (frontend)
**Target Platform**: Linux server (Django + FastAPI share ORM), browser SPA
**Project Type**: Web application (Django backend + React frontend in a monorepo)
**Performance Goals**: Picker reachable < 2s (SC-001); template list is small and unpaginated — layout sized for ~20 templates in one view (FR-011)
**Constraints**: Strict org isolation (constitution I); reuse existing `document_entries` patterns (constitution II); no PII in templates
**Scale/Scope**: ~8 seeded global templates + occasional org-scoped ones; depth-2 field trees, ≤ 50 fields per template

## Constitution Check

*GATE: re-checked after Phase 1 design — PASS.*

| Principle | Assessment |
|-----------|------------|
| **I. Protect Sensitive Data** | Templates carry config only, no PII. Picker queryset enforces `is_active AND (org IS NULL OR org = request.organisation)`; retrieve returns `404` (not `403`) for hidden/cross-org ids → no existence leak. PASS. |
| **II. Respect the Architecture** | Mirrors `document_entries` conventions: `BaseModel` + hashid, `SchemaField(FieldConfig)` reuse, access-policy gating, namespace-mounted router. One deliberate deviation (JSON-blob fields vs relational) — justified in Complexity Tracking. PASS. |
| **III. Test What Matters** | Backend: org-scoping, active filter, global visibility, `fields` schema validation, `404` on hidden ids. Frontend: template→form mapping and "name stays blank" (the behavior most likely to regress). PASS. |
| **IV. Ship Incrementally** | Two independently shippable slices (model+admin+seed; then picker). YAGNI honored — read-only API, no merge logic, no extra constraints. PASS. |
| **V. Make Failures Visible** | Template fetch errors surfaced via existing `ApiError` component; admin authoring errors via Pydantic validation. No new async paths. PASS. |

No unjustified violations. Gate passes.

## Project Structure

### Documentation (this feature)

```text
specs/004-document-type-templates/
├── plan.md              # This file
├── spec.md              # Feature spec (clarified)
├── research.md          # Design decisions D1–D7
├── data-model.md        # DocumentTypeTemplate + TemplateField schema
├── quickstart.md        # End-to-end manual verification
├── contracts/
│   └── document-type-templates-api.md   # Read-only list/retrieve contract
└── checklists/
    └── requirements.md  # Spec quality checklist
```

### Source Code (repository root)

Backend — within the existing `apps/document_entries/` app (no new app):

```text
backend/django/apps/document_entries/
├── models.py                       # + DocumentTypeTemplate
├── template_schema.py              # + TemplateField / TemplateFieldChild Pydantic models (reuse FieldConfig + shared field validation)
├── serializers.py                  # + DocumentTypeTemplateListSerializer, DocumentTypeTemplateSerializer (field_count)
├── document_type_template_view_set.py   # + ReadOnlyModelViewSet (custom org-visible get_queryset)
├── urls.py                         # register the template viewset
├── admin.py                        # + DocumentTypeTemplateAdmin
├── factories.py                    # + DocumentTypeTemplateFactory
└── migrations/                     # + model migration, + idempotent seed data migration
backend/django/common/access_policy/
└── document_entry_template_access_policy.py   # gate on document_types:edit (or reuse existing policy)
backend/django/tests/test_apps/test_document_entries/
└── test_document_type_template_viewset.py     # scoping / visibility / validation tests
```

Frontend — within the existing `documentEntry/documentTypes/` feature:

```text
client/src/app/documentEntry/documentTypes/
├── DocumentTypesListPage.tsx       # "New Document Type" opens the picker modal (both entry points)
├── DocumentTypeCreatePage.tsx      # read location.state.templateId → fetch + pre-fill (name blank)
└── templates/
    ├── TemplatePickerModal.tsx     # two-panel: card grid (left) + field-preview side panel (right); navigates with router state
    ├── TemplateFieldPreview.tsx    # renders the selected template's field tree (group children indented)
    └── templateIcon.tsx            # icon-identifier → Tabler icon map with fallback
client/src/types/documentEntry.ts   # + DocumentTypeTemplate types (list + detail)
```

**Structure Decision**: Extend the existing `document_entries` Django app and `documentEntry/documentTypes` frontend feature — no new app or top-level module. This keeps templates co-located with the Document Type code they pre-fill and inherits its routing, access policy, and subscription gating.

## Phased Delivery

Aligned to the spec's user stories; each phase is independently testable.

**Phase A — Model, admin, seed (foundation; spec US2 / P2)**
- `DocumentTypeTemplate` model + migration; `TemplateField` Pydantic schema reusing `FieldConfig` and shared field-validation rules (codename regex, ≤50 fields, group bounds, depth-2).
- Django admin: editable template + JSON `fields` (validated); list/filter by `is_active` and organisation.
- `DocumentTypeTemplateFactory`; idempotent data migration seeding the 8 global templates.
- Tests: schema validation (bad codename, >50 fields, child-with-children rejected), admin save round-trip.
- *Demoable*: superadmin creates/deactivates templates; global vs org-scoped visible in admin.

**Phase B — Read-only API (foundation for picker)**
- `DocumentTypeTemplateViewSet` (ReadOnlyModelViewSet) with custom `get_queryset` (active + global-OR-own-org, ordered by name); list vs detail serializers; `field_count`.
- Access policy gating on `document_types:edit`; `404` for hidden/cross-org ids.
- Detail serializer returns the full `fields` tree (incl. group `children`) — single source for both the picker side-panel preview (FR-003a) and wizard pre-fill (FR-004). No separate preview endpoint.
- Tests: org isolation (no cross-org leak), active filter, global visibility, list payload shape, detail pre-fill payload, `404` behavior.

**Phase C — Create-flow picker (spec US1 / P1)**
- `TemplatePickerModal` opened from both "New Document Type" entry points on the list page; **two-panel layout** — card grid (left) with pre-selected "Start from scratch", persistent field-preview panel (right). All templates rendered at once, no pagination; layout sized for ~20 cards (FR-011, AS-6).
- On card select: fetch the template detail (`useApiQuery` keyed by id, cached) and render `TemplateFieldPreview` — top-level fields (name + type) plus group `children` indented under their parent (FR-003a, AS-3). "Start from scratch" selected → panel shows an empty/placeholder state.
- Proceed: navigate to `/types/new` with `location.state.templateId` (or none). `DocumentTypeCreatePage` on mount, if `templateId`, reuse the cached detail to pre-fill `description` / `instructions` / `fields` — **name left blank**; all values editable.
- `templateIcon` map with fallback; `DocumentTypeTemplate` TS types (list + detail).
- Tests: card-select populates the preview panel with the full tree (group children indented); template→form mapping keeps name blank; "start from scratch" yields blank form + empty preview; re-pick fully replaces state.

## Complexity Tracking

| Decision | Why Needed | Simpler/Other Alternative Rejected Because |
|----------|------------|-------------------------------------------|
| `fields` as Pydantic JSON blob instead of a relational `DocumentTypeTemplateField` | Templates are write-once snapshots, never queried/joined/reconciled; frontend wants nested JSON anyway; consistent with existing `SchemaField` usage. Less code. | A relational model adds a table, migration, inline admin, and a re-nesting serializer for data never queried relationally. Documented upgrade path exists (research D1) if admin authoring becomes painful. |
| Custom `get_queryset` instead of `OrgQuerySetMixin` | Need global-OR-own-org (`org IS NULL OR org = request.org`); the mixin filters strictly by a required org FK. | Reusing the mixin can't express the nullable-global case. The custom predicate is one line. |
| Reuse the detail endpoint for both side-panel preview and pre-fill | One read-only payload already carries the full field tree; the picker fetches it on select and hands the same cached object to the wizard. | A dedicated "preview" endpoint or embedding full `fields` in the list response duplicates the tree and bloats the list payload for cards never opened. |
