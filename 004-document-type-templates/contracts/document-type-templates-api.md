# API Contract: Document Type Templates

**Feature**: 004-document-type-templates | **Date**: 2026-06-24

Read-only API consumed by the create-flow template picker. Management is Django-admin only (no write endpoints in v1). All IDs are hashid-encoded. Mounted under the existing `document-entries` namespace.

| Method | Path | Permission | Purpose |
|--------|------|------------|---------|
| `GET` | `/api/v1/document-entries/document-type-templates/` | authenticated + `document_types:edit` | List active templates visible to the user's org, **each with full `payload`** |
| `GET` | `/api/v1/document-entries/document-type-templates/{id}/` | authenticated + `document_types:edit` | Retrieve one template (same shape); deep-link / refresh only |

**Visibility / scoping** (both endpoints): `is_active=True AND (organisation IS NULL OR organisation = request.organisation)`. Sorted by `name` ascending. No pagination (return all).

**One serializer, no list/detail split** (proposal §3/§5). The set is tiny (~20 templates, each ≤50 fields, depth-2), so the list returns the full `payload` inline — no payload-size reason to trim it. A single `DocumentTypeTemplateSerializer` serves both endpoints; `field_count` is a `SerializerMethodField` derived from `payload.fields`. Because the list already carries `payload`, the picker needs **no per-card detail fetch** — the retrieve endpoint exists only for the deep-link / page-refresh case on the create page.

---

## List — `GET /document-type-templates/`

Returns every visible template with its full `payload`, so the picker can render both the card and the side-panel preview without a second request. Each item carries card metadata (`id`, `name`, `blurb`, `icon`), `field_count`, and the complete `payload` (`description`, `instructions`, `fields`).

**200 Response**:
```json
[
  {
    "id": "a1B2c3",
    "name": "Invoice",
    "blurb": "Supplier invoices with line items and totals",
    "icon": "invoice",
    "field_count": 10,
    "payload": {
      "description": "Extracts structured data from supplier invoices including line items and totals.",
      "instructions": "Extract the invoice number, dates, vendor, and all line items.",
      "fields": [
        {
          "codename": "invoice_number",
          "name": "Invoice Number",
          "config": { "type": "string" },
          "description": "",
          "is_critical": true,
          "confidence_threshold": null,
          "children": []
        },
        {
          "codename": "line_items",
          "name": "Line Items",
          "config": { "type": "repeatable_group", "description": "Invoice line items", "min_items": 0, "max_items": 200 },
          "description": "",
          "is_critical": false,
          "confidence_threshold": null,
          "children": [
            { "codename": "description", "name": "Description", "config": { "type": "string" }, "description": "", "is_critical": false, "confidence_threshold": null },
            { "codename": "amount", "name": "Amount", "config": { "type": "currency" }, "description": "", "is_critical": false, "confidence_threshold": null }
          ]
        }
      ]
    }
  }
]
```

Empty list (no templates, or all deactivated) → `[]`. The picker still renders "Start from scratch".

---

## Retrieve — `GET /document-type-templates/{id}/`

Identical object shape to a single list element (same serializer). Used only for the deep-link / page-refresh case where the picker's in-memory list isn't available. `payload.description`, `payload.instructions`, and `payload.fields` map directly onto `DocumentTypeFormValues` (the **name is not pre-filled**).

**200 Response**: same object shape as a list element above.

The `payload.fields` shape is identical to the `DocumentTypeField` tree the create wizard already produces, so the frontend maps it straight into `DocumentTypeFieldFormValues[]`.

---

## Errors

| Status | When |
|--------|------|
| `401` | Unauthenticated |
| `403` | Authenticated but lacks `document_types:edit` |
| `404` | Template id not found, inactive, or not visible to the user's org (no leakage of existence across orgs) |

A `404` for inactive/cross-org templates (rather than `403`) avoids confirming that a hidden template exists.
