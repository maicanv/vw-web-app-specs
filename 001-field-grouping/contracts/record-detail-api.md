# API Contract: Record Detail Endpoint

**Endpoint**: `GET /document-entries/records/{id}/`  
**Change type**: Breaking — hard cutover  
**Consumers**: `ExtractionRecordDetailPage.tsx`, `DocumentEntryRenderer.tsx`, internal API tests  
**Release note required**: Yes — must call out `field_values` removal

---

## Change Summary

| Field | Before | After |
|-------|--------|-------|
| `field_values` | `ExtractedFieldValue[]` | **Removed** |
| `fields` | — | Unified polymorphic list: top-level value fields + group entries |

---

## TypeScript Types (client)

```typescript
// Existing — unchanged
export interface ExtractedFieldValue {
  id: string;
  field_codename: string;
  field_name: string;
  field_type: string;
  display_value: string | null;
  raw_value: string | null;
  confidence: number | null;
  is_critical: boolean;
  display_order: number;
}

// New — group entry shapes (field_type discriminates)
export interface RepeatableGroupEntry {
  field_codename: string;
  field_name: string;
  field_type: 'repeatable_group';
  description: string;
  min_items: number;
  max_items: number;
  items: ExtractedFieldValue[][] | null;  // one inner array per extracted item
}

export interface SingleObjectGroupEntry {
  field_codename: string;
  field_name: string;
  field_type: 'single_object_group';
  description: string;
  optional: boolean;
  fields: ExtractedFieldValue[] | null;  // null when optional and not found
  not_found: boolean;
}

export type FieldsListEntry = ExtractedFieldValue | RepeatableGroupEntry | SingleObjectGroupEntry;

// Modified — field_values removed, unified fields list added
export interface ExtractionRecordDetail {
  id: string;
  document_type: { id: string; name: string };
  history_id: string;
  application_id: string;
  status: string;
  email_received_at: string;
  overall_confidence: number | null;
  needs_review_reason: string;
  created_at: string;
  email_subject: string;
  email_sender: string;
  email_message_id: string;
  extraction_source: string | null;
  classification_rationale: string;
  failure_stage: string | null;
  failure_message: string;
  failed_attachment_names: string[];
  trace_id: string;
  // NEW SHAPE — replaces field_values
  fields: FieldsListEntry[];
}
```

---

## Django Serializer Contract

```python
# ExtractionRecordDetailSerializer output shape
{
    # ... all existing meta fields unchanged ...
    "fields": [
        # top-level value fields (field_type != group variant)
        ExtractedFieldValueSerializer,
        # ...
        # repeatable group entry
        {
            "field_codename": str,
            "field_name": str,
            "field_type": "repeatable_group",
            "description": str,
            "min_items": int,
            "max_items": int,
            "items": [[ExtractedFieldValueSerializer]] | None,
        },
        # single object group entry
        {
            "field_codename": str,
            "field_name": str,
            "field_type": "single_object_group",
            "description": str,
            "optional": bool,
            "fields": [ExtractedFieldValueSerializer] | None,
            "not_found": bool,
        },
    ]
}
```

---

## Migration Checklist

Before deploy, verify all consumers updated:

- [ ] `ExtractionRecordDetailSerializer` — `field_values` removed, unified `fields` list added
- [ ] `ExtractionRecordDetail` TypeScript type updated
- [ ] `ExtractionRecordDetailPage.tsx` — renders unified `fields` list (value fields + group entries)
- [ ] `DocumentEntryRenderer.tsx` — renders unified `fields` list read-only
- [ ] All Django API tests referencing `field_values` updated
- [ ] Release notes drafted
