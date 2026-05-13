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
| `header` | — | `ExtractedFieldValue[]` (header-level fields only) |
| `groups` | — | `Record<string, GroupResult>` keyed by group codename |

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

// New
export type GroupKind = 'repeatable' | 'single_object';

export interface GroupResult {
  name: string;
  kind: GroupKind;
  optional: boolean;
  min_items: number | null;   // repeatable only
  max_items: number | null;   // repeatable only
  items: ExtractedFieldValue[][] | null;   // repeatable: array of items, each item is array of fields
  fields: ExtractedFieldValue[] | null;    // single_object: flat array; null if not_found
  not_found: boolean;                      // true when optional single_object absent
}

// Modified — field_values removed
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
  header: ExtractedFieldValue[];
  groups: Record<string, GroupResult>;
}
```

---

## Django Serializer Contract

```python
# ExtractionRecordDetailSerializer output shape
{
    # ... all existing meta fields unchanged ...
    "header": [ExtractedFieldValueSerializer],          # group=NULL
    "groups": {
        "<codename>": {
            "name": str,
            "kind": "repeatable" | "single_object",
            "optional": bool,
            "min_items": int | None,
            "max_items": int | None,
            "items": [[ExtractedFieldValueSerializer]] | None,
            "fields": [ExtractedFieldValueSerializer] | None,
            "not_found": bool,
        }
    }
}
```

---

## Group Management Endpoints (new)

**Nested under document types:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/document-entries/document-types/{id}/groups/` | List groups for a document type |
| `POST` | `/document-entries/document-types/{id}/groups/` | Create a group |
| `PATCH` | `/document-entries/document-types/{id}/groups/{group_id}/` | Update group (name, description, limits; codename immutable if active) |
| `DELETE` | `/document-entries/document-types/{id}/groups/{group_id}/` | Delete group (blocked if document type is active) |
| `POST` | `/document-entries/document-types/{id}/groups/{group_id}/fields/` | Move a field into this group |
| `DELETE` | `/document-entries/document-types/{id}/groups/{group_id}/fields/{field_id}/` | Move field back to header level (blocked if document type is active) |

**Group serializer shape:**
```json
{
  "id": "uuid",
  "codename": "invoice_lines",
  "name": "Invoice Lines",
  "description": "...",
  "kind": "repeatable",
  "optional": false,
  "min_items": 0,
  "max_items": 200,
  "display_order": 0,
  "fields": [{ "id": "...", "codename": "...", "name": "...", "display_order": 0 }]
}
```

---

## Migration Checklist

Before deploy, verify all consumers updated:

- [ ] `ExtractionRecordDetailSerializer` — `field_values` removed, `header` + `groups` added
- [ ] `ExtractionRecordDetail` TypeScript type updated
- [ ] `ExtractionRecordDetailPage.tsx` — renders `header` + `groups` instead of `field_values`
- [ ] `DocumentEntryRenderer.tsx` — renders `header` + `groups` read-only
- [ ] All Django API tests referencing `field_values` updated
- [ ] Release notes drafted
