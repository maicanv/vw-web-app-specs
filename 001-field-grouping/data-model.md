# Data Model: Document Type Field Grouping

**Phase**: 1 — Design  
**Date**: 2026-05-12

---

## Modified Model: `DocumentTypeField`

**Added field** (nullable for backwards compatibility):

```
DocumentTypeField
├── ... (all existing fields unchanged) ...
└── parent_field  FK → self (CASCADE, null=True, blank=True, related_name="children")
                  — null means top-level (header) field
                  — non-null means this field belongs to the parent group field
```

**Migration note**: All existing fields get `parent_field=NULL` (top-level). No data loss.

### New `FieldConfig` variants

Two new variants added to the `field_type` / `FieldConfig` discriminated union:

```
RepeatableGroupConfig
├── description   TextField (default="") — extraction hint to the model
├── min_items     IntegerField (default=0) — minimum items to extract
└── max_items     IntegerField (default=200) — hard cap on items

SingleObjectGroupConfig
├── description   TextField (default="") — extraction hint to the model
└── optional      BooleanField (default=False) — if True and absent, omit from output
```

A `DocumentTypeField` with `field_type = repeatable_group` or `field_type = single_object_group`
acts as a group container. Its children (via `parent_field`) are the member fields.

**No separate `FieldGroup` model or DB table.**

---

## Modified Model: `ExtractedFieldValue`

**Added field** (nullable for backwards compatibility):

```
ExtractedFieldValue
├── ... (all existing fields unchanged) ...
└── group_item_index  IntegerField (null=True, blank=True)
                      — 0-based index within a repeatable group's items
                      — null for top-level and single_object_group fields
```

The parent group field is reachable via `document_type_field.parent_field` — no separate group FK needed.

**Group item semantics**:
- Top-level field: `parent_field=NULL`, `group_item_index=NULL`
- Single object group member: `parent_field=<group_field>`, `group_item_index=NULL`
- Repeatable group member, item 0: `parent_field=<group_field>`, `group_item_index=0`
- Repeatable group member, item 1: `parent_field=<group_field>`, `group_item_index=1`

**Migration note**: All existing `ExtractedFieldValue` rows get `group_item_index=NULL`.

---

## Limits (constants)

```python
MAX_FIELDS_PER_DOCUMENT_TYPE = 50   # tree-wide: top-level + all group children combined
MAX_ITEMS_PER_REPEATABLE_GROUP = 200
```

No separate group cap — group fields count toward the 50-field tree-wide limit.

---

## Immutability Rules (extended from existing field rules)

When `DocumentType.status == ACTIVE`:
- Group field `codename` → read-only (cannot be updated)
- Group `DocumentTypeField` rows → cannot be deleted
- Child `DocumentTypeField` rows → cannot be deleted, cannot be reparented

Same enforcement point as existing field immutability: `DocumentTypeSerializer` / `DocumentTypeUpdateSerializer` validation.

Delete is blocked on active types, so CASCADE on `parent_field` is safe — it only fires
during schema teardown or hard-delete paths that are already gated.

---

## State Transitions

```
Group field lifecycle mirrors DocumentTypeField:
  Draft DocumentType → group field can be created / renamed / deleted / reordered; children can be added/removed
  Active DocumentType → group field is frozen (codename immutable, cannot delete, cannot remove children)
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

**After (replacement)** — single unified `fields` list, polymorphic by `field_type`:
```json
{
  "fields": [
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
    },
    {
      "field_codename": "invoice_lines",
      "field_name": "Invoice Lines",
      "field_type": "repeatable_group",
      "description": "Table of line items.",
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
    {
      "field_codename": "sender",
      "field_name": "Sender",
      "field_type": "single_object_group",
      "description": "Sending party details.",
      "optional": true,
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
  ]
}
```

**Optional single object group not found**:
```json
{
  "field_codename": "sender",
  "field_name": "Sender",
  "field_type": "single_object_group",
  "optional": true,
  "fields": null,
  "not_found": true
}
```

---

## Extraction Payload to LLM (vw-llm-app)

**Extended `doc_type` dict passed to `_extract()`** — single nested `fields` list:
```json
{
  "name": "...",
  "description": "...",
  "instructions": "...",
  "fields": [
    {"codename": "invoice_number", "name": "Invoice Number", "type": "string", "required": true},
    {
      "codename": "invoice_lines",
      "name": "Invoice Lines",
      "type": "repeatable_group",
      "description": "Table of line items. May span multiple rows.",
      "min_items": 0,
      "max_items": 200,
      "fields": [
        {"codename": "line_description", "name": "Description", "type": "string", "required": false}
      ]
    }
  ]
}
```

**Updated `_ExtractionResult` Pydantic model** — single recursive `fields` key:
```python
class _FieldValue(BaseModel):
    value: Any = None

class _GroupItem(BaseModel):
    fields: dict[str, Any]  # field_codename → extracted value

class _RepeatableGroupResult(BaseModel):
    items: list[_GroupItem] | None = None

class _SingleObjectGroupResult(BaseModel):
    fields: _GroupItem | None = None  # None if optional and not found

class _ExtractionResult(BaseModel):
    fields: dict[str, Any]  # recursive: top-level codename → value or nested group result
```
