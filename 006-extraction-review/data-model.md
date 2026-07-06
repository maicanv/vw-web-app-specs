# Data Model: Extraction Review

**Date**: 2026-07-06 | **Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Existing models are reused; this feature adds two models and mutates none destructively. Variant parts are tagged with their OQ.

## Existing (unchanged schema, new behaviour)

### ExtractionRecord (`apps/document_entries/models.py`)
- `overall_confidence: int | None` — now compared against the document type threshold at extraction write-back (OQ-1 governs computation).
- `status` — `NEEDS_REVIEW` now also set by confidence rules (today: only missing critical fields). Existing `ALLOWED_STATUS_TRANSITIONS` unchanged: `needs_review → {extracted, skipped, rejected}`.
- `needs_review_reason: str` — now structured/multi-reason capable: newline-joined human-readable reasons ("Overall confidence 61% below threshold 80%", "Field 'IBAN' 42% below threshold", "Confidence unavailable"). Never cleared on transition (feeds FR-012 "originally flagged").

### ExtractedFieldValue
- `raw_value` — becomes the immutable extracted original (write-once, already the convention).
- `standardized_value` / `display_value` — overwritten by corrections.
- `is_manually_edited` — set `True` by the correction endpoint (today: dead column).
- `confidence` — never mutated by corrections (OQ-3 display handled in serializer).

### DocumentType / DocumentTypeField
- `global_confidence_threshold`, `confidence_threshold` — already stored; now enforced. No schema change.

### RouteDelivery / DeliveryAttempt
- Unchanged. Re-send after correction rides the existing `ALLOW_RESEND` flow; payload is rebuilt from current field values at send time, so corrected values flow through with zero delivery-side change.

### auditlog LogEntry (django-auditlog, already registered)
- Source for the user-facing audit endpoint. Field-value saves produce before → after diffs; status transitions produce confirm/reject events. Exposed via a record-scoped read-only endpoint (see contracts).

## New models

### ReviewerAssignment (new, `apps/document_entries/models.py`)

| Field | Type | Notes |
|-------|------|-------|
| `organisation` | FK organisations.Organisation | tenant scoping (constitution I) |
| `document_type` | FK DocumentType, `related_name="reviewer_assignments"` | **OQ-5 seam** — axis A. Option B: replace with FK data_source; Option C: both nullable + CheckConstraint(exactly one); Option D: FK workspace only |
| `user` | FK users.User | must be an org/workspace member (validated in serializer, A6) |
| `created_by` | FK users.User | who assigned (audit) |
| timestamps | BaseModel | |

Constraints: `UniqueConstraint(document_type, user)`. auditlog-registered.

Derived predicate (single seam for OQ-5/OQ-6):
- `records_for_reviewer(user) -> QuerySet[ExtractionRecord]` — needs_review records of assigned document types; workspace admins/managers additionally see records of document types with **zero** assignments (FR-020).
- `can_review(user, record) -> bool` — has assignment on the record's document type, or admin/manager fallback per existing document_entries access policy.

### ExtractionRecordLock (new, `apps/locks/models.py`, extends `AbstractLock`)

| Field | Type | Notes |
|-------|------|-------|
| `resource` | OneToOne ExtractionRecord, `related_name="lock"` | mirrors ApplicationLock pattern |
| inherited | holder, organisation, key, expiry | from AbstractLock |

- TTL default 15 min, refreshed on save activity (**OQ-8**: hard enforcement = correction endpoint rejects non-holders with 409; soft = UI-only warning; model identical either way).
- `AbstractAccessOverride` subclass only if takeover proves needed — deferred.

## State transitions (unchanged set, new triggers)

```
extracted ──(auto/manual send)──► sent / send_failed
needs_review ──confirm──► extracted ──► (auto-delivery if routes configured)
needs_review ──reject───► rejected   (terminal, excluded from delivery)
needs_review ──skip─────► skipped
```

New trigger into `needs_review` at extraction write-back, evaluated in order, reasons accumulated:
1. Missing critical fields (existing rule, unchanged)
2. Confidence unavailable (`overall_confidence is None`) — FR-004
3. Overall confidence below document-type threshold — FR-001
4. Field confidence below threshold, per **OQ-2** predicate (default: critical fields only) — FR-002

Corrections never change status (**OQ-3** default); Confirm is the only path out of the hold.

## Validation rules

- Correction payload: values validated against the field's `field_type`/`FieldConfig` (reuse existing field validation from the wizard serializers).
- Corrections allowed on any record status (FR-008), including `sent`; on non-sent records the record stays/goes eligible per existing `DELIVERABLE_RECORD_STATUSES`.
- Reviewer assignment: user must belong to the document type's organisation; assigner must hold the document-type manage permission (**OQ-6** default).
- Lock: corrections by non-holders rejected while an unexpired lock exists (**OQ-8** default hard).
