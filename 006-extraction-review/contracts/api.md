# API Contracts: Extraction Review

**Date**: 2026-07-07 (rewritten after source-story revision)

Base: `/api/v1/document-entries/` (existing router) unless noted. All endpoints org-scoped by the existing access policies. Non-additive changes (marked ⚠) are consumed only by the same-repo client and land in the same deploy.

## Existing endpoints reused / extended

| Endpoint | Role in this feature |
|----------|----------------------|
| `PATCH /extraction-records/{id}/status/` (`update_status`) | **Confirm** (`{"status": "extracted"}`), **Reject** (`{"status": "rejected"}`), and — with the new transitions — **Select for review / re-open** (`{"status": "needs_review"}`, appends a "Selected for review by X" reason). Still enforces `ALLOWED_STATUS_TRANSITIONS`, triggers auto-delivery on confirm, and returns 409 if the record is locked by another user. |
| `POST /route-deliveries/{id}/send/` | Manual send / retry / re-send after correction. Now refuses (`400/409`) when the route's `resend_policy` is `block` and the delivery already succeeded (FR-018/028). |
| `GET /extraction-records/` + filters | Records list; gains `review_queue=true` (below). |
| `GET/PATCH /document-types/{id}/` | Gains `review_config` (full ReviewConfig object, see data-model). Partial update replaces the whole config object. |
| `POST/PATCH /output-routes/` | Route payload changes: ⚠ `repeat_policy` → `resend_policy` (`allow`\|`block`); ⚠ `delivery_mode` removed. |

## New / extended endpoints

### 1. Correct values — `POST /extraction-records/{id}/correct/`

Bulk-corrects field values on any record, any status (FR-014..016).

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

- `original_value` = immutable `raw_value` (before → after, FR-015). Confidence kept, edited badge in UI (D4).
- Errors: `400` type-validation failure per field; `409` record locked by another user; `403` per `can_review` / existing access policy.
- Never changes `status` (D4) — Confirm stays explicit.

### 2. Platform audit — `GET /api/v1/audit/`

New thin `apps/audit` read-only endpoint over auditlog (FR-019..021). Org-scoped per subsystem (research R6). Query params: `subsystem` (required for v1; `extraction_records` is the only value shipped), `date_from`, `date_to`, `actor`, pagination.

Response `200`:
```json
{
  "results": [
    {
      "actor": {"id": 7, "name": "Jane Doe"},
      "timestamp": "2026-07-07T14:03:00Z",
      "subsystem": "extraction_records",
      "action": "correct",
      "object": {"type": "extraction_record", "id": "aB3xK", "label": "Invoice INV-2041"},
      "changes": [{"field": "IBAN", "before": "NL91ABNA0417164", "after": "NL91ABNA0417164300"}],
      "originally_flagged": true,
      "flag_reason": "Field 'IBAN' 42% below threshold 80%"
    }
  ]
}
```

`action` ∈ `confirm | correct | reject | select_for_review | resend | status_change`. Read-only, paginated. Permission: existing document-entry view access for the `extraction_records` subsystem; future subsystems bring their own guard.

### 3. Reviewer assignment — `GET/PUT /document-types/{id}/reviewers/`

Requires `document_types:edit`. PUT replaces the set; grants/revokes the Reviewer role assignment as a side effect (see data-model).

```json
// GET / PUT response
{"reviewers": [{"user_id": 7, "name": "Jane Doe"}, {"user_id": 9, "name": "Bob"}]}
// PUT request
{"user_ids": [7, 9]}
```

- `400` if a user is not an org/workspace member.

### 4. Review queue — existing list endpoint, new filter + count

- `GET /extraction-records/?review_queue=true` → needs_review records per `records_for_reviewer(user)` (assigned document types; zero-assignment fallback for workspace admins/managers).
- `GET /extraction-records/review-queue-count/` → `{"count": 12}` for the sidebar badge.
- ⚠ The client's records-list status filter drops the `needs_review` option (FR-012) — server keeps accepting it (additive server, UI-only removal).

### 5. Record lock — `POST /extraction-records/{id}/claim/`, `DELETE /extraction-records/{id}/claim/`

Mirrors the locks-app view pattern; hard lock, 5-minute TTL (D7), refreshed on save activity.

- `POST` → `200 {"holder": {...}, "expires_at": "..."}` or `409` with current holder if held by someone else.
- `DELETE` releases own lock; auto-expiry after TTL.
- Record detail response gains `lock` (`null` when free) so the UI renders "being reviewed by X".
- `correct/` and `update_status` return `409` when another user holds an unexpired lock.

### 6. Threshold surfacing (detail serializer additions, FR-022)

Record detail response gains: `review_config` echo (`applied_threshold`s at read time) and per-value `below_threshold: bool`, so the UI highlights without duplicating the rules client-side.

## Delivery behaviour contract (no new endpoint)

- Auto-delivery unchanged for unflagged records; needs_review records are held (existing `DELIVERABLE_STATUSES` gating, FR-008/009).
- A delivered record that passed through review (confirmed or corrected) uses the route's normal delivery method and UID handling, unchanged — no separate post-review method/UID was added (dropped from scope, too much work for now).
