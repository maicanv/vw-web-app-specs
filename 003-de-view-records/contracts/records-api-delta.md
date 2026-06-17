# API Contract Delta: Extraction Records List

**Endpoint**: `GET /api/v1/document-entries/records/`

**Change type**: Additive — new query parameters only. Response shape unchanged.

---

## New Query Parameters

| Param | Type | Example | Notes |
|-------|------|---------|-------|
| `status` | string (comma-separated) | `extracted,needs_review` | Filter by ExtractionRecord processing status. Valid values: `extracted`, `needs_review`, `failed`, `sent`, `skipped`, `send_failed`, `rejected`. Multiple values = OR logic. |
| `delivery_status` | string (comma-separated) | `pending,warning` | Filter by aggregated delivery status. Valid values: `pending`, `warning`, `success`, `error`. Multiple values = OR logic. |
| `confidence_max` | float [0.0–1.0] | `0.8` | Return records with overall_confidence ≤ this value. |
| `date_from` | ISO 8601 date | `2026-01-01` | Filter `email_received_at` ≥ this date (inclusive, UTC). |
| `date_to` | ISO 8601 date | `2026-06-30` | Filter `email_received_at` ≤ this date (inclusive, UTC). |
| `search` | string | `acme invoice` | Case-insensitive match on `email_sender` OR `email_subject`. |
| `ordering` | string | `-email_received_at` | Sort order. Accepted values: `email_received_at` (asc), `-email_received_at` (desc). Default: `-email_received_at`. |

## Existing Parameters (unchanged)

| Param | Type | Notes |
|-------|------|-------|
| `document_type` | hashid string | Filter by document type (existing) |
| `history` | hashid string | Filter by history item (existing) |

## Validation

Filtering is implemented via `django-filter` (`ExtractionRecordFilterSet`) plus DRF `SearchFilter` / `OrderingFilter`. Validation is delegated to the filterset field types — no bespoke validation. Malformed input (e.g. a non-date in `date_from`) follows `django-filter` defaults; out-of-set choice values and reversed date ranges simply narrow the result set rather than returning a 400.

## Response Shape

**List endpoint** (`/records/`): unchanged — `ExtractionRecordListSerializer` output.

**Detail endpoint** (`/records/{id}/`): two new fields added to `ExtractionRecordDetailSerializer`:

```json
"email_body": "Please find attached the invoice for...",
"attachments": [
  {
    "filename": "invoice.pdf",
    "mime_type": "application/pdf",
    "failed": false
  },
  {
    "filename": "broken.xlsx",
    "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "failed": true
  }
]
```

- `email_body` is `History.input["content"]`; `null` when `history` is null or the email had no body.
- `attachments` carries metadata only — **no** `download_url`. Bytes are fetched on demand from the streaming-proxy endpoint below.
- `failed: true` entries correspond to filenames in `failed_attachment_names`.
- `attachments` returns `[]` when no attachments exist or `history` is null.

**Attachment download (new endpoint)**: `GET /records/{id}/attachments/{index}/download/` — same-origin streaming proxy. Streams the attachment bytes (`Content-Type: <mime_type>`, `Content-Disposition: inline`) read via `default_storage.open()` (authenticated GCS client — no signed URL generated; nothing GCS-hosted is returned to the client). `404` for an out-of-range index, missing/invalid `gcs_path`, or a failed attachment.
