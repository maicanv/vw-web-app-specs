# API Contract: Document Type Templates

**Feature**: 004-document-type-templates | **Date**: 2026-06-24

Read-only API consumed by the create-flow template picker. Management is Django-admin only (no write endpoints in v1). All IDs are hashid-encoded. Mounted under the existing `document-entries` namespace.

| Method | Path | Permission | Purpose |
|--------|------|------------|---------|
| `GET` | `/api/v1/document-entries/document-type-templates/` | authenticated + `document_types:edit` | List active templates visible to the user's org |
| `GET` | `/api/v1/document-entries/document-type-templates/{id}/` | authenticated + `document_types:edit` | Retrieve one template with full fields for pre-fill |

**Visibility / scoping** (both endpoints): `is_active=True AND (organisation IS NULL OR organisation = request.organisation)`. Sorted by `name` ascending. No pagination (return all).

---

## List — `GET /document-type-templates/`

Lightweight payload for the picker grid. Omits the full `fields` tree; exposes `field_count` for the card.

**200 Response**:
```json
[
  {
    "id": "a1B2c3",
    "name": "Invoice",
    "blurb": "Supplier invoices with line items and totals",
    "icon": "invoice",
    "field_count": 10
  },
  {
    "id": "d4E5f6",
    "name": "Purchase Order",
    "blurb": "Vendor POs with ordered items and delivery info",
    "icon": "purchase_order",
    "field_count": 9
  }
]
```

Empty list (no templates, or all deactivated) → `[]`. The picker still renders "Start from scratch".

---

## Retrieve — `GET /document-type-templates/{id}/`

Full payload used to pre-fill the wizard after selection. `description`, `instructions`, and `fields` map directly onto `DocumentTypeFormValues` (the **name is not pre-filled**).

**200 Response**:
```json
{
  "id": "a1B2c3",
  "name": "Invoice",
  "blurb": "Supplier invoices with line items and totals",
  "icon": "invoice",
  "description": "Extracts structured data from supplier invoices including line items and totals.",
  "instructions": "Extract the invoice number, dates, vendor, and all line items.",
  "field_count": 10,
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
      "codename": "invoice_date",
      "name": "Invoice Date",
      "config": { "type": "date", "date_formats": ["%Y-%m-%d"] },
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
```

The `fields` shape is identical to the `DocumentTypeField` tree the create wizard already produces, so the frontend maps it straight into `DocumentTypeFieldFormValues[]`.

---

## Errors

| Status | When |
|--------|------|
| `401` | Unauthenticated |
| `403` | Authenticated but lacks `document_types:edit` |
| `404` | Template id not found, inactive, or not visible to the user's org (no leakage of existence across orgs) |

A `404` for inactive/cross-org templates (rather than `403`) avoids confirming that a hidden template exists.
