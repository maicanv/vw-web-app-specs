# Data Model: Extraction Review

**Date**: 2026-07-07 (rewritten after source-story revision) | **Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Two new models, one new pydantic config, column changes on `OutputRoute`, and one auditlog registration. All migrations lossless; the single column drop (`delivery_mode`) is preceded by a behaviour-preserving data migration.

## Changed existing models

### DocumentType (`apps/document_entries/models.py`)

| Change | Detail |
|--------|--------|
| + `review_config` | `SchemaField(schema=ReviewConfig, default=ReviewConfig)` — see below |
| `global_confidence_threshold` | kept (non-destructive) but no longer read at flag time; data migration seeds `review_config.average_below` from it where set |

**ReviewConfig** (new pydantic model, `review_config.py`, mirrors `payload_config.py`):

| Field | Type | Meaning |
|-------|------|---------|
| `enabled` | `bool = True` | Review Records toggle (FR-001) |
| `all_records` | `bool = False` | rule: every record to review |
| `average_below` | `int \| None` (0–100) | rule: overall (mean) confidence below N |
| `sender_domains` | `list[str] \| None` | rule: sender's domain in list |
| `critical_field_below` | `int \| None` (0–100) | rule: any critical-marked field below N |
| `any_field_below` | `int \| None` (0–100) | rule: any field below N |
| `specific_fields_below` | `int \| None` (0–100) | rule: any of `specific_fields` below N (FR-032) |
| `specific_fields` | `list[str] \| None` | `DocumentTypeField` ids the "specific fields below N" rule checks; populated by the wizard's multi-select dropdown |

A rule is enabled when set/true (the specific-fields rule additionally requires `specific_fields` non-empty); enabled rules OR-combine (FR-002). Validation: thresholds 0–100; domains normalised lowercase, no `@`; `specific_fields` ids must belong to the same document type and to a field type that carries a confidence score (A10).

### OutputRoute

| Change | Detail |
|--------|--------|
| `repeat_policy` → `resend_policy` | values `allow` \| `block` (data migration: `allow_resend→allow`, `prevent_duplicates→block`). **Block** = refuse manual re-send of an already successfully delivered record (FR-028) — enforced at resend eligibility, replacing the payload-hash-equality guard |
| − `delivery_mode` | dropped (FR-029). Data migration first: document types owning a `manual` route get `review_config.all_records = True`; `PENDING_APPROVAL` stays in the status enum for historical rows only |

Record UID selection and a distinct post-review delivery method/UID were dropped from scope (too much work for now). A reviewed (confirmed or corrected) record is delivered through the route's existing payload build and method, unchanged.

### ExtractionRecord — schema unchanged, behaviour changes

- `overall_confidence` — mean of field confidences (D1), now compared against `review_config` rules at extraction write-back.
- `needs_review_reason` — newline-joined structured reasons ("Overall confidence 61% below threshold 80%", "Field 'IBAN' 42% below threshold 80%", "Confidence unavailable", "Selected for review by Jane Doe"). Never cleared; doubles as the "went through review" marker for the "originally flagged" audit flag.

### ExtractedFieldValue — schema unchanged

- `raw_value` — immutable extracted original (before).
- `standardized_value` / `display_value` — overwritten by corrections (after).
- `is_manually_edited` — set `True` by the correction endpoint.
- `confidence` — never mutated (D4); UI shows the edited badge.
- **Now `@auditlog.register`ed** (currently unregistered) so corrections yield before → after LogEntry diffs for the Audit section.

### auditlog LogEntry (django-auditlog)

Source for the platform Audit API (`apps/audit`). No org FK → the audit endpoint scopes per subsystem via org-scoped object subqueries (research R6). Subsystem `extraction_records` = LogEntry rows for `ExtractionRecord` + `ExtractedFieldValue`, plus delivery attempts mapped as `resend`.

## New models

### ReviewerAssignment (new, `apps/document_entries/models.py`)

| Field | Type | Notes |
|-------|------|-------|
| `organisation` | FK organisations.Organisation | tenant scoping (constitution I) |
| `document_type` | FK DocumentType, `related_name="reviewer_assignments"` | routing axis (D5) |
| `user` | FK users.User | must be an org/workspace member (A5, serializer-validated) |
| `created_by` | FK users.User | who assigned |
| timestamps | BaseModel | |

Constraints: `UniqueConstraint(document_type, user)`. auditlog-registered.

Role linkage (D6, clarification): permission constant `DOCUMENT_RECORDS_REVIEW = "document_records:review"`; org-scoped seeded `Role(identifier="reviewer")` carrying it. First assignment for a user ensures the Reviewer `RoleAssignment`; deleting their last assignment removes it. Role = capability (access policy checks the permission string), assignment = scope (queryset filter).

Derived predicates:
- `records_for_reviewer(user) -> QuerySet[ExtractionRecord]` — needs_review records of assigned document types; workspace admins/managers additionally see records of document types with **zero** assignments (FR-027).
- `can_review(user, record) -> bool` — assignment on the record's document type, or admin/manager fallback per existing access policy.

### ExtractionRecordLock (new, `apps/locks/models.py`, extends `AbstractLock`)

| Field | Type | Notes |
|-------|------|-------|
| `resource` | OneToOne ExtractionRecord, `related_name="lock"` | mirrors ApplicationLock pattern |
| inherited | holder, organisation, key, `expires_at`, status | from AbstractLock |

- **Hard** lock (D7): correct/confirm/reject from non-holders → 409 while `is_locked`. TTL **5 minutes** (`expires_at = now + 5min`), refreshed on save activity.
- Removing a reviewer's assignments releases their active locks (edge case: reviewer removed mid-edit).
- `AbstractAccessOverride` subclass deferred until takeover is needed.

## State transitions

Existing set plus new transitions **into** `needs_review` (R3):

```
needs_review ──confirm──► extracted ──► (auto-delivery if routes configured)
needs_review ──reject───► rejected     (reversible; excluded from delivery while rejected)
needs_review ──skip─────► skipped
extracted / sent / send_failed ──select for review──► needs_review   (NEW, FR-013)
rejected ──select for review / re-open──► needs_review               (NEW, clarification Q4)
```

Triggers into `needs_review` at extraction write-back (reasons accumulated, OR):
1. Missing critical fields (existing rule, kept)
2. Confidence unavailable (`overall_confidence is None`) — FR-006
3. Enabled `review_config` rules: all-records, average-below, sender-domain, critical-field-below, any-field-below, specific-fields-below — FR-002/004/032

Review disabled (`review_config.enabled = False`) ⇒ no auto-flagging (FR-003); select-for-review still works.

Corrections never change status (D4); Confirm is the only path out of the hold.

## Validation rules

- Correction payload: values validated against the field's `field_type`/`FieldConfig` (reuse wizard serializers' validation). Allowed on any status, including `sent` (FR-014); unsent corrected records remain/become eligible per `DELIVERABLE_RECORD_STATUSES` (FR-016).
- ReviewConfig: thresholds 0–100; enabling review with zero rules is valid but flags nothing (warn in UI).
- ReviewConfig specific-fields rule: `specific_fields` must be non-empty and reference only fields belonging to this document type with a confidence-bearing type (A10); saving `specific_fields_below` with an empty `specific_fields` list is a validation error, not a silently-disabled rule.
- Reviewer assignment: user must belong to the document type's organisation; assigner needs `document_types:edit`.
- Lock: mutations by non-holders rejected (409) while an unexpired lock exists.
- Resend: manual re-send refused when `resend_policy = block` and the delivery already succeeded (FR-018).
