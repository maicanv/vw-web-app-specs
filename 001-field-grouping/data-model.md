# Data Model: Document Type Field Grouping

**Phase**: 1 — Design  
**Date**: 2026-05-12

---

## New Model: `FieldGroup`

```
FieldGroup
├── id                    UUID PK
├── document_type         FK → DocumentType (CASCADE, related_name="groups")
├── codename              CharField(100) — machine identifier, regex ^[a-zA-Z][a-zA-Z0-9_]*$
├── name                  CharField(200)
├── description           TextField (default="") — extraction hint to the model
├── kind                  CharField choices: REPEATABLE | SINGLE_OBJECT
├── optional              BooleanField (default=False) — only meaningful for SINGLE_OBJECT
├── min_items             IntegerField (default=0, null=True) — only for REPEATABLE
├── max_items             IntegerField (default=200, null=True) — only for REPEATABLE
├── display_order         IntegerField (default=0)
├── created_at            auto
└── updated_at            auto

Constraints:
  UNIQUE (document_type, codename)  — deferred, same pattern as DocumentTypeField
  CHECK: kind == REPEATABLE → optional ignored (handled by min_items=0)
  CHECK: max_items >= min_items when both set
  CHECK: max_items <= 200
```

---

## Modified Model: `DocumentTypeField`

**Added field** (nullable for backwards compatibility):

```
DocumentTypeField
├── ... (all existing fields unchanged) ...
└── group    FK → FieldGroup (SET_NULL, null=True, blank=True, related_name="fields")
             — null means header-level
```

**Migration note**: Existing fields all get `group=NULL` (header level). No data loss.

---

## Modified Model: `ExtractedFieldValue`

**Added fields** (both nullable for backwards compatibility):

```
ExtractedFieldValue
├── ... (all existing fields unchanged) ...
├── group             FK → FieldGroup (SET_NULL, null=True, blank=True)
│                     — snapshot reference; null for header-level values
└── group_item_index  IntegerField (null=True, blank=True)
                      — 0-based index within repeatable group items
                      — null for header-level and single-object group fields
```

**Group item semantics**:
- Header field: `group=NULL`, `group_item_index=NULL`
- Single object group field: `group=<group>`, `group_item_index=NULL`
- Repeatable group field, item 0: `group=<group>`, `group_item_index=0`
- Repeatable group field, item 1: `group=<group>`, `group_item_index=1`

**Migration note**: All existing `ExtractedFieldValue` rows get `group=NULL`, `group_item_index=NULL`.

---

## Limits (constants)

```python
MAX_FIELDS_PER_DOCUMENT_TYPE = 50   # existing — now counts header + all group fields
MAX_GROUPS_PER_DOCUMENT_TYPE = 10   # new
MAX_ITEMS_PER_REPEATABLE_GROUP = 200  # new
```

---

## Immutability Rules (extended from existing field rules)

When `DocumentType.status == ACTIVE`:
- `FieldGroup.codename` → read-only (cannot be updated)
- `FieldGroup` rows → cannot be deleted
- `DocumentTypeField` rows inside a group → cannot be deleted, cannot be moved out

Same enforcement point as existing field immutability: `DocumentTypeSerializer` / `DocumentTypeUpdateSerializer` validation.

---

## State Transitions

```
FieldGroup lifecycle mirrors DocumentTypeField:
  Draft DocumentType → group can be created / renamed / deleted / reordered
  Active DocumentType → group is frozen (codename immutable, cannot delete, cannot remove fields)
```

---

## API Shape: Record Detail Response

**Before (removed)**:
```json
{
  "field_values": [
    {
      "id": "...",
      "field_codename": "invoice_number",
      "field_name": "Invoice Number",
      "field_type": "string",
      "display_value": "INV-001",
      "raw_value": "INV-001",
      "confidence": 92,
      "is_critical": true,
      "display_order": 0
    }
  ]
}
```

**After (replacement)**:
```json
{
  "header": [
    {
      "id": "...",
      "field_codename": "invoice_number",
      "field_name": "Invoice Number",
      "field_type": "string",
      "display_value": "INV-001",
      "raw_value": "INV-001",
      "confidence": 92,
      "is_critical": true,
      "display_order": 0
    }
  ],
  "groups": {
    "invoice_lines": {
      "name": "Invoice Lines",
      "kind": "repeatable",
      "optional": false,
      "min_items": 0,
      "max_items": 200,
      "items": [
        [
          {
            "id": "...",
            "field_codename": "line_description",
            "field_name": "Description",
            "field_type": "string",
            "display_value": "Consulting services",
            "raw_value": "Consulting services",
            "confidence": 88,
            "is_critical": false,
            "display_order": 0
          }
        ]
      ]
    },
    "sender": {
      "name": "Sender",
      "kind": "single_object",
      "optional": true,
      "items": null,
      "fields": [
        {
          "id": "...",
          "field_codename": "sender_name",
          "field_name": "Name",
          "field_type": "string",
          "display_value": "ACME Corp",
          "raw_value": "ACME Corp",
          "confidence": 95,
          "is_critical": false,
          "display_order": 0
        }
      ]
    }
  }
}
```

**Group absent (optional single object not found)**:
```json
"groups": {
  "sender": {
    "name": "Sender",
    "kind": "single_object",
    "optional": true,
    "fields": null,
    "not_found": true
  }
}
```

---

## Extraction Payload to LLM (vw-llm-app)

**Extended `doc_type` dict passed to `_extract()`**:
```json
{
  "name": "...",
  "description": "...",
  "instructions": "...",
  "fields": [...],
  "groups": [
    {
      "codename": "invoice_lines",
      "name": "Invoice Lines",
      "description": "Table of line items. May span multiple rows.",
      "kind": "repeatable",
      "min_items": 0,
      "max_items": 200,
      "fields": [
        {"codename": "line_description", "name": "Description", "type": "string", "required": false}
      ]
    }
  ]
}
```

**Updated `_ExtractionResult` Pydantic model**:
```python
class _GroupItem(BaseModel):
    # keys = field codenames, values = extracted values
    __root__: dict[str, Any]

class _GroupResult(BaseModel):
    items: list[_GroupItem] | None = None   # repeatable
    fields: _GroupItem | None = None        # single_object

class _ExtractionResult(BaseModel):
    header: dict[str, Any]          # field_codename → value
    groups: dict[str, _GroupResult]  # group_codename → result
```
