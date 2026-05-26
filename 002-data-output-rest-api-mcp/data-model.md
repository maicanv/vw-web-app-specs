# Data Model: Data Output via REST API

**Feature**: `002-data-output-rest-api-mcp`
**Date**: 2026-05-26

---

## New Models

### `OutputRoute`

**App**: `document_entries`
**Table**: `document_entries_outputroute` (Django default)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | AutoField | PK | |
| `document_type` | FK → DocumentType | CASCADE, `related_name="output_routes"` | |
| `organisation` | FK → Organisation | CASCADE | Denorm for queryset filtering |
| `api_endpoint` | OneToOneField → ApiEndpoint | PROTECT, `related_name="output_route"` | Route owns endpoint; viewset `perform_destroy` deletes endpoint first |
| `label` | CharField(255) | not null | Display name |
| `delivery_mode` | CharField(50) | choices, not null | `"auto"` \| `"requires_approval"` |
| `repeat_policy` | CharField(50) | choices, not null | `"allow_resend"` \| `"prevent_duplicates"` |
| `value_transforms` | JSONField | default=dict | Per-codename transform config (see schema below) |
| `enabled` | BooleanField | default=True | Auto delivery skipped when False |
| `created_at` | DateTimeField | auto_now_add | |
| `updated_at` | DateTimeField | auto_now | |

**`value_transforms` JSON schema**:
```json
{
  "<field_codename>": {
    "date_format": "ISO 8601",
    "strip_non_alphanumeric": false,
    "missing_value_behaviour": "null"
  }
}
```
- `date_format`: string; default `"ISO 8601"` (emits `YYYY-MM-DD`); other values are arbitrary format strings passed to `strftime`-style formatting
- `strip_non_alphanumeric`: boolean; default `false`
- `missing_value_behaviour`: `"null"` (default) | `"empty_string"`

**Delivery mode choices** (also as a `TextChoices` enum in `models.py`):
```python
class DeliveryMode(models.TextChoices):
    AUTO = "auto", "Auto"
    REQUIRES_APPROVAL = "requires_approval", "Requires Approval"
```

**Repeat policy choices**:
```python
class RepeatPolicy(models.TextChoices):
    ALLOW_RESEND = "allow_resend", "Allow Re-send"
    PREVENT_DUPLICATES = "prevent_duplicates", "Prevent Duplicates"
```

**`__str__`**: `f"{self.label} ({self.document_type_id})"`

**Indexes**: `(document_type, enabled)` composite — used by delivery trigger query.

---

### `DeliveryAttempt`

**App**: `document_entries`
**Table**: `document_entries_deliveryattempt`

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | AutoField | PK | |
| `extraction_record` | FK → ExtractionRecord | CASCADE, `related_name="delivery_attempts"` | |
| `output_route` | FK → OutputRoute | SET_NULL, null=True, `related_name="delivery_attempts"` | Preserved if route is deleted |
| `route_label_snapshot` | CharField(255) | not null | Route label at delivery time |
| `endpoint_url_snapshot` | URLField(2000) | not null | Endpoint URL at delivery time |
| `status` | CharField(50) | choices, not null | `"success"` \| `"failed"` |
| `http_status_code` | IntegerField | null=True | Null for pre-request failures (SSRF block, payload too large) |
| `error_message` | TextField | blank=True | |
| `payload_hash` | CharField(64) | not null | SHA-256 hex digest (sorted-keys JSON, UTF-8) |
| `attempt_number` | PositiveIntegerField | default=1 | Monotonic per (record, route) pair |
| `created_at` | DateTimeField | auto_now_add | |

**Delivery status choices**:
```python
class DeliveryAttemptStatus(models.TextChoices):
    SUCCESS = "success", "Success"
    FAILED = "failed", "Failed"
```

**`__str__`**: `f"Attempt {self.attempt_number} for record {self.extraction_record_id} → {self.status}"`

**Indexes**:
- `(extraction_record, output_route)` — used by duplicate-prevention check and log display
- `(extraction_record, output_route, created_at DESC)` — used by latest-attempt query

---

## Migration

**File**: `backend/django/apps/document_entries/migrations/000N_add_output_routes.py`

Operations (in order):
1. `CreateModel(OutputRoute)` — including all fields and indexes
2. `CreateModel(DeliveryAttempt)` — including all fields and indexes

Both tables are new; no existing columns are modified. Backward-safe (no data loss on rollback — drop tables).

---

## Existing Models — No Changes

| Model | Why untouched |
|-------|--------------|
| `DocumentType` | Reverse `output_routes` relation is added by `OutputRoute.document_type` FK — no migration on this side |
| `ExtractionRecord` | Reverse `delivery_attempts` relation added automatically — no migration |
| `ApiEndpoint` | `output_route` reverse accessor added — no migration; `organisation` field already present (denorm) |

---

## TP VWE-1496 Constraints on PayloadBuilder

Copied from TP §2.3 for implementor reference — these constraints govern the `data` section assembly:

| `parent_field` | `group_item_index` | Placement in `data` |
|---|---|---|
| NULL | NULL | `data[codename] = value` |
| `<group field>` (single_object) | NULL | `data[group_codename][codename] = value` |
| `<group field>` (repeatable) | 0, 1, 2… | `data[group_codename][idx][codename] = value` |

**Group fields themselves have no `ExtractedFieldValue` rows** — they are structural. The PayloadBuilder iterates the `DocumentTypeField` tree; EFV rows are look-ups, not the iteration source.

**Absent optional single-object group** (TP §3.2): when the group is optional and no child EFV rows exist, the API returns `fields: null, not_found: true`. For the output payload, the group key is **omitted** from `data` entirely (not rendered as `null` or `{}`).

---

## Delivery Status Derivation

`delivery_status` on the API response is computed per route per record. It is NOT stored in any model.

```
given route.enabled == False                               → "not_configured"
given route.delivery_mode == "requires_approval"
  and no DeliveryAttempt for (record, route)              → "pending_approval"
given latest DeliveryAttempt.status == "success"           → "sent"
given latest DeliveryAttempt.status == "failed"            → "send_failed"
given route.delivery_mode == "auto"
  and no DeliveryAttempt for (record, route)              → "pending"
```

`can_confirm`: `delivery_status == "pending_approval"` and `not readonly`
`can_retry`: `delivery_status == "send_failed"` and `not readonly`
`can_resend`: `delivery_status == "sent"` and `route.repeat_policy == "allow_resend"` and `not readonly`

---

## `common/utils/ssrf_guard.py`

Not a model — a utility. Documented here for completeness.

```python
class SSRFBlockedError(Exception):
    pass

def check_url(url: str) -> None:
    """Raises SSRFBlockedError if url resolves to a blocked address."""
```

Blocked ranges (evaluated after DNS resolution):
- RFC 1918 private: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- Loopback: 127.0.0.0/8, ::1
- Link-local: 169.254.0.0/16, fe80::/10
- AWS/GCP metadata: 169.254.169.254
- Non-http(s) schemes

Uses `socket.getaddrinfo` to resolve the hostname before checking ranges.
