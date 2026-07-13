# Research: Extraction Review — Confidence & Control Step

**Date**: 2026-07-07 (rewritten after source-story revision; supersedes the 2026-07-06 open-question version) | **Spec**: [spec.md](./spec.md)

The prior version of this file planned around open questions OQ-1..OQ-10. The story revision answered all of them (spec Resolved Decisions D1–D9) and added scope: configurable review rules (Review Records step), a dedicated Needs Review sidebar queue, a platform-wide Audit section, a dedicated Reviewer role, and two changes to existing delivery settings (FR-028/029). Reviewer notifications were dropped (A7); Record UID selection and post-review delivery method/UID were dropped as too much work for now (A9). This file records the design decisions for the revised scope, grounded in the current code.

## As-built baseline (verified in code)

- `ExtractionRecord.overall_confidence` = mean of field confidences × 100 (`services.py:510-513`). `needs_review_reason` free-text exists and is never cleared on transition.
- Needs Review currently fires **only** on missing critical fields (`services.py:587-593`); the stored thresholds (`DocumentType.global_confidence_threshold`, `DocumentTypeField.confidence_threshold`) are never read at runtime. `DocumentTypeField.is_critical` exists.
- `ALLOWED_STATUS_TRANSITIONS` (enums.py) models reviewer outcomes: `needs_review → {extracted, skipped, rejected}`. `update_status` on `ExtractionRecordViewSet` applies them and triggers auto-delivery on → extracted. **Confirm = transition to extracted; Reject = transition to rejected — these endpoints exist.** No transition *into* needs_review exists (needed for Select-for-review and reject re-open).
- `ExtractedFieldValue.is_manually_edited` exists, never set — no edit endpoint. `raw_value` / `standardized_value` / `display_value` columns exist per value.
- `OutputRoute` (models.py ~325) carries `delivery_mode` (auto|manual, enums.py `DeliveryMode`) and `repeat_policy` (allow_resend|prevent_duplicates, `RepeatPolicy`). HTTP method lives on the linked `ApiEndpoint` (via `WithSingleEndpointMixin`, `api_integrations/mixins.py:93`). `PayloadConfig.email_id_as_guid` (payload_config.py:41) is the only UID-shaped setting today.
- `DeliveryService.trigger_auto_delivery` (services.py:1146) filters routes on `delivery_mode=AUTO`; `RouteDeliveryService.initialize_and_dispatch` (~1180) parks MANUAL routes at `PENDING_APPROVAL`. The resend guard inside `DeliveryService.deliver` (~1049) blocks only when `repeat_policy == PREVENT_DUPLICATES` **and** the new payload hash equals `last_successful_payload_hash` — i.e. it blocks identical payloads, not resends per se; `is_override` (manual send) bypasses it.
- `django-auditlog`: `DocumentType`, `DocumentTypeField`, `ExtractionRecord` are registered; **`ExtractedFieldValue`, `OutputRoute`, `RouteDelivery` are NOT**. Actor capture via `AuditlogActorMixin` (common/utils/mixins.py). There is **no user-facing audit endpoint** — LogEntry only surfaces in the Django admin history view.
- `apps/locks` provides `AbstractLock` (holder, organisation, `expires_at` absolute timestamp, key, status, `is_locked`) with `ApplicationLock` as the pattern; base `ResourceLockViewSet` and `ResourceLockSerializer` exist.
- RBAC: roles are **dynamic org-scoped rows** (`apps/roles.Role`: `role_name`, `identifier`, `permissions: ArrayField[str]`) assigned via `RoleAssignment`; permission strings live in `common/constants/permissions.py` (`Permissions` StrEnum) and are checked by access policies in `common/access_policy/document_entry_type_access_policy.py`. Existing doc-entry perms: `document_types:edit`, `document_types:create-delete`.
- Frontend: sidebar in `client/src/app/layout/DefaultNav.tsx` (document-entry children at lines 88-107, Management/org area from line 111). Document-type wizard stepper in `DocumentTypeCreatePage.tsx` (steps rendered 568-618, `STEPS_WITHOUT_OUTPUT=5` / `STEPS_WITH_OUTPUT=6`, index-based `validateStep` in `documentTypeFormValidation.ts`). Records status filter (`needs_review` is one MultiSelect option) in `records/components/RecordFilters.tsx`. Route form fields (`delivery_mode`, `repeat_policy`, endpoint `method` default POST, `email_id_as_guid`) in `components/OutputRouteForm.tsx`.

## Decisions

### R1 — Review configuration storage (FR-001..003, US1)

**Decision**: a `review_config` `SchemaField` (django-pydantic-field) on `DocumentType`, holding a `ReviewConfig` pydantic model: `enabled: bool = True` plus one optional block per rule — `all_records: bool`, `average_below: int | None`, `sender_domains: list[str] | None`, `critical_field_below: int | None`, `any_field_below: int | None`, `specific_fields_below: int | None` + `specific_fields: list[str] | None` (the chosen `DocumentTypeField` ids, added 2026-07-13, FR-032). A rule is "enabled" when its value is set/true (for the specific-fields rule: `specific_fields_below` set **and** `specific_fields` non-empty). Evaluation is OR across enabled rules (spec clarification).

**Rationale**: mirrors the established `PayloadConfig` SchemaField pattern on `OutputRoute`; six rules with heterogeneous parameters fit one validated JSON blob far better than six column pairs or a rule-row table nobody joins against. Rules are only read at extraction write-back — no query-by-rule need. The specific-fields rule reuses the same threshold-plus-scope shape as critical-field-below, but lets the admin pick the field set explicitly instead of relying on the field's `is_critical` marker — useful when a field matters for one document type's review policy without being globally "critical".

**Alternatives rejected**: separate `ReviewRule` model (relational overkill for ≤6 singleton rules); flat columns on DocumentType (6 nullable columns + toggle, migration churn every time a rule is added); reusing `critical_field_below` by having the multi-select set `is_critical` on the chosen fields (rejected — conflates a field-level marker used elsewhere with a rule-scoped selection, and would mutate field definitions as a side-effect of saving a review rule).

**Migration/back-compat**: data migration seeds `average_below` from `global_confidence_threshold` where set (so current stored thresholds become live rules, matching user intent when they set them). `global_confidence_threshold` and `DocumentTypeField.confidence_threshold` columns stay (non-destructive) but stop being the source of truth for flagging; per-field thresholds may refine `any_field_below`/`critical_field_below` later (spec A2).

### R2 — Flag evaluation (FR-004..007, US2)

**Decision**: `evaluate_needs_review(record, field_values, review_config) -> list[str]` in `services.py`, called at extraction write-back. Rules in order, reasons accumulated (newline-joined into `needs_review_reason`): missing critical fields (existing rule, kept), confidence unavailable (`overall_confidence is None`), then the enabled config rules (all-records, average-below, sender-domain match on the record's sender, critical-field-below, any-field-below, specific-fields-below — checks each `field_values` entry whose `field_id` is in `review_config.specific_fields` against `specific_fields_below`). Any non-empty reason list ⇒ status `needs_review`. `compute_overall_confidence` stays the mean (D1), extracted into a named function.

**Rationale**: OR semantics fall out of reason accumulation; structured reasons feed FR-005 display and FR-019 audit ("originally flagged"). Sender-domain rule matches against `ExtractionRecord.sender`'s domain part.

### R3 — Transitions into needs_review (US4 Select-for-review, reject re-open)

**Decision**: extend `ALLOWED_STATUS_TRANSITIONS` with `extracted → needs_review`, `rejected → needs_review`, `sent → needs_review`, `send_failed → needs_review`, applied through the existing `update_status` action. A manual selection appends a "Selected for review by <user>" reason.

**Rationale**: the spec requires Select-for-review on unflagged records (any lifecycle stage) and reject re-open (clarification Q4). Reusing `update_status` keeps one transition enforcement point and auto-audits via the registered model. No new endpoint.

### R4 — Correction persistence (FR-014..016, US5)

**Decision**: unchanged from the previous plan — `correct/` bulk endpoint; `raw_value` is the immutable extracted original; corrections overwrite `standardized_value`/`display_value` and set `is_manually_edited=True`; stored confidence never mutated (D4: keep score + edited badge); correction never changes status — explicit Confirm required (D4).

**Alternatives rejected**: `FieldValueRevision` history table — auditlog (now registered on `ExtractedFieldValue`, see R6) captures per-save diffs.

### R5 — Delivery-setting changes (FR-028/029)

**FR-028 Resend policy**: rename `repeat_policy` → `resend_policy` with values `allow` | `block` (lossless data migration `allow_resend→allow`, `prevent_duplicates→block`). **Semantic change**: `block` refuses any manual re-send of a delivery that already succeeded (`last_successful_payload_hash is not None` / status ever SENT), not merely byte-identical payloads. Guard moves from the hash comparison in `DeliveryService.deliver` to the resend-eligibility check (`get_can_resend` + `RouteDeliveryViewSet.send`). Coordinated backend+frontend rename is safe: the client ships from the same repo; no external consumers of this admin-config field.

**FR-029 Delivery Mode removal**: drop `OutputRoute.delivery_mode` and the MANUAL/`PENDING_APPROVAL` parking path in `initialize_and_dispatch`. Data migration first: any document type owning a `manual` route gets `review_config.all_records = True` (behaviour-preserving — its records now hold in review instead of pending approval), then the column and enum are removed. `PENDING_APPROVAL` status stays in the enum for historical rows; nothing new enters it.

**Dropped from scope**: a Record UID field selection and a distinct post-review delivery method/UID (originally proposed in the source story) were cut as too much work for now. A reviewed (confirmed or corrected) record is delivered through the exact same payload-build and method the route already uses for its normal flow — no new `OutputRoute` columns for this. Revisit if a downstream system needs a different identifier or method after review.

### R6 — Platform-wide Audit section (FR-019..021, US7)

**Decision**: register `ExtractedFieldValue` with auditlog (masking nothing new — values already render in the product; exclude noisy internals) so corrections produce before→after diffs. New read-only, org-scoped endpoint `GET /api/v1/audit/` (new thin `apps/audit` DRF viewset over `auditlog.models.LogEntry`) with a `subsystem` filter; the first subsystem, `extraction_records`, maps to LogEntry rows for `ExtractionRecord` + `ExtractedFieldValue` (+ delivery attempts serialized as `resend` actions). Serializer maps LogEntry → {actor, timestamp, action ∈ confirm|correct|reject|select_for_review|resend|status_change, changes (before→after), originally_flagged (from `needs_review_reason`)}. Org scoping: LogEntry has no organisation FK → filter by (content_type, object_pk ∈ org-scoped subquery) per subsystem; each subsystem contributes its own scoping rule.

**Rationale**: FR-020 demands a platform surface, not a per-record tab — a dedicated small app keeps document_entries from owning a platform concern while subsystems stay pluggable (A8: only the extraction subsystem ships now). Auditlog reuse avoids a bespoke ReviewAction table.

**Alternatives rejected**: record-scoped `audit/` sub-endpoint only (previous plan) — doesn't satisfy FR-020/021; new ReviewAction model — duplicates auditlog.

### R7 — Reviewer role + assignment (FR-023..027, US9)

**Decision**: per spec clarification, **one global Reviewer role, scoped by assignment**. Concretely: new permission constant `DOCUMENT_RECORDS_REVIEW = "document_records:review"`; a seeded org-scoped `Role(identifier="reviewer")` carrying it (created on demand); `ReviewerAssignment(organisation, document_type, user, created_by)` in document_entries with `UniqueConstraint(document_type, user)`. Creating a user's first assignment ensures the Reviewer `RoleAssignment`; deleting their last removes it. Access: `can_review(user, record)` = assignment on the record's document type, or admin/manager fallback via existing policy; document types with **zero** assignments fall back to workspace admins/managers (FR-027). Queue: `records_for_reviewer(user)` powers the sidebar queue + count. Assigning reviewers requires the existing `document_types:edit` permission.

**Rationale**: matches the roles-app reality (dynamic rows, permission arrays) — "role = capability, assignment = scope" maps to RoleAssignment + ReviewerAssignment respectively, with the access policy checking the permission string and the queryset applying scope.

### R8 — Record lock (FR-026)

**Decision**: `ExtractionRecordLock(AbstractLock)` in `apps/locks`, OneToOne `resource → ExtractionRecord`, **hard** enforcement (correct/confirm/reject from non-holders → 409 while an unexpired lock exists), TTL **5 minutes** (D7 — changed from the previous plan's 15), refreshed on save activity. Claim/release via the existing `ResourceLockViewSet` base pattern. `AbstractAccessOverride` subclass deferred until takeover is needed.

### R9 — Needs Review queue surface (FR-011..013, US4)

**Decision**: new sidebar entry "Needs Review" under the document-entry section (`DefaultNav.tsx`) with a pending-count badge from a lightweight `GET /extraction-records/review-queue-count/`; the queue page reuses the records list component filtered server-side by `review_queue=true` (= `records_for_reviewer(user)`). Remove `needs_review` from the status MultiSelect options in `RecordFilters.tsx` (FR-012). "Select for review" = row/detail action calling `update_status` → `needs_review` (R3).

### R10 — Dropped from the previous plan

- **Notifications** (`notify_reviewers`, throttling): out of scope this revision (A7). The pending count is the in-app signal.
- **Open-question branch seams / OQ tags**: obsolete — all questions resolved; the seams collapse into the single implementations above.
- **Round-robin (D8) and bulk-assign (D9)**: out of scope; `ReviewerAssignment` accommodates both later without schema change.
