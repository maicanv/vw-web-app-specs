# API Contracts: Extraction Review

Base: `/api/v1/document-entries/` (existing router). All endpoints org-scoped by the existing access policy. Variant parts tagged with their OQ.

## Existing endpoints reused as-is

| Endpoint | Role in this feature |
|----------|----------------------|
| `POST /extraction-records/{id}/update_status/` | **Confirm** (`{"status": "extracted"}`) and **Reject** (`{"status": "rejected"}`). Already enforces `ALLOWED_STATUS_TRANSITIONS` and triggers auto-delivery on confirm. |
| `POST /route-deliveries/{id}/send/` | Manual send / retry / **re-send after correction** (payload rebuilt from current values). |
| `GET /extraction-records/` + filters | Records list; gains `my_queue=true` filter (below). |
| `PATCH /document-types/{id}/` | Threshold configuration (fields already exist). |

## New / extended endpoints

### 1. Correct values — `POST /extraction-records/{id}/correct/`

Bulk-corrects field values on any record, any status (FR-008..FR-010).

Request:
```json
{
  "values": [
    {"field_value_id": 123, "value": "NL91ABNA0417164300"},
    {"field_value_id": 456, "value": "2026-07-01"}
  ]
}
```

Response `200`: record detail (existing serializer) with per-value additions:
```json
{
  "field_values": [
    {
      "id": 123,
      "display_value": "NL91ABNA0417164300",
      "original_value": "NL91ABNA0417164",
      "is_manually_edited": true,
      "confidence": 42
    }
  ]
}
```

- `original_value` = immutable `raw_value` (before → after, FR-009).
- `confidence` display semantics per **OQ-3** (default: original score kept; UI shows edited badge).
- Errors: `400` type-validation failure per field; `409` record locked by another user (**OQ-8** default hard lock); `403` per `can_review` / existing access policy.
- Never changes `status` (**OQ-3** default; if auto-clear wins, this endpoint calls the existing transition internally — contract unchanged apart from `status` in the response).

### 2. Audit trail — `GET /extraction-records/{id}/audit/`

User-facing history (FR-012/FR-013), sourced from auditlog entries.

Response `200`:
```json
{
  "results": [
    {
      "actor": {"id": 7, "name": "Jane Doe"},
      "timestamp": "2026-07-06T14:03:00Z",
      "action": "correct",          
      "changes": [{"field": "IBAN", "before": "NL91ABNA0417164", "after": "NL91ABNA0417164300"}],
      "originally_flagged": true,
      "flag_reason": "Field 'IBAN' 42% below threshold 80%"
    }
  ]
}
```

`action` ∈ `confirm | correct | reject | resend | status_change`. Read-only, paginated.

### 3. Reviewer assignment — `GET/PUT /document-types/{id}/reviewers/` (**OQ-5 seam**)

Manage-permission required (**OQ-6** default). PUT replaces the set (simplest for a config UI).

```json
// GET / PUT response
{"reviewers": [{"user_id": 7, "name": "Jane Doe"}, {"user_id": 9, "name": "Bob"}]}
// PUT request
{"user_ids": [7, 9]}
```

- `400` if a user is not an org/workspace member.
- OQ-5 option B/C/D relocates this sub-resource (data source / workspace) — payload shape survives all options.
- OQ-10 "yes" adds `POST /document-types/reviewers/bulk/` — additive.

### 4. Reviewer queue — existing list endpoint, new filter

`GET /extraction-records/?my_queue=true` → needs_review records per `records_for_reviewer(user)` (assigned document types + zero-assignment fallback for admins/managers). Plus `GET /extraction-records/queue-count/` (or a count field on the list response) for the badge. Predicate is the **OQ-5** seam; the contract is option-independent.

### 5. Record lock — `POST /extraction-records/{id}/claim/`, `DELETE /extraction-records/{id}/claim/`

Mirrors the existing locks-app view pattern (**OQ-8**).

- `POST` → `200 {"holder": {...}, "expires_at": "..."}` or `409` with current holder if held by someone else.
- `DELETE` releases own lock; auto-expiry after TTL (default 15 min).
- Record detail response gains a `lock` object (`null` when free) so the UI can render "being reviewed by X".

### 6. Threshold surfacing (detail serializer additions, FR-014)

Record detail response gains: `applied_threshold` (the document-type value at read time) and per-value `below_threshold: bool` so the UI highlights without duplicating the rule client-side.

## Notifications (internal contract, **OQ-7**)

`notify_reviewers(record)` fires when a record enters needs_review: in-app = queue count (no new endpoint); email via existing async task path, throttled per reviewer (default option C). No public API surface.
