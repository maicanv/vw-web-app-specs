# API Contract: Record Detail — Delivery Section

**Feature**: `002-data-output-rest-api-mcp`

---

## Record Detail — Delivery Extension

**Endpoint**: `GET /api/v1/document-entries/records/{record_id}/`

The existing response gains a `delivery` array appended to the top-level response object. No existing fields are removed or renamed.

**New `delivery` field** (array of route delivery status objects):

```json
{
  "id": "rec_abc",
  "status": "success",
  "fields": [...],
  "delivery": [
    {
      "output_route_id": "route_xyz",
      "route_label": "ERP Inbound",
      "endpoint_url": "https://erp.example.com/inbound",
      "delivery_mode": "auto",
      "repeat_policy": "allow_resend",
      "enabled": true,
      "delivery_status": "sent",
      "can_confirm": false,
      "can_retry": false,
      "can_resend": true,
      "attempts": [
        {
          "id": 1,
          "created_at": "2026-05-26T10:05:00Z",
          "status": "success",
          "http_status_code": 200,
          "error_message": "",
          "payload_hash": "a3f9d2e1...",
          "attempt_number": 1
        }
      ]
    },
    {
      "output_route_id": "route_deleted",
      "route_label": null,
      "endpoint_url": null,
      "delivery_mode": null,
      "repeat_policy": null,
      "enabled": null,
      "delivery_status": "send_failed",
      "can_confirm": false,
      "can_retry": true,
      "can_resend": false,
      "attempts": [
        {
          "id": 2,
          "created_at": "2026-05-26T10:06:00Z",
          "status": "failed",
          "http_status_code": 503,
          "error_message": "Service Unavailable",
          "payload_hash": "b4e8f3c2...",
          "attempt_number": 1
        }
      ]
    }
  ]
}
```

**`delivery_status` values**: `"pending_approval"` | `"pending"` | `"sent"` | `"send_failed"` | `"not_configured"`

**Notes**:
- When `output_route` is NULL (route deleted), route metadata fields are null but `attempts` are still returned from `DeliveryAttempt.route_label_snapshot` / `endpoint_url_snapshot`.
- `can_confirm`, `can_retry`, `can_resend` are always `false` in history detail view context.
- Secret headers do NOT appear in `attempts` or in the endpoint URL.

---

## Deliver Action

```
POST /api/v1/document-entries/records/{record_id}/deliver/
```

Triggers delivery for a specific route for this record. Handles Confirm & Send, Retry, and Re-send.

**Request body**:
```json
{
  "output_route_id": "route_xyz"
}
```

**Business logic** (service layer):
- `output_route_id` must belong to the record's DocumentType
- Route must be enabled
- For `requires_approval` routes: must have `delivery_status == "pending_approval"` or `"send_failed"` or (`"sent"` + `allow_resend`)
- For `prevent_duplicates` routes: re-send is blocked if latest attempt is `"success"` — returns `409 Conflict`

**Response 200**:
```json
{
  "output_route_id": "route_xyz",
  "delivery_status": "sent",
  "attempt": {
    "id": 2,
    "created_at": "2026-05-26T10:10:00Z",
    "status": "success",
    "http_status_code": 200,
    "error_message": "",
    "payload_hash": "c5d7a4b3...",
    "attempt_number": 2
  }
}
```

**Response 202**: Delivery in progress (async — unlikely in v1 but reserved).

**Errors**:
- `400`: Invalid `output_route_id` or route not associated with this record's document type
- `403`: Permission denied (no `manage_integrations`)
- `409`: Duplicate blocked by `prevent_duplicates` policy
- `422`: Route disabled; connection not in CONNECTED status

---

## History Detail View

**Endpoint**: same `GET /api/v1/document-entries/records/{record_id}/` but consumed from the history detail context.

The serializer sets `is_history=True` context flag (or history detail uses a separate serializer subclass). Result: `can_confirm`, `can_retry`, `can_resend` are all `false`. No other changes to the `delivery` array structure.

**Migration checklist** (release gate):
- [ ] `delivery` field present in record detail response for all existing records (returns empty array if no output routes configured — not null)
- [ ] No existing `delivery` key collision in the response (confirm no other serializer adds this key)
- [ ] History detail view correctly shows read-only delivery section
- [ ] Secret headers confirmed absent from all delivery log responses
