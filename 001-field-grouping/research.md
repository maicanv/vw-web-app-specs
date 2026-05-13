# Research: Document Type Field Grouping

**Phase**: 0 — Codebase survey  
**Date**: 2026-05-12  
**Feeds into**: plan.md, data-model.md, contracts/

---

## Decision 1: Group storage model

**Decision**: New `DocumentTypeGroup` model as a FK target; `DocumentTypeField` gets a nullable `group` FK (null = header level) plus a nullable `item_index` is NOT stored on the field — instead, extracted group items are stored via a new `ExtractionGroupItem` model (FK to ExtractionRecord + DocumentTypeGroup + `item_index`), and `ExtractedFieldValue` gets a nullable `group_item` FK.

**Wait — YAGNI re-evaluation**: Adding a full `ExtractionGroupItem` model adds complexity. The simpler approach: add `group` (FK to `DocumentTypeGroup`, nullable) and `group_item_index` (IntegerField, nullable) directly to `ExtractedFieldValue`. Header-level fields have both null. Repeatable group fields have group + 0-based index. Single object group fields have group + NULL index (only one "item").

**Rationale**: Mirrors how `ExtractedFieldValue` already stores `field_codename`/`field_name`/`field_type` as denormalised snapshots. Querying all values for a record and grouping them in the serializer is already how the current list is built — we extend that pattern. No new model required.

**Alternatives considered**:
- Separate `ExtractionGroupItem` join table: more normalised but adds a model and join query with no benefit at current scale.
- JSON blob on ExtractionRecord: loses queryability and breaks confidence calculation.

---

## Decision 2: Extraction prompt restructuring (vw-llm-app)

**Decision**: Extend the existing `_extract()` method in `document_entry_side_effect_handler.py`. Pass groups as a separate `groups_json` parameter to the prompt template. Update `_ExtractionResult` Pydantic model to include a `groups` dict mapping group codenames to either a list of item dicts (repeatable) or a single item dict (single object).

**Rationale**: The current flow already builds `schema_json` from fields and passes it to the prompt. Adding `groups_json` is a parallel extension. The `structured_output(_ExtractionResult)` call constrains the LLM to the declared shape — updating the Pydantic model automatically updates the JSON schema the LLM receives.

**Alternatives considered**:
- Separate extraction call per group: more token-efficient for very large documents but doubles LLM calls and complicates partial-failure handling.
- Flat extraction + post-processing grouping: loses the structural hint to the LLM; proven ineffective for nested structures.

---

## Decision 3: Record detail API response shape

**Decision**: Replace `field_values: [...]` with a structured object:
```json
{
  "header": [...],
  "groups": {
    "<codename>": {
      "name": "...",
      "kind": "repeatable" | "single_object",
      "optional": false,
      "items": [[...], [...]]   // repeatable: array of arrays
      // OR
      "fields": [...]           // single_object: flat array
    }
  }
}
```
Hard cutover — `field_values` key removed. All consumers updated in the same release.

**Rationale**: Confirmed via clarification Q1. Flat `field_values` list consumed only by: `ExtractionRecordDetailPage.tsx`, `DocumentEntryRenderer.tsx`, internal API tests. All are in-product and can be migrated simultaneously.

**Affected consumers identified**:
- `ExtractionRecordDetailSerializer` in `extraction_record_view_set.py`
- `ExtractionRecordDetailPage.tsx` (TypeScript type `ExtractionRecordDetail`)
- `DocumentEntryRenderer.tsx` (TypeScript type `ExtractionEntry`)
- Any API tests referencing `field_values`

---

## Decision 4: Frontend group management UI (FieldsStep)

**Decision**: Extend `FieldsStep.tsx` with a two-panel layout — header fields panel + groups panel. Groups are displayed as collapsible sections. Fields can be moved between header and group via a context menu / drag-drop. The existing `hello-pangea/dnd` drag-drop library is already installed and used in `FieldsStep.tsx`.

**Rationale**: Reuses existing drag-drop infrastructure. Avoids introducing a new library. The existing `MAX_FIELDS_PER_DOCUMENT_TYPE` constant maps to the 50-field limit (FR-011); add a `MAX_GROUPS_PER_DOCUMENT_TYPE = 10` constant.

---

## Decision 5: Frontend grouped review rendering

**Decision**: 
- Single object groups → `GroupCard` component (new, reusable)
- Repeatable groups → `RepeatableGroupSection` component (new, with Mantine `Tabs` or `Accordion`)
- Reuse existing `ConfidenceBar` component for both field-level and group-level confidence
- Group-level confidence = average of all field confidences in the group (computed client-side from the returned fields array)

**Rationale**: `ConfidenceBar` is already shared between `ExtractionRecordDetailPage` and `DocumentEntryRenderer`. Creating `GroupCard` and `RepeatableGroupSection` as shared components under `client/src/app/documentEntry/components/` keeps both pages in sync.

---

## Key file locations (confirmed)

| Concern | File |
|---------|------|
| Django models | `backend/django/apps/document_entries/models.py` |
| Django serializers | `backend/django/apps/document_entries/serializers.py` |
| Document type viewset | `backend/django/apps/document_entries/document_type_view_set.py` |
| Extraction record viewset | `backend/django/apps/document_entries/extraction_record_view_set.py` |
| Extraction LLM handler | `vw-llm-app/actions/document_entry_side_effect_handler.py` |
| Prompt templates | `vw-llm-app/prompts/templates.py` |
| Fields wizard step | `client/src/app/documentEntry/documentTypes/steps/FieldsStep.tsx` |
| Record detail page | `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx` |
| History renderer | `client/src/app/application/history/actionRenderers/DocumentEntryRenderer.tsx` |
