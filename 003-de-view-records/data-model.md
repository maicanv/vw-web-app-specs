# Data Model: Document Entry — View Records

**Branch**: `003-de-view-records` | **Date**: 2026-06-16

> This feature is read-only. No new models and no model field additions are anticipated. The section below documents the existing models as-built, pending Phase 0 spike confirmation.

---

## Existing Models (as-built)

### ExtractionRecord

`backend/django/apps/document_entries/models.py`

| Field | Type | Notes |
|-------|------|-------|
| `organisation` | FK → Organisation | CASCADE — org-scoping anchor |
| `application` | FK → Application | CASCADE |
| `document_type` | FK → DocumentType | SET_NULL — record survives type deletion |
| `history` | FK → ApplicationHistory | Link to the triggering email event |
| `status` | CharField (TextChoices) | `extracted`, `needs_review`, `sent`, `send_failed`, `rejected`, `skipped`, `failed` |
| `email_received_at` | DateTimeField | RFC 5321 received timestamp |
| `email_subject` | CharField(512) | |
| `email_sender` | CharField(320) | Raw email address |
| `overall_confidence` | FloatField (nullable) | Stored at extraction time — confirmed by Spike 1; supports DB-level `lte` filtering |
| `needs_review_reason` | TextField (nullable) | Human-readable reason why review is needed |
| `extraction_source` | CharField (TextChoices) | `body_only`, `attachments_only`, `both` |
| `failed_attachment_names` | JSONField | list[str] of attachment names that failed extraction |
| `failure_stage` | CharField (TextChoices, nullable) | `classification`, `extraction`, `parsing` |
| `failure_message` | TextField (nullable) | Sanitized error description |
| `classification_rationale` | TextField (nullable) | LLM classification explanation |
| `metadata` | JSONField | Stores `trace_id` and other runtime metadata |

**Indexes** (existing): `(organisation, created_at)`, `(organisation, status)`, `(application, status)`

> **Spike 1 resolved**: `overall_confidence` is a stored `FloatField(null=True)`. No migration required.

---

### ExtractedFieldValue

| Field | Type | Notes |
|-------|------|-------|
| `record` | FK → ExtractionRecord | CASCADE |
| `document_type_field` | FK → DocumentTypeField | SET_NULL (snapshot at extraction time) |
| `field_codename` | CharField | Snapshotted at extraction time |
| `field_name` | CharField | Snapshotted at extraction time |
| `field_type` | CharField | Snapshotted at extraction time |
| `raw_value` | TextField | Raw extracted string |
| `display_value` | TextField | Formatted/standardized value |
| `confidence` | FloatField | Per-field confidence [0.0–1.0] |
| `is_critical` | BooleanField | Snapshotted at extraction time |
| `is_manually_edited` | BooleanField | Reserved for future edit feature |
| `group_item_index` | IntegerField (nullable) | For repeatable group rows |
| `display_order` | IntegerField | Ordering within parent |

---

### DocumentType (reference — no changes)

Relevant fields for this feature:
- `global_confidence_threshold` — `FloatField` used as the fallback threshold for highlighting low-confidence values
- `name` — displayed in list and detail views

### DocumentTypeField (reference — no changes)

Relevant fields for this feature:
- `confidence_threshold` — per-field override (nullable); takes precedence over `global_confidence_threshold` for highlighting

---

## State Transitions (ExtractionRecord.status)

```
skipped          ← no matching document type found
failed           ← extraction error
extracted        → needs_review (low confidence / missing critical)
                 → sent (auto-delivery or manual confirm)
needs_review     → sent (manual confirm)
                 → rejected (manual reject)
sent             → send_failed (delivery failure)
send_failed      → retrying (auto-retry, if configured)
```

> This view feature exposes all statuses as read-only. Status transitions are not triggered from this UI (except via the existing `update_status` endpoint, which is already wired).

---

## No Migration Required (baseline)

Unless Spike 1 reveals `overall_confidence` is computed, no schema changes are needed for this feature.
