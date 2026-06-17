# Research: Document Entry — View Records

**Branch**: `003-de-view-records` | **Date**: 2026-06-16

---

## Key Finding: Feature Is Substantially Already Implemented

The `ExtractionRecord` list and detail pages, viewset, serializers, access policy, sidebar navigation, and routes all exist. This plan covers only the **delta** between the current implementation and the spec requirements.

---

## Decision: Backend Filtering Architecture

**Question**: How to add processing_status, delivery_status, confidence_threshold, and date_range filters to `ExtractionRecordViewSet`?

**Decision**: Adopt `django-filter` via the existing `ExtractionRecordFilterSet`, plus DRF's `SearchFilter` and `OrderingFilter` — the same backend stack as `DocumentTypeViewSet` in the same app. The current manual `query_params` extraction for `document_type` / `history` is folded into the filterset.

**Rationale**: `django-filter` is already a project dependency (`25.1`), already used by sibling viewsets (`DocumentTypeViewSet`, `OutputRouteViewSet`, `ApiEndpointViewSet`), and `ExtractionRecordFilterSet` already exists. Declarative filters keep this viewset consistent with the rest of the app and let field types handle validation. (Updated after PR review — the earlier "manual extraction / YAGNI" stance diverged from established convention.)

**Alternatives considered**: Manual `request.query_params.get(...)` extraction — rejected: diverges from the codebase's established `DjangoFilterBackend` pattern and duplicates validation the filterset gives for free.

---

## Decision: Search by Sender / Subject

**Question**: How to implement keyword search across email_sender and email_subject fields on `ExtractionRecord`?

**Decision**: Use Django's `Q` object with `__icontains` on both fields combined with OR. Accept a single `search` query param. Apply when `search` param is non-empty.

**Rationale**: Both fields are `CharField` on the model — standard icontains search is idiomatic and already indexed via `(organisation, created_at)`. Full-text search (PostgreSQL `search_vector`) is not warranted for V1 given the field types and expected volume.

**Alternatives considered**: PostgreSQL `SearchVector` — more accurate relevance ranking but adds complexity and a migration. Deferred.

---

## Decision: Ordering / Sorting

**Question**: How to make the list sortable by date rather than hard-coded `-id`?

**Decision**: Change the default ordering in `get_queryset()` from `-id` to `-email_received_at`. Accept an optional `ordering` query param for ascending (`email_received_at`) / descending (`-email_received_at`). Enable the date column sort on the frontend table.

**Rationale**: The spec requires sorting by date & time. The `email_received_at` field is the semantically correct field (not `created_at` which reflects DB insertion time, not email receipt time). The `(organisation, created_at)` index is close; `email_received_at` is a VARCHAR in the model — check if an index is warranted based on query plan.

---

## Decision: Confidence Threshold Filter

**Question**: `overall_confidence` is not a stored field — it is computed by the backend. How to filter by confidence threshold?

**Decision**: Investigate whether `overall_confidence` is stored on `ExtractionRecord` or computed in the serializer. If stored: filter via `overall_confidence__lte=threshold` in queryset. If computed in serializer: either (a) store it on the model and filter there, or (b) filter post-serialization (inefficient — avoid). Prefer stored approach.

**Resolution**: Research agent confirmed `overall_confidence` appears on `ExtractionRecordListSerializer` — verify if it is a model field or `SerializerMethodField`. If it is a `SerializerMethodField`, a migration to store it as a `FloatField` (populated at extraction time by the side effect handler) is needed for efficient filtering. If it is already a model field, no migration required.

**Action for Phase 1**: Read `ExtractionRecord` model definition to confirm field type; document finding in data-model.md.

---

## Decision: Extraction Sources Panel (Detail View)

**Question**: Can attachment files be retrieved from an `ExtractionRecord` for inline display?

**Answer: YES — GCS paths are persisted. ✅ Resolved.**

**Findings** (from codebase investigation):
- When an email arrives, attachments are uploaded to GCS and `gcs_path` is stored on each attachment entry in `History.input["attachments"]` (note: `History.input` IS the email dict directly — not wrapped under an `"email"` key).
- `ExtractionRecord` has a FK to `History`, which is already `select_related()` in the retrieve queryset.
- Email body is at `History.input["content"]` — the same field rendered by the history detail view.

**Decision** (revised after PR review — see [tp_vwe1460_view_records.md §3.3–§3.4](../../technical_proposals/tp_vwe1460_view_records.md)):

> ⚠️ The original decision (signed URLs via `default_storage.url()`, `download_url` in response) was superseded. The agreed design:

- `ExtractionRecordDetailSerializer` adds two `SerializerMethodField`s: `email_body` (reads `History.input["content"]`) and `attachments` (reads `History.input["attachments"]`, returns `[{filename, mime_type, failed}]` — **no `download_url`**).
- A new `attachment_download` `@action` on `ExtractionRecordViewSet` streams attachment bytes via `default_storage.open()` (authenticated GCS service-account client — `.url()` never called; no GCS URL exposed to the client).
- `default_storage.url(gcs_path)` is **not used** in this feature — it would generate a signed URL, which is undesirable (CSP whitelist required, signed-URL expiry problem for long-open previews).

**Implication for FR-009/FR-010**: Full inline viewer + download is feasible. Requires the serializer fields (tasks T001–T002) and the streaming proxy endpoint (task T003), plus the frontend `ExtractionSourcePanel` (tasks T006–T008).

---

## Decision: Low-Confidence Value Highlighting

**Question**: The detail page shows confidence bars but does not color-code values below threshold. What threshold should be used?

**Decision**: Use `DocumentType.global_confidence_threshold` (confirmed on the model) as the threshold for highlighting. Per-field `DocumentTypeField.confidence_threshold` takes precedence when set. Both thresholds are already returned in the detail serializer's `document_type` nested object and in the field definitions. The frontend already applies red borders to missing critical fields — extend the same pattern with an amber/orange highlight for low-confidence values.

**Rationale**: Thresholds are already configured and returned in the API response. No backend changes needed — this is a pure frontend enhancement.

---

## Current Implementation Status (confirmed by codebase research)

| Area | Status | Notes |
|------|--------|-------|
| `ExtractionRecord` model | ✅ Complete | All fields present; `overall_confidence` confirmed as stored field |
| `ExtractionRecordViewSet` | ⚠️ Partial | Missing filters: processing_status, delivery_status, confidence threshold, date range, search, sort |
| `ExtractionRecordListSerializer` | ✅ Complete | All spec columns present |
| `ExtractionRecordDetailSerializer` | ✅ Complete | trace_id, history link, field values, delivery status all present |
| List page — columns | ✅ Complete | All 7 spec columns shown |
| List page — filtering/search/sort | ❌ Missing | No filter panel, no search, no sort |
| Detail page — email metadata | ✅ Complete | Subject, sender, date, type, trace ID, history link |
| Detail page — field values + critical highlight | ✅ Complete | Red border on missing critical fields |
| Detail page — extraction sources panel | ❌ Missing | GCS paths persisted in History.input; need serializer field + frontend panel |
| Detail page — side-by-side comparison | ❌ Missing | Feasible once attachment URLs exposed via serializer |
| Detail page — low-confidence highlighting | ⚠️ Partial | Bars shown; no color coding below threshold |
| Detail page — failed attachment notice | ❌ Missing | `failed_attachment_names` not rendered in UI |
| Routes & navigation | ✅ Complete | `/document-entries/records` + `:id` routes exist; sidebar entry exists |
| Feature gating | ✅ Complete | `DocumentEntryFeatureGuard` wraps all routes |
| Access policy | ✅ Complete | `DocumentEntryRecordAccessPolicy` |
