# API Contract: Output Routes

**Feature**: `002-data-output-rest-api-mcp`
**Base path**: `/api/v1/document-entries/document-types/{document_type_id}/output-routes/`

> **TP deviation (DEV-12)**: TP §3.1 uses a flat URL (`/output-routes/?document_type={id}`). The implementation uses nested routing under `document-types/` for org-scoping clarity. See research.md DEV-12.

---

## Permissions

| Action | Required permissions |
|--------|---------------------|
| `list`, `retrieve` | `DOCUMENT_TYPES_EDIT` |
| `create`, `update`, `partial_update`, `destroy` | `DOCUMENT_TYPES_EDIT` **and** `API_INTEGRATIONS_CREATE_DELETE` |

> TP §3.1 referenced `ApiIntegrationAccessPolicy` with `API_INTEGRATIONS_EDIT`. Implementation uses a dedicated `OutputRouteAccessPolicy` — mutations require both document-type edit access and connector management access. See research.md D-011.

---

## List Output Routes

```
GET /api/v1/document-entries/document-types/{document_type_id}/output-routes/
```

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
    "headers": []
  }
}
```

`endpoint.connection_id` references an existing `ApiConnection` (must be in CONNECTED status).
`endpoint.key` must be a unique slug within the connection's provider.
`endpoint.headers` is a list of header objects (matching the existing `ApiEndpoint` headers format).
`endpoint.body_template` is omitted — set to `"{{content}}"` automatically by the system (TP §4.1).
AI-specific fields (`parameters`, `configuration`) are hidden from this form.

**Response 201**: Same shape as list item.

**Errors**:
- `400`: label missing, endpoint validation failed, connection not in CONNECTED status, endpoint key collision
- `403`: insufficient permissions (`DOCUMENT_TYPES_EDIT` + `API_INTEGRATIONS_CREATE_DELETE` required)
- `404`: document_type_id not found in organisation

---

## Retrieve Output Route

```
GET /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/
```

**Response 200**: Same shape as list item.

---

## Update Output Route

```
PATCH /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/
```

Partial update. `endpoint.connection_id` is locked after creation (omit it from PATCH requests).
All other fields are patchable: `label`, `delivery_mode`, `repeat_policy`, `enabled`, `value_transforms`, `endpoint.name`, `endpoint.key`, `endpoint.path`, `endpoint.method`, `endpoint.headers`.

SSRF re-check runs when endpoint fields change OR when `enabled` changes to `true` (DNS may have changed since creation).

**Response 200**: Updated route (list shape).

**Errors**:
- `400`: validation error, endpoint key collision, SSRF block on resolved URL
- `403`: insufficient permissions
- `404`: route not found in organisation

---

## Delete Output Route

```
DELETE /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/
```

Deletes the route and its owned `ApiEndpoint` (explicit cascade: route deleted first, then endpoint). `DeliveryAttempt` rows for this route are preserved (FK set to NULL).

**Response 204**: No content.

**Errors**:
- `403`: insufficient permissions
- `404`: route not found in organisation

---

## Payload Preview

```
GET /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/payload-preview/
Query params: record_id (optional, hashid)
```

> **TP deviation (DEV-11)**: TP §3.3 defines preview as `POST /output-routes/preview/` at collection level, accepting unsaved route config. v1 scopes this to saved routes only (detail endpoint, GET). Unsaved-route preview deferred to v2.

Returns the shaped payload for the route's current `value_transforms`.
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

**Errors**:
- `400`: record belongs to a different DocumentType
- `404`: route or record not found in organisation

---

## Endpoint Test

```
POST /api/v1/document-entries/document-types/{document_type_id}/output-routes/{route_id}/test/
```

Fires a live request to the route's endpoint with the current preview payload. No `DeliveryAttempt` is written. Reuses `BaseConnectionService.test_endpoint()` for header redaction and execution.

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
- `403`: insufficient permissions
- `404`: route not found in organisation
- `422`: connection not in CONNECTED status
