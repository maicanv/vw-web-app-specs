# API Contract: Output Routes

**Feature**: `002-data-output-rest-api-mcp`
**Base path**: `/api/v1/document-entries/document-types/{document_type_id}/output-routes/`

---

## List Output Routes

```
GET /api/v1/document-entries/document-types/{document_type_id}/output-routes/
```

**Auth**: Organisation-scoped. Requires `manage_integrations` permission.

**Response 200**:
```json
[
  {
    "id": "abc123",
    "label": "ERP Inbound",
    "endpoint_url": "https://erp.example.com/inbound",
    "endpoint_method": "POST",
    "delivery_mode": "auto",
    "repeat_policy": "allow_resend",
    "enabled": true,
    "value_transforms": {
      "delivery_date": {
        "date_format": "DD/MM/YYYY",
        "strip_non_alphanumeric": false,
        "missing_value_behaviour": "null"
      }
    },
    "created_at": "2026-05-26T10:00:00Z"
  }
]
```

---

## Create Output Route

```
POST /api/v1/document-entries/document-types/{document_type_id}/output-routes/
```

**Request body**:
```json
{
  "label": "ERP Inbound",
  "delivery_mode": "auto",
  "repeat_policy": "prevent_duplicates",
  "enabled": true,
  "value_transforms": {},
  "endpoint": {
    "connection_id": "conn_xyz",
    "name": "ERP POST endpoint",
    "key": "erp-inbound",
    "path": "/inbound",
    "method": "POST",
    "headers": {}
  }
}
```

`endpoint.connection_id` references an existing `ApiConnection` (must be in CONNECTED status).
`endpoint.key` must be a unique slug within the organisation.
`endpoint.body_template` is omitted — set automatically by the system.
AI-specific fields (`parameters`, `configuration`) are hidden from this form.

**Response 201**: Same shape as list item.

**Errors**:
- `400`: label missing, endpoint validation failed, connection not in CONNECTED status
- `404`: document_type_id not found in organisation

---

## Retrieve Output Route

```
GET /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/
```

**Response 200**: Same as list item shape plus `value_transforms` (full detail).

---

## Update Output Route

```
PATCH /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/
```

Partial update. All fields except `endpoint.connection_id` (connection is locked after creation).
Endpoint URL/method/headers can be changed. `label`, `delivery_mode`, `repeat_policy`, `enabled`, `value_transforms` all patchable.

**Response 200**: Updated route.

---

## Delete Output Route

```
DELETE /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/
```

Deletes the route and its owned `ApiEndpoint`. `DeliveryAttempt` rows for this route are preserved (FK set to NULL).

**Response 204**: No content.

---

## Payload Preview

```
GET /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/payload-preview/
Query params: record_id (optional, hashid)
```

Returns the sample payload for the route's current transforms.
- Without `record_id`: uses mock/type-appropriate placeholder values; repeatable groups show 2 example items.
- With `record_id`: uses real extracted values from that record (must belong to the same DocumentType + organisation).

**Response 200**:
```json
{
  "format": "json",
  "payload": {
    "general": {
      "date": "2026-05-26",
      "time": "10:00:00",
      "record_id": "abc123",
      "document_type": "Mercitalia Pickup Codes",
      "trace_id": "ef8b9a64-..."
    },
    "email": {
      "id": "ef8b9a64-...",
      "from": "sender@example.com",
      "to": "recipient@example.com",
      "subject": "Example Subject",
      "received_at": "2026-05-26T10:00:00+00:00"
    },
    "data": {
      "Provider": "EXAMPLE PROVIDER",
      "pickup_codes": [
        { "uti_code": "RCLU1279128", "reference": "1513445" }
      ]
    }
  }
}
```

---

## Endpoint Test

```
POST /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/test/
```

Per TP §3.3 — fires a live request to the route's endpoint with the current preview payload (no `DeliveryAttempt` written). Reuses `BaseConnectionService.test_endpoint()` for header redaction and execution.

**Request body**: `{}` (optional `record_id` for real-data preview; omitted = mock data)

**Response 200**:
```json
{
  "status_code": 200,
  "response_body": "truncated response snippet (≤ 500 chars)",
  "timestamp": "2026-05-26T10:00:00Z"
}
```

**Notes**:
- Request headers (including secrets) are **never** in the response.
- Failures (timeout, SSRF block, non-2xx) return `200` with the failure's `status_code` and error snippet — this endpoint reports what happened, it does not error out on a failed test.

**Errors**:
- `403`: Permission denied
- `404`: Route not found in organisation
- `422`: Connection not in CONNECTED status
