# Tasks: Document Entry — View Records

**Feature**: VWE-1460 | **Branch**: `003-de-view-records`
**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Proposal**: [tp_vwe1460_view_records.md](../../technical_proposals/tp_vwe1460_view_records.md) | **Contract**: [contracts/records-api-delta.md](./contracts/records-api-delta.md)

> **Delta feature**: The `ExtractionRecord` list page, detail page, viewset, serializers, access policy, sidebar navigation, and routes all exist. Tasks below cover only the gaps confirmed in [research.md](./research.md) and defined in the technical proposal. User Story 1 (Browse Records) is already implemented — verified in Phase 0 spike.

---

## Table of Contents

- [Format](#format-id-p-story-description)
- [Path Conventions](#path-conventions)
- [Phase 1: US3 — Record Detail Backend (P1)](#phase-1-us3--record-detail-backend-p1)
- [Phase 2: US3 — Record Detail Frontend (P1)](#phase-2-us3--record-detail-frontend-p1)
- [Phase 3: US2 — Filter, Search, Sort Backend (P2)](#phase-3-us2--filter-search-sort-backend-p2)
- [Phase 4: US2 — Filter, Search, Sort Frontend (P2)](#phase-4-us2--filter-search-sort-frontend-p2)
- [Phase 5: Polish & Cross-Cutting](#phase-5-polish--cross-cutting)
- [Dependencies & Execution Order](#dependencies--execution-order)
- [Implementation Strategy](#implementation-strategy)
- [Notes](#notes)

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no incomplete-task dependency)
- **[Story]**: User story this task belongs to — [US2] or [US3]
- Include exact file paths in every description

## Path Conventions

- **Backend viewset**: `backend/django/apps/document_entries/extraction_record_view_set.py`
- **Backend serializers**: `backend/django/apps/document_entries/serializers.py`
- **Backend filterset**: `backend/django/apps/document_entries/filters.py`
- **Backend tests**: `backend/django/apps/document_entries/tests/`
- **Frontend records**: `client/src/app/documentEntry/records/`
- **i18n**: `client/src/locales/en.json`

---

## Phase 1: US3 — Record Detail Backend (P1)

**Goal**: Expose `email_body` and `attachments` metadata on the detail serializer; add the same-origin streaming proxy that serves attachment bytes via the authenticated GCS client — no signed URL, no GCS URL ever sent to the client.

**Independent Test**: `GET /api/v1/document-entries/records/{id}/` returns `email_body` (string or null) and `attachments` ([{filename, mime_type, failed}]). `GET /api/v1/document-entries/records/{id}/attachments/0/download/` streams bytes with `Content-Disposition: inline`. Unauthenticated or wrong-org request returns 403/404. Failed-attachment index returns 404. `default_storage.url` is never called.

### Implementation

- [ ] T001 [US3] Add `email_body` as `SerializerMethodField` on `ExtractionRecordDetailSerializer` — reads `obj.history.input.get("content")` (note: `History.input` IS the email dict directly — not wrapped under an `"email"` key); returns `null` when `history` is `None` or key absent — in `backend/django/apps/document_entries/serializers.py`
- [ ] T002 [US3] Add `attachments` as `SerializerMethodField` on `ExtractionRecordDetailSerializer` — reads `obj.history.input.get("attachments", [])`, returns `[{filename, mime_type, failed}]` (no `download_url`); marks `failed: true` when filename in `obj.failed_attachment_names`; chain-nested attachments excluded; returns `[]` when `history` is `None` — in `backend/django/apps/document_entries/serializers.py`
- [ ] T003 [US3] Add `attachment_download` `@action(detail=True, url_path="attachments/<int:index>/download")` to `ExtractionRecordViewSet`: fetch record via `get_object()` (enforces `ExtractionRecordAccessPolicy` — org-scoping applied here); resolve `gcs_path` from `record.history.input["attachments"][index]`; raise `Http404` for out-of-range index, missing/invalid `gcs_path`, or filename in `failed_attachment_names`; open bytes with `default_storage.open(gcs_path, "rb")` (authenticated GCS service-account client — `.url()` never called); wrap in `try/except Exception` calling `logger.exception()` before re-raising; return `StreamingHttpResponse(file, content_type=mime_type)` with `Content-Disposition: inline; filename="{filename}"` — in `backend/django/apps/document_entries/extraction_record_view_set.py`

### Tests

- [ ] T004 [P] [US3] Write serializer tests in `backend/django/apps/document_entries/tests/test_extraction_record_serializer.py`: `test_email_body_returns_content_from_history_input`, `test_email_body_null_when_history_is_null`, `test_attachments_field_returns_metadata_list` (two attachments → `{filename, mime_type, failed}`, no `download_url`), `test_attachments_field_marks_failed_attachment`, `test_attachments_field_when_history_is_null`, `test_attachments_field_when_history_has_no_attachments`
- [ ] T005 [P] [US3] Write proxy endpoint tests in `backend/django/apps/document_entries/tests/test_extraction_record_view_set.py`: `test_attachment_download_streams_file` (200, streamed bytes, correct headers; mock `default_storage.open`), `test_attachment_download_out_of_range_index_returns_404`, `test_attachment_download_failed_attachment_returns_404`, `test_attachment_download_missing_gcs_path_returns_404`, `test_attachment_download_no_history_returns_404`, `test_attachment_download_enforces_access_policy` (unauthenticated / wrong-org → 403/404), `test_attachment_download_streams_not_redirects` (response body is file bytes; assert `default_storage.url` never called)

**Checkpoint**: Detail serializer returns `email_body` + `attachments` metadata; proxy streams bytes; org-scoping enforced; all tests pass in Docker.

---

## Phase 2: US3 — Record Detail Frontend (P1)

**Goal**: Render a two-column detail layout — source panel on the left (email body + attachment previews), extracted field values on the right. Highlight low-confidence values amber. Show "file unavailable" for failed attachments.

**Independent Test**: Open a record detail — source panel appears with `extraction_source` badge, sanitised email body, and per-attachment tiles. A PDF attachment shows an inline `<iframe>` whose `src` is the same-origin proxy URL. A non-PDF shows a download link to the same URL. A failed attachment shows "file unavailable" badge with preview/download disabled. A field whose `confidence < threshold` shows amber highlight; a field with a per-field `confidence_threshold` uses that, not the global threshold. FR-011 confidence display and FR-012 critical-field red border remain visible in the refactored layout.

### Implementation

- [ ] T006 [US3] Create `ExtractionSourcePanel.tsx` — renders `extraction_source` mode badge (`body_only` / `attachments_only` / `both`); if source includes body, renders `email_body` HTML via `dangerouslySetInnerHTML` + `DOMPurify.sanitize()` (same pattern as `ApplicationHistoryEmail.tsx:256`); renders one `AttachmentPreview` per entry in `attachments` array — in `client/src/app/documentEntry/records/components/ExtractionSourcePanel.tsx`
- [ ] T007 [US3] Add `AttachmentPreview` sub-component inside `ExtractionSourcePanel.tsx` — receives `{ recordId, attachmentIndex, filename, mimeType, failed }` props; sub-component builds proxy URL internally: `/api/v1/document-entries/records/{recordId}/attachments/{attachmentIndex}/download/`; renders `<iframe src={proxyUrl}>` for PDFs; download link (same `proxyUrl`) for non-PDF types; "File unavailable" badge (no URL rendered, no endpoint call) when `failed: true` — in `client/src/app/documentEntry/records/components/ExtractionSourcePanel.tsx`
- [ ] T008 [US3] Update `ExtractionRecordDetailPage.tsx` — adopt two-column layout (`Grid` or `SimpleGrid`); render `ExtractionSourcePanel` on the left (pass `recordId`, `email_body`, `extraction_source`, `attachments` from detail query result); keep existing extracted field values section on the right; stack vertically on narrow viewports; **verify FR-011 confidence display and FR-012 critical-field red border still render in right column after layout refactor** — in `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`
- [ ] T009 [US3] Add low-confidence highlighting in field value rendering — apply amber/warning colour to confidence bar and field row when `value.confidence < threshold`, where `threshold = field.confidence_threshold ?? record.document_type.global_confidence_threshold`; both values already returned by `ExtractionRecordDetailSerializer` — in `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx` and/or `FieldTable` / `RenderGroupList` child components
- [ ] T010 [P] [US3] Add i18n keys for detail page to `client/src/locales/en.json`: `documentEntry.records.detail.extractionSource.label`, `documentEntry.records.detail.extractionSource.emailBody`, `documentEntry.records.detail.extractionSource.fileUnavailable`, `documentEntry.records.detail.extractionSource.download`, `documentEntry.records.detail.lowConfidence`

### Tests (Vitest)

- [ ] T011 [P] [US3] Write Vitest tests for `AttachmentPreview` and `ExtractionSourcePanel` in `client/src/app/documentEntry/records/components/ExtractionSourcePanel.test.tsx`: `AttachmentPreview` renders `<iframe>` with correct proxy `src` for PDFs; renders download link (not iframe) for non-PDF; renders "file unavailable" badge (no `src`, no link) when `failed: true`; `ExtractionSourcePanel` renders sanitised `email_body` when `extraction_source` is `body_only` or `both`; renders correct `extraction_source` mode badge
- [ ] T012 [P] [US3] Write Vitest tests for low-confidence highlighting in `client/src/app/documentEntry/records/ExtractionRecordDetailPage.test.tsx`: field with `confidence < global_confidence_threshold` receives warning colour; field with per-field `confidence_threshold` uses that override, not the global; field at or above threshold has no warning colour

**Checkpoint**: Detail page renders source panel side-by-side with extracted values; PDFs preview inline via same-origin proxy; low-confidence values highlighted; FR-011 and FR-012 still render correctly.

---

## Phase 3: US2 — Filter, Search, Sort Backend (P2)

**Goal**: Extend the existing `ExtractionRecordFilterSet` and wire `DjangoFilterBackend + SearchFilter + OrderingFilter` on `ExtractionRecordViewSet`, replacing the current manual `query_params.get()` approach. Consistent with `DocumentTypeViewSet` in the same app.

**Independent Test**: `GET /records/?status=needs_review` returns only matching records. `GET /records/?delivery_status=pending` includes records with no `RouteDelivery` rows. `GET /records/?confidence_max=0.8` excludes records with null confidence. `GET /records/?search=acme` matches sender or subject case-insensitively. `GET /records/?ordering=email_received_at` returns ascending. Unknown filter values / reversed date range return empty results, not 400. `GET /records/?history={hashid}` still works (moved to filterset).

### Implementation

- [ ] T013 [US2] Extend `ExtractionRecordFilterSet` in `backend/django/apps/document_entries/filters.py` — add: `history = HashidFilter(field_name="history_id")` (moves the existing manual param into the filterset); `status` as `MultipleChoiceFilter` (choices from `ExtractionRecordStatus`, comma-separated OR logic); `delivery_status` as a `MethodFilter` that applies `route_deliveries__status__in=[...]` + `.distinct()` with an OR branch for `pending` (ORs in `route_deliveries__isnull=True` when `pending` is among the requested values); `confidence_max` as `NumberFilter(field_name="overall_confidence", lookup_expr="lte")`; `date_from` as `DateFilter(field_name="email_received_at", lookup_expr="date__gte")`; `date_to` as `DateFilter(field_name="email_received_at", lookup_expr="date__lte")`
- [ ] T014 [US2] Update `ExtractionRecordViewSet` in `backend/django/apps/document_entries/extraction_record_view_set.py` — set `filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]`, `filterset_class = ExtractionRecordFilterSet`, `search_fields = ["email_sender", "email_subject"]`, `ordering_fields = ["email_received_at"]`, `ordering = ["-email_received_at"]`; remove manual `query_params.get()` calls for `document_type` and `history` (both now handled by filterset)

### Tests

- [ ] T015 [P] [US2] Write filter tests in `backend/django/apps/document_entries/tests/test_extraction_record_view_set.py`: `test_filter_by_status_single`, `test_filter_by_status_multiple` (OR logic), `test_filter_by_confidence_max`, `test_filter_by_date_from`, `test_filter_by_date_to`, `test_filter_by_date_range`, `test_reversed_date_range_returns_empty`, `test_unknown_status_value_returns_empty`, `test_filter_by_history_param_still_works` (confirms history filter moved to filterset without regression)
- [ ] T016 [P] [US2] Write delivery status tests: `test_filter_by_delivery_status_pending_includes_no_delivery_records`, `test_filter_by_delivery_status_success`, `test_filter_by_delivery_status_multi_value` (OR) — in `backend/django/apps/document_entries/tests/test_extraction_record_view_set.py`
- [ ] T017 [P] [US2] Write search and ordering tests: `test_search_by_sender`, `test_search_by_subject`, `test_search_case_insensitive`, `test_ordering_asc`, `test_ordering_desc`, `test_default_ordering_is_desc_received_at` — in `backend/django/apps/document_entries/tests/test_extraction_record_view_set.py`

**Checkpoint**: All filter, search, and ordering params work per `contracts/records-api-delta.md`; manual `query_params.get()` removed; history filter works via filterset; all tests pass in Docker.

---

## Phase 4: US2 — Filter, Search, Sort Frontend (P2)

**Goal**: Add a filter panel above the records table, wire all filter/search/sort state to URL params, enable date column sorting, and add inline date range validation.

**Independent Test**: Apply "Needs review" status filter — only matching rows shown. Clear filter — all rows return. Set `date_from > date_to` — inline error shown, no API request fired. Search "acme@" — only matching rows. Sort date column descending — newest first. Refresh page — filter state persists from URL params.

### Implementation

- [ ] T018 [US2] Create `RecordFilters.tsx` — `MultiSelect` for processing status (all 7 `ExtractionRecordStatus` values), `MultiSelect` for delivery status (`pending`, `warning`, `success`, `error`), `NumberInput` 0–100% confidence threshold (maps to `confidence_max` param as `value / 100`), `DatePickerInput` range for date range (`date_from` / `date_to`), `TextInput` for keyword search (debounced), "Clear all" action; all filter state persisted in URL search params; resets pagination to page 1 on any change — in `client/src/app/documentEntry/records/components/RecordFilters.tsx`
- [ ] T019 [US2] Update `ExtractionRecordListPage.tsx` — render `RecordFilters` above table, wire all filter values to `usePaginatedApiQuery` params, enable date column sort (`enableSorting: true` on `email_received_at` column; map TanStack sort state → `ordering` param: `email_received_at` / `-email_received_at`); add inline date range validation: when `date_from > date_to`, show inline error and suppress API request — in `client/src/app/documentEntry/records/ExtractionRecordListPage.tsx`
- [ ] T020 [P] [US2] Add i18n keys to `client/src/locales/en.json`: `documentEntry.records.filters.status`, `documentEntry.records.filters.deliveryStatus`, `documentEntry.records.filters.confidence`, `documentEntry.records.filters.dateRange`, `documentEntry.records.filters.search`, `documentEntry.records.filters.clearAll`

### Tests (Vitest)

- [ ] T021 [P] [US2] Write Vitest tests for `RecordFilters` in `client/src/app/documentEntry/records/components/RecordFilters.test.tsx`: all controls render; each filter emits correct URL param on change; "Clear all" resets all URL params and resets pagination; `date_from > date_to` shows inline error and does not fire a request; confidence 80% maps to `confidence_max=0.8`
- [ ] T022 [P] [US2] Write Vitest tests for list page sort in `client/src/app/documentEntry/records/ExtractionRecordListPage.test.tsx`: sorting date column asc sends `ordering=email_received_at`; sorting desc sends `ordering=-email_received_at`; filter change resets pagination to page 1

**Checkpoint**: Filter panel renders; all 7 US2 acceptance scenarios work end-to-end; filter state persists in URL on page refresh.

---

## Phase 5: Polish & Cross-Cutting

- [ ] T023 Fill secondary language files with English values for all new i18n keys added in T010 and T020 — do not translate, fill with exact English string per project convention (check all files in `client/src/locales/` except `en.json`)
- [ ] T024 [P] Update `specs/003-de-view-records/data-model.md` — replace "TBD — see Spike 1" for `overall_confidence` with confirmed type `FloatField(null=True)`; note no migration required; note email_received_at is `DateTimeField` (not VARCHAR)
- [ ] T025 Run full backend tests via Docker and confirm no regressions: `docker compose exec django pytest apps/document_entries/ -v` from `backend/`; manually verify list page loads within 2 seconds with 1,000 records (SC-002)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (US3 backend)**: No dependencies — start immediately
- **Phase 2 (US3 frontend)**: Depends on Phase 1 (needs real API to test `email_body`, `attachments`, proxy URL)
- **Phase 3 (US2 backend)**: Independent of Phases 1–2 — can run in parallel with Phase 1 if team capacity allows
- **Phase 4 (US2 frontend)**: Depends on Phase 3 (filter params must exist in API)
- **Phase 5 (Polish)**: Depends on all phases complete

### Within Phase 1

- T001 and T002 (serializer fields) are independent — implement in parallel
- T004 and T005 (tests) can be written concurrently with T001–T003

### Within Phase 2

- T006/T007 (`ExtractionSourcePanel` + `AttachmentPreview`) → then T008 (wire into detail page)
- T009 (low-confidence) is independent of T006–T008 — can be written in parallel
- T010–T012 (i18n + tests) are independent of each other once T006–T009 complete

### Within Phase 3

- T013 (filterset) → then T014 (wire to viewset) — sequential
- T015, T016, T017 (tests) can run in parallel once T013–T014 are done

### User Story Dependencies

- **US3 (P1)**: No dependency on US2
- **US2 (P2)**: No dependency on US3 — prioritise US3 first due to P1 priority, but both can run in parallel with two developers

---

## Parallel Example: Two-Developer Split

```
Developer A (US3):              Developer B (US2):
Phase 1: T001, T002, T003 →    Phase 3: T013, T014 →
         T004, T005          →           T015, T016, T017
Phase 2: T006–T012           →  Phase 4: T018–T022
```

Shared files to coordinate on: `serializers.py` (T001, T002) and `extraction_record_view_set.py` (T003 vs T014). Agree on who edits first or use short-lived feature branches and merge.

---

## Implementation Strategy

### MVP: US3 First (highest user value — P1)

1. Complete Phase 1: US3 backend (T001–T005)
2. Complete Phase 2: US3 frontend (T006–T012)
3. **VALIDATE**: Open a record detail — source panel, PDF preview, low-confidence highlighting all work
4. Move to Phase 3 + 4: US2 filters

### Incremental Delivery

1. Phase 1 + 2 → US3 complete → demo/merge
2. Phase 3 + 4 → US2 complete → demo/merge
3. Phase 5 → Polish → final merge

---

## Notes

- **`[P]`** = different files, no incomplete-task dependency — safe to run in parallel
- **Backend tests**: run via Docker only — `docker compose exec django pytest` from `backend/`
- **Secondary language files**: fill new i18n keys with English value as-is (no translation)
- **`History.input` key path**: `History.input` IS the email dict directly (`input=email` in application_view_set.py) — NOT wrapped under `"email"` key. Attachments at `History.input["attachments"]`; body at `History.input["content"]`
- **Streaming proxy**: `default_storage.open()` uses authenticated GCS service-account client — `.url()` (signed-URL method) is never called; no GCS URL is ever sent to the client
- **`delivery_status` MethodFilter**: records with no `RouteDelivery` rows count as `pending` — OR branch: `route_deliveries__isnull=True` when `pending` is among the requested values
- **PDF inline preview**: `<iframe src={proxyUrl}>` (same-origin) — CSP-clean; `default-src 'self'` covers it with no `frame-src` change needed; no PDF.js
- **`AttachmentPreview`** owns its URL logic internally — parent passes only `{recordId, attachmentIndex, filename, mimeType, failed}`
- **`history` filter**: now in `ExtractionRecordFilterSet` (T013) — the manual `query_params.get("history")` in the viewset is removed in T014
