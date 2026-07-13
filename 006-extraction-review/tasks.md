---

description: "Task list for Extraction Review — Confidence & Control Step"
---

# Tasks: Extraction Review — Confidence & Control Step

**Input**: Design documents from `/specs/006-extraction-review/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/api.md](./contracts/api.md), [quickstart.md](./quickstart.md)

**Tests**: Included per project convention (behaviour tests, backend only — no frontend tests unless explicitly requested). Run via `docker compose exec django pytest ...` from `backend/`.

**Organization**: Tasks are grouped by user story (spec.md priorities) so each story ships and is testable independently. Migrations require explicit user confirmation before running per project convention — tasks that run `makemigrations`/`migrate` are flagged.

## Table of Contents

- [Phase 1: Setup](#phase-1-setup)
- [Phase 2: Foundational](#phase-2-foundational-blocking-prerequisites)
- [Phase 3: User Story 1 — Configure review rules (P1)](#phase-3-user-story-1---configure-which-records-go-to-review-priority-p1)
- [Phase 4: User Story 2 — Flag records (P1)](#phase-4-user-story-2---flag-records-for-review-by-the-configured-rules-priority-p1)
- [Phase 5: User Story 3 — Hold before sending (P1)](#phase-5-user-story-3---hold-flagged-records-before-sending-priority-p1)
- [Phase 6: User Story 4 — Needs Review queue (P2)](#phase-6-user-story-4---dedicated-needs-review-queue-in-the-sidebar-priority-p2)
- [Phase 7: User Story 5 — Correct values (P2)](#phase-7-user-story-5---correct-values-on-any-record-priority-p2)
- [Phase 8: User Story 6 — Re-send + delivery settings (P2)](#phase-8-user-story-6---re-send-after-correction-priority-p2)
- [Phase 9: User Story 7 — Platform audit (P2)](#phase-9-user-story-7---platform-wide-audit-section-priority-p2)
- [Phase 10: User Story 8 — Threshold signalling (P3)](#phase-10-user-story-8---signal-low-confidence-in-the-record-ui-priority-p3)
- [Phase 11: User Story 9 — Reviewer assignment (P3)](#phase-11-user-story-9---reviewer-assignment-with-a-dedicated-reviewer-role-priority-p3-dependent-story)
- [Phase 12: Polish & Cross-Cutting](#phase-12-polish--cross-cutting-concerns)
- [Dependencies & Execution Order](#dependencies--execution-order)
- [Implementation Strategy](#implementation-strategy)

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Maps the task to a spec.md user story (US1–US9)
- Every task states an exact file path

## Path Conventions

- Backend: `backend/django/apps/document_entries/`, `backend/django/apps/locks/`, `backend/django/apps/audit/` (new), `backend/django/common/`
- Backend tests: `backend/django/tests/test_apps/test_document_entries/`, `backend/django/tests/test_apps/test_audit/` (new)
- Frontend: `client/src/app/documentEntry/`, `client/src/app/layout/`, `client/src/app/audit/` (new)
- All backend commands run from `backend/` via `docker compose exec django ...`

---

## Phase 1: Setup

**Purpose**: Nothing new to scaffold — the feature extends the existing `apps.document_entries` app and `apps.locks`, plus one new thin app.

- [ ] T001 Create the new `apps/audit` Django app skeleton (`backend/django/apps/audit/__init__.py`, `apps.py`, `urls.py`) and register it in `backend/django/manage_platform/settings.py` `INSTALLED_APPS`; wire `apps/audit/urls.py` into the project's root URL config at `/api/v1/audit/`

**Checkpoint**: `apps/audit` importable and routed; no behaviour yet.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Schema, config model, and permission plumbing that every user story reads or writes. No user story can be implemented until this phase is done and migrated.

**⚠️ CRITICAL**: Do not start Phase 3+ until migrations in this phase are generated, reviewed, and the user has confirmed running them.

- [ ] T002 [P] Add `ReviewConfig` pydantic model (`enabled: bool = True`, `all_records: bool = False`, `average_below: int | None`, `sender_domains: list[str] | None`, `critical_field_below: int | None`, `any_field_below: int | None`, `specific_fields_below: int | None`, `specific_fields: list[str] | None`) in new file `backend/django/apps/document_entries/review_config.py`, mirroring `payload_config.py`'s style; validate `specific_fields_below` requires a non-empty `specific_fields`
- [ ] T003 Add `DocumentType.review_config = SchemaField(schema=ReviewConfig, default=ReviewConfig)` in `backend/django/apps/document_entries/models.py` (depends on T002)
- [ ] T004 Rename `OutputRoute.repeat_policy` → `resend_policy`, add `ResendPolicy` choices (`allow`, `block`) replacing `RepeatPolicy` in `backend/django/apps/document_entries/enums.py`, and update the field in `backend/django/apps/document_entries/models.py`
- [ ] T005 Run `docker compose exec django python manage.py makemigrations document_entries` for T003/T004 (schema migration adding `review_config`, renaming `repeat_policy`→`resend_policy` with a values-preserving `RunPython` data step `allow_resend→allow`, `prevent_duplicates→block`) — **stop and get user confirmation before running `migrate`**
- [ ] T006 [P] Add a data migration in `backend/django/apps/document_entries/migrations/` seeding `review_config.average_below` from `DocumentType.global_confidence_threshold` where set (reverse operation: no-op restore, since the source column is untouched) — **user-confirmed migrate**
- [ ] T007 [P] Add `DOCUMENT_ENTRIES_REVIEW = "document_entries:review"` to `backend/django/common/constants/permissions.py` under a new "Document Entries Review Permissions" section
- [ ] T008 [P] Add `ExtractionRecordLock(AbstractLock)` in `backend/django/apps/locks/models.py`: `resource = models.OneToOneField("document_entries.ExtractionRecord", on_delete=models.CASCADE, related_name="lock")`, `UniqueConstraint(fields=["organisation", "holder", "resource"], name="unique_extraction_record_lock")`, `@auditlog.register(exclude_fields=["key", "created_at", "updated_at"])` — mirrors `ApplicationLock`
- [ ] T009 Run `docker compose exec django python manage.py makemigrations locks` for T008 — **user-confirmed migrate**
- [ ] T010 [P] Register `ExtractedFieldValue` with `django-auditlog` in `backend/django/apps/document_entries/models.py` (`@auditlog.register(exclude_fields=[...])` above the class, excluding write-once snapshot fields per the model's existing `Meta` comment: `field_codename`, `field_name`, `field_type`, `is_critical`, `display_order`, `created_at`, `updated_at`)
- [ ] T011 Extend `ALLOWED_STATUS_TRANSITIONS` in `backend/django/apps/document_entries/enums.py`: add `ExtractionRecordStatus.EXTRACTED`, `SENT`, `SEND_FAILED`, `REJECTED` → `{ExtractionRecordStatus.NEEDS_REVIEW}` (Select-for-review / reject re-open, depends on T004 for the enums module already being touched)

**Checkpoint**: `review_config` and `resend_policy` exist and are migrated; `ExtractedFieldValue` is audited; the lock model exists; new transitions are declared. User stories can now proceed.

---

## Phase 3: User Story 1 - Configure which records go to review (Priority: P1) 🎯 MVP

**Goal**: An admin can toggle review on/off for a document type and enable any combination of six OR-combined flag rules.

**Independent Test**: Open a document type's Review Records step, toggle review on, enable "average confidence below N", save, and verify the setting persists and reads back correctly via `GET /document-types/{id}/`.

### Tests for User Story 1

- [ ] T012 [P] [US1] Test `review_config` persistence and defaults (`enabled=True` on a fresh document type), plus `specific_fields_below` validation (rejects an empty `specific_fields` list, rejects a field id from another document type) in `backend/django/tests/test_apps/test_document_entries/test_document_type_viewset.py`
- [ ] T013 [P] [US1] Test the `average_below` seeding migration converts an existing `global_confidence_threshold` value correctly in a new `backend/django/tests/test_apps/test_document_entries/test_migrations.py`

### Implementation for User Story 1

- [ ] T014 [US1] Add `review_config` to `DocumentTypeSerializer` / `DocumentTypeUpdateSerializer` in `backend/django/apps/document_entries/serializers.py`, validating thresholds 0–100, lower-casing/stripping `@` from `sender_domains` entries, and validating `specific_fields` ids belong to this document type and to a confidence-bearing field type
- [ ] T015 [US1] Rename `client/src/app/documentEntry/documentTypes/steps/ConfidenceStep.tsx` to `ReviewRecordsStep.tsx` (avoids collision with the existing `ReviewStep.tsx` final wizard step) and implement the toggle + six rule inputs (thresholds via `NumberInput`, `sender_domains` via a tags input, and "specific fields below N" via a `NumberInput` + `MultiSelect` sourced from the wizard's in-progress field list, excluding boolean/group field types), wired to `review_config` on the form
- [ ] T016 [US1] Register `ReviewRecordsStep` in the stepper in `client/src/app/documentEntry/documentTypes/DocumentTypeCreatePage.tsx` (import, add to the steps array, bump `STEPS_WITHOUT_OUTPUT` / `STEPS_WITH_OUTPUT`, extend `validateStep` in `client/src/app/documentEntry/documentTypes/documentTypeFormValidation.ts` for threshold bounds)
- [ ] T017 [US1] Add `review_config` to the document-type form types/payload in `client/src/app/documentEntry/documentTypes/useDocumentTypeForm.ts` (or the equivalent form-state file) and `buildSavePayload` in `DocumentTypeCreatePage.tsx`
- [ ] T018 [US1] Add new i18n keys to `client/src/translations/en.json` only, per project convention — do not add or fill them in the secondary locale files (they are translated later in bulk)

**Checkpoint**: Review Records step is fully functional and independently testable — config saves and loads.

---

## Phase 4: User Story 2 - Flag records for review by the configured rules (Priority: P1)

**Goal**: After extraction, records matching an enabled rule are marked Needs Review with a structured reason.

**Independent Test**: Enable a rule, run an extraction that matches it, and verify the record is `needs_review` with the matching reason; run one that doesn't match and verify it stays unflagged.

### Tests for User Story 2

- [ ] T019 [P] [US2] Unit tests for `compute_overall_confidence` (mean of field confidences, `None` when no scorable fields) in `backend/django/tests/test_apps/test_document_entries/test_services.py`
- [ ] T020 [P] [US2] Unit tests for `evaluate_needs_review`: each rule independently (incl. specific-fields-below with 1 and 2+ selected fields, and a case where a non-selected field is below N but does not flag), OR combination across multiple enabled rules, confidence-unavailable reason, boundary case (`confidence == threshold` does not flag), `review_config.enabled=False` → no auto-flag, in `backend/django/tests/test_apps/test_document_entries/test_services.py`

### Implementation for User Story 2

- [ ] T021 [US2] Extract the existing inline confidence-mean calculation into `compute_overall_confidence(field_values) -> int | None` in `backend/django/apps/document_entries/services.py`
- [ ] T022 [US2] Implement `evaluate_needs_review(record, field_values, review_config) -> list[str]` in `backend/django/apps/document_entries/services.py`: OR-combines missing-critical-fields (existing rule), confidence-unavailable, and the six `review_config` rules (incl. specific-fields-below, checking each `field_values` entry whose `field_id` is in `review_config.specific_fields`); returns human-readable reasons (e.g. `"Overall confidence 61% below threshold 80%"`, `"Field 'IBAN' 42% below threshold 80%"`, `"Confidence unavailable"`)
- [ ] T023 [US2] Call `evaluate_needs_review` at extraction write-back (where the missing-critical-fields check currently fires) in `backend/django/apps/document_entries/services.py`; join returned reasons with `\n` into `ExtractionRecord.needs_review_reason` and set `status = needs_review` when non-empty

**Checkpoint**: Flagging works end-to-end from extraction through to a visible `needs_review` status with a reason — testable independently of the queue/UI work in later phases.

---

## Phase 5: User Story 3 - Hold flagged records before sending (control step) (Priority: P1)

**Goal**: `needs_review` records never auto-send; unflagged records still do. The reviewer explains-why UI and the three actions (Confirm & Send, Correct, Reject) are available.

**Independent Test**: With automatic REST output configured, produce one flagged and one unflagged record; verify only the unflagged one sends automatically.

### Tests for User Story 3

- [ ] T024 [P] [US3] Integration test: a `needs_review` record is excluded from `DeliveryService`'s auto-delivery path and from manual-send eligibility, in `backend/django/tests/test_apps/test_document_entries/test_delivery_service.py`
- [ ] T025 [P] [US3] Integration test: confirming a `needs_review` record via `update_status` transitions it to `extracted` and triggers auto-delivery when applicable (regression pin on existing behaviour), in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`

### Implementation for User Story 3

- [ ] T026 [US3] Verify `DELIVERABLE_STATUSES` / `DELIVERABLE_RECORD_STATUSES` in `backend/django/apps/document_entries/enums.py` already exclude `needs_review` (no code change expected — confirm via T024); if a gap is found, close it here
- [ ] T027 [US3] Add a blocked-send explainer to `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`: when `status === 'needs_review'`, render the flag reasons (from `needs_review_reason`) and the Confirm & Send / Correct / Reject action bar (Confirm/Reject reuse the existing `update_status` call; Correct is wired in Phase 7)

**Checkpoint**: The hold is enforced and explained in the UI — flagged records are provably safe from auto-delivery.

---

## Phase 6: User Story 4 - Dedicated Needs Review queue in the sidebar (Priority: P2)

**Goal**: A sidebar entry lists all Needs Review records with a pending count; the old list filter is removed; Select-for-review adds an unflagged record to the queue.

**Independent Test**: Produce several `needs_review` records, verify they all appear under the sidebar entry with a correct count; verify the old list filter is gone; use Select for review on an unflagged record and verify it joins the queue.

### Tests for User Story 4

- [ ] T028 [P] [US4] Test `records_for_reviewer`-backed `review_queue=true` filter returns only `needs_review` records in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`
- [ ] T029 [P] [US4] Test `review-queue-count/` returns the correct count in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`
- [ ] T030 [P] [US4] Test new transitions: `extracted`/`sent`/`send_failed` → `needs_review` (select-for-review, appends "Selected for review by <user>" to `needs_review_reason`) and `rejected` → `needs_review` (reject re-open) via `update_status`, in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`

### Implementation for User Story 4

- [ ] T031 [US4] Add a `review_queue` boolean predicate to `ExtractionRecordFilterSet` in `backend/django/apps/document_entries/filters.py` (initially: `status=needs_review` scoped to the requesting user's accessible document types — the `records_for_reviewer` refinement lands in Phase 11 once `ReviewerAssignment` exists)
- [ ] T032 [US4] Add a `review-queue-count/` list-level `@action` (`detail=False`, `methods=["get"]`) to `ExtractionRecordViewSet` in `backend/django/apps/document_entries/extraction_record_view_set.py`, returning `{"count": <int>}` using the same `review_queue` predicate
- [ ] T033 [US4] Extend `update_status` handling in `backend/django/apps/document_entries/extraction_record_view_set.py` / `UpdateStatusSerializer` in `serializers.py` to append `"Selected for review by {user}"` to `needs_review_reason` when the target status is `needs_review` and the source was not already `needs_review`
- [ ] T034 [US4] Add a `NeedsReviewPage.tsx` in `client/src/app/documentEntry/records/` that reuses `ExtractionRecordListPage.tsx`'s list components filtered by `review_queue=true`
- [ ] T035 [US4] Add a **Needs Review** entry with a pending-count badge (polling `review-queue-count/`) to `client/src/app/layout/DefaultNav.tsx`, routed to `NeedsReviewPage`
- [ ] T036 [US4] Remove the `needs_review` option from the status filter in `client/src/app/documentEntry/records/components/RecordFilters.tsx`
- [ ] T037 [US4] Add a **Select for review** control to `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx` (and/or the list row actions) for non-`needs_review` records, calling `update_status` with `{"status": "needs_review"}`

**Checkpoint**: The dedicated queue, count, and select-for-review flow work end-to-end.

---

## Phase 7: User Story 5 - Correct values on any record (Priority: P2)

**Goal**: Users can fix extracted values on any record regardless of status; before/after is visible; corrections don't change status.

**Independent Test**: Open any record, change a value, verify persistence, before → after display, kept confidence with edited marker, and (for unsent records) sendability.

### Tests for User Story 5

- [ ] T038 [P] [US5] Test `correct/` endpoint: values persist, `is_manually_edited=True`, `raw_value` untouched, `confidence` untouched, status untouched, allowed on `sent` records, type-validation 400s, in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`
- [ ] T039 [P] [US5] Test corrections produce before→after `LogEntry` diffs via the newly registered `ExtractedFieldValue` auditlog, in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`

### Implementation for User Story 5

- [ ] T040 [US5] Add a `CorrectValuesSerializer` (`values: [{"field_value_id", "value"}]`) in `backend/django/apps/document_entries/serializers.py`, validating each value against its `ExtractedFieldValue.field_type` (reuse the wizard's `FieldConfig` validation)
- [ ] T041 [US5] Add `correct` `@action` (`detail=True`, `methods=["post"]`) to `ExtractionRecordViewSet` in `backend/django/apps/document_entries/extraction_record_view_set.py`: writes `standardized_value`/`display_value`, sets `is_manually_edited=True`, never touches `status` or `confidence`
- [ ] T042 [US5] Add `original_value` (from `raw_value`) to the field-value representation in `ExtractionRecordDetailSerializer` in `backend/django/apps/document_entries/serializers.py`
- [ ] T043 [US5] Add inline click-to-edit correction to `client/src/app/documentEntry/components/FieldTable.tsx`: before → after display from `original_value`, an "edited" badge when `is_manually_edited`, confidence kept and shown alongside the badge
- [ ] T044 [US5] Wire `FieldTable.tsx`'s save action to `POST /extraction-records/{id}/correct/` from `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`

**Checkpoint**: Correction works on any record status with a visible audit trail.

---

## Phase 8: User Story 6 - Re-send after correction (Priority: P2)

**Goal**: A corrected, already-sent record can be re-sent subject to the route's resend policy; Block refuses re-send of an already-successful delivery.

**Independent Test**: Correct a value on a `sent` record whose route allows resend, re-send, verify a new delivery attempt with the corrected payload; repeat on a route that blocks resend and verify refusal.

### Tests for User Story 6

- [ ] T045 [P] [US6] Test `resend_policy=block` refuses `POST /route-deliveries/{id}/send/` for an already successfully delivered record (was previously the `prevent_duplicates` + no-override 409 case; now unconditional) in `backend/django/tests/test_apps/test_document_entries/test_route_delivery_viewset.py`
- [ ] T046 [P] [US6] Test a re-send after correction carries the corrected payload and creates a new `DeliveryAttempt`, in `backend/django/tests/test_apps/test_document_entries/test_route_delivery_viewset.py`
- [ ] T047 [P] [US6] Migration test: `delivery_mode` removal migration sets `review_config.all_records=True` for document types owning a `manual` route, in `backend/django/tests/test_apps/test_document_entries/test_migrations.py`

### Implementation for User Story 6

- [ ] T048 [US6] Move the resend-eligibility guard in `_check_eligibility` (`backend/django/apps/document_entries/route_delivery_view_set.py`) from payload-hash comparison to `resend_policy == ResendPolicy.BLOCK and delivery already succeeded` (`last_successful_payload_hash is not None` or status has ever been `SENT`)
- [ ] T049 [US6] Add a data migration in `backend/django/apps/document_entries/migrations/` that sets `review_config.all_records = True` on every `DocumentType` owning an `OutputRoute` with `delivery_mode = manual`, then drop the `delivery_mode` column and the now-unused `DeliveryMode` enum from `models.py`/`enums.py` — **user-confirmed migrate**; remove the `MANUAL`/`PENDING_APPROVAL` parking branch from `initialize_and_dispatch` in `backend/django/apps/document_entries/services.py` (keep `PENDING_APPROVAL` in `RouteDeliveryStatus` for historical rows)
- [ ] T050 [US6] Update `OutputRouteForm.tsx` in `client/src/app/documentEntry/components/`: rename the `repeat_policy` field/labels to **Resend policy** (Allow/Block) and remove the **Delivery Mode** field entirely
- [ ] T051 [US6] Update all `repeat_policy`/`delivery_mode` references in `client/src/app/documentEntry/components/DeliverySection.tsx` and any shared `types.ts`/`utils.ts` in `client/src/app/documentEntry/records/` to the renamed field and dropped column

**Checkpoint**: Re-send + delivery-setting changes work; a manual-mode document type behaves identically via the review-all rule.

---

## Phase 9: User Story 7 - Platform-wide audit section (Priority: P2)

**Goal**: A new Audit section under the Organisation sidebar area shows who did what, when, before → after, action type, and originally-flagged, filterable by subsystem.

**Independent Test**: Perform each review action, open Audit, filter to Extraction Record modification, verify all fields render correctly.

### Tests for User Story 7

- [ ] T052 [P] [US7] Test the `extraction_records` subsystem maps `LogEntry` rows for `ExtractionRecord` + `ExtractedFieldValue` to `{actor, timestamp, action, changes, originally_flagged}` correctly for confirm/correct/reject/select_for_review/resend/status_change, in new `backend/django/tests/test_apps/test_audit/test_audit_view_set.py`
- [ ] T053 [P] [US7] Test org isolation: a user cannot see `LogEntry` rows for another organisation's records through `GET /api/v1/audit/`, in `backend/django/tests/test_apps/test_audit/test_audit_view_set.py`

### Implementation for User Story 7

- [ ] T054 [US7] Add a subsystem registry in `backend/django/apps/audit/subsystems.py`: a mapping of subsystem key → `{content_types: [...], org_scope_queryset_fn, action_mapper_fn}`, shipping one entry, `extraction_records`
- [ ] T055 [US7] Implement the `extraction_records` subsystem's org-scoping subquery (LogEntry has no org FK — filter `(content_type, object_pk) ∈` an org-scoped `ExtractionRecord`/`ExtractedFieldValue` subquery) and action mapper (`confirm | correct | reject | select_for_review | resend | status_change`, `originally_flagged` derived from a non-empty `needs_review_reason`) in `backend/django/apps/audit/subsystems.py`
- [ ] T056 [US7] Add `AuditLogEntrySerializer` in `backend/django/apps/audit/serializers.py` producing the shape from `contracts/api.md` (`actor`, `timestamp`, `subsystem`, `action`, `object`, `changes[]`, `originally_flagged`, `flag_reason`)
- [ ] T057 [US7] Add a read-only `AuditViewSet` (`GET /api/v1/audit/`) in `backend/django/apps/audit/views.py` with `subsystem` (required), `date_from`, `date_to`, `actor` query params and pagination, registered in `backend/django/apps/audit/urls.py`
- [ ] T058 [US7] Add `client/src/app/audit/AuditPage.tsx`: subsystem filter (only Extraction Record modification for now), actor/date filters, a table rendering actor/time/action/before→after/originally-flagged
- [ ] T059 [US7] Add an **Audit** entry under the Organisation area in `client/src/app/layout/DefaultNav.tsx`, routed to `AuditPage`

**Checkpoint**: The audit surface is live and org-isolated for the shipped subsystem.

---

## Phase 10: User Story 8 - Signal low confidence in the record UI (Priority: P3)

**Goal**: Record detail UI highlights values below the applied threshold, shows the threshold, and recommends an action.

**Independent Test**: Open a flagged record and verify low-confidence values are highlighted with the applied threshold and a recommendation shown.

### Tests for User Story 8

- [ ] T060 [P] [US8] Test the detail serializer's `applied_threshold`/per-value `below_threshold` fields reflect the document type's `review_config` correctly, in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`

### Implementation for User Story 8

- [ ] T061 [US8] Add `applied_threshold` (echoed from `review_config`) and per-value `below_threshold: bool` to `ExtractionRecordDetailSerializer` in `backend/django/apps/document_entries/serializers.py`
- [ ] T062 [US8] Use the server-provided `below_threshold`/`applied_threshold` for highlighting and a recommendation banner in `client/src/app/documentEntry/components/FieldTable.tsx` and `client/src/app/documentEntry/components/ConfidenceBar.tsx` (replacing any client-side threshold fallback from the VWE-1460 work, so the rule lives in one place)

**Checkpoint**: Highlighting is server-driven and consistent with the flagging rules.

---

## Phase 11: User Story 9 - Reviewer assignment with a dedicated Reviewer role (Priority: P3, dependent story)

**Goal**: Admins assign reviewers per document type; assignment grants a Reviewer role; the queue scopes to assignments with an admins/managers fallback; a hard 5-minute lock prevents colliding edits.

**Independent Test**: Assign a reviewer to a document type, produce a Needs-Review record for it, verify the reviewer sees it while a non-assigned member doesn't; verify the lock blocks a second reviewer until release/timeout.

### Tests for User Story 9

- [ ] T063 [P] [US9] Test `ReviewerAssignment` creation grants the seeded Reviewer `RoleAssignment`, and removing the last assignment revokes it, in `backend/django/tests/test_apps/test_document_entries/test_reviewer_assignment.py` (new file)
- [ ] T064 [P] [US9] Test `records_for_reviewer(user)`: assigned reviewer sees only their document types' `needs_review` records; workspace admin/manager sees zero-assignment document types' records too; org isolation, in `backend/django/tests/test_apps/test_document_entries/test_reviewer_assignment.py`
- [ ] T065 [P] [US9] Test `GET/PUT /document-types/{id}/reviewers/`: PUT replaces the set, non-member user → 400, requires `document_types:edit`, in `backend/django/tests/test_apps/test_document_entries/test_document_type_viewset.py`
- [ ] T066 [P] [US9] Test `ExtractionRecordLock`: claim/contest (409 with holder)/release/expiry (5 min); expiry unblocks `correct`/`update_status`; removing a reviewer's assignment releases their active locks, in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`

### Implementation for User Story 9

- [ ] T067 [P] [US9] Add `ReviewerAssignment` model (`organisation` FK, `document_type` FK `related_name="reviewer_assignments"`, `user` FK, `created_by` FK, `UniqueConstraint(document_type, user)`, auditlog-registered) in `backend/django/apps/document_entries/models.py`
- [ ] T068 Run `docker compose exec django python manage.py makemigrations document_entries` for T067 — **user-confirmed migrate**
- [ ] T069 [US9] Add a data migration/fixture seeding an org-scoped `Role(identifier="reviewer", permissions=[DOCUMENT_ENTRIES_REVIEW])` per organisation (or a signal on organisation creation, matching how other seeded roles are created) — **user-confirmed migrate**
- [ ] T070 [US9] Add `save`/`delete` signal handling in `backend/django/apps/document_entries/signals.py` (or a service method) so a user's first `ReviewerAssignment` grants the Reviewer `RoleAssignment` and their last removal revokes it
- [ ] T071 [US9] Implement `records_for_reviewer(user)` and `can_review(user, record)` in `backend/django/apps/document_entries/services.py`: assigned document types' `needs_review` records, plus (for users with `document_types:edit` acting as admin/manager) document types with zero `ReviewerAssignment` rows
- [ ] T072 [US9] Wire `records_for_reviewer` into the `review_queue` filter from T031 in `backend/django/apps/document_entries/filters.py`, replacing the Phase 6 placeholder scoping
- [ ] T073 [US9] Add `ReviewerAssignmentSerializer` (`GET`/`PUT` `{"reviewers": [...]}` / `{"user_ids": [...]}`) validating org/workspace membership, in `backend/django/apps/document_entries/serializers.py`
- [ ] T074 [US9] Add `reviewers` `@action` (`detail=True`, `methods=["get", "put"]`, requires `document_types:edit`) to `DocumentTypeViewSet` in `backend/django/apps/document_entries/document_type_view_set.py`
- [ ] T075 [US9] Add `ExtractionRecordLockSerializer` in `backend/django/apps/document_entries/serializers.py` and a `claim`/`unclaim` `@action` pair (`detail=True`, `methods=["post", "delete"]`, `url_path="claim"`) on `ExtractionRecordViewSet` in `backend/django/apps/document_entries/extraction_record_view_set.py`, mirroring `apps/locks/base/views.py`'s acquire/release pattern with a 5-minute `expires_at`
- [ ] T076 [US9] Return a `lock` object (`null` when free) on `ExtractionRecordDetailSerializer`, and enforce 409 in `correct` (T041) and `update_status` when another user holds an unexpired lock, in `backend/django/apps/document_entries/extraction_record_view_set.py`
- [ ] T077 [US9] Add a reviewer multi-select (existing org/workspace members) to `client/src/app/documentEntry/documentTypes/steps/ReviewRecordsStep.tsx`, calling `GET/PUT /document-types/{id}/reviewers/`
- [ ] T078 [US9] Add a lock banner ("being reviewed by X") to `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`, claiming the lock on entering edit mode and releasing on exit/unmount

**Checkpoint**: Reviewer assignment, scoped queue, role grant/revoke, and locking all work together.

---

## Phase 12: Polish & Cross-Cutting Concerns

**Purpose**: Final validation across all stories.

- [ ] T079 [P] Add any remaining new i18n keys to `client/src/translations/en.json` only (secondary locale files are translated later in bulk — leave untouched)
- [ ] T080 Run the full behaviour suite: `docker compose exec django pytest tests/test_apps/test_document_entries/ tests/test_apps/test_audit/ -x -q` from `backend/`
- [ ] T081 Walk through [quickstart.md](./quickstart.md)'s "Exercise the feature" steps end-to-end against a running stack

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies
- **Foundational (Phase 2)**: depends on Setup; BLOCKS all user stories; contains migrations requiring user confirmation before running
- **User Stories (Phase 3–11)**: all depend on Foundational; can proceed in spec priority order or in parallel per story once Foundational is migrated
- **Polish (Phase 12)**: depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Foundational only
- **US2 (P1)**: reads `review_config` → depends on US1's config surface existing (T003/T014), though flagging logic (T021–T023) can be built and unit-tested against the model directly without waiting for the wizard UI
- **US3 (P1)**: depends on US2 (needs a flagged status to hold)
- **US4 (P2)**: depends on US2 (needs `needs_review` records to queue) and Phase 2's transition additions (T011)
- **US5 (P2)**: independent after Foundational (T010's auditlog registration)
- **US6 (P2)**: independent after Foundational (T004/T005's resend_policy rename); the delivery-mode migration (T049) needs US1's `review_config` (T003) to exist as the migration target
- **US7 (P2)**: depends on US5 (needs corrections to audit) and US3/US4 (needs review actions to audit)
- **US8 (P3)**: depends on US1 (review_config) and US2 (flagging) for the threshold to surface
- **US9 (P3)**: depends on US4 (scopes the queue built there)

### Parallel Opportunities

- All `[P]`-marked tasks within a phase touch different files and can run concurrently
- After Phase 2 is migrated, US1, US5, and US6 can be staffed in parallel; US2/US3/US4 form a priority-ordered chain; US7 and US9 tail their dependencies

---

## Implementation Strategy

### MVP First

1. Phase 1 (Setup) + Phase 2 (Foundational) — confirm and run migrations
2. Phase 3 (US1) → Phase 4 (US2) → Phase 5 (US3) — the P1 chain: configure, flag, hold. **This alone delivers the core promise (SC-001, SC-002).**
3. Stop and validate independently before continuing

### Incremental Delivery

4. Phase 6 (US4, queue) → Phase 7 (US5, correction) → Phase 8 (US6, re-send + delivery settings) → Phase 9 (US7, audit)
5. Phase 10 (US8, signalling) and Phase 11 (US9, reviewer assignment) close out the P3 stories
6. Phase 12 (Polish) validates the whole feature against quickstart.md

## Notes

- `[P]` tasks touch different files with no dependency on an incomplete task
- Every migration task is flagged **user-confirmed migrate** — never run `migrate` without asking first
- No frontend tests are included per project convention
- Avoid: vague tasks, same-file conflicts within a `[P]` group, cross-story dependencies that break independent testability
