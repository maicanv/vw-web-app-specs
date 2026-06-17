# Implementation Plan: Document Entry — View Records

**Branch**: `003-de-view-records` | **Date**: 2026-06-16 | **Spec**: [spec.md](./spec.md)

> **Context**: The `ExtractionRecord` list and detail infrastructure (model, serializers, viewset, access policy, frontend pages, routes, and sidebar navigation) are already implemented. This plan covers **only the delta** — the gaps between the current implementation and spec requirements. See [research.md](./research.md) for full gap analysis.

---

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Phase 0: Research Spike](#phase-0-research-spike)
- [Phase 1: Backend Delta](#phase-1-backend-delta)
- [Phase 2: Frontend Delta](#phase-2-frontend-delta)
- [Complexity Tracking](#complexity-tracking)

---

## Summary

The Document Entry — View Records feature is substantially implemented. The remaining work is:

1. **Backend**: Add filtering (processing_status, delivery_status, confidence_threshold, date_range), keyword search (sender/subject), and flexible ordering to `ExtractionRecordViewSet`.
2. **Frontend list page**: Add filter panel, search bar, and enable date column sorting.
3. **Frontend detail page**: Add extraction source panel (with "file unavailable" handling), side-by-side source/extracted comparison (scoped to what's feasible — see Phase 0 spike), and low-confidence value highlighting.

The core data model, serializers, access policy, navigation, and routes require no changes.

---

## Technical Context

**Language/Version**: TypeScript 5 (frontend), Python 3.12 (backend)
**Primary Dependencies**: React 18, Mantine UI, TanStack Query (frontend); Django 4.2, DRF (backend)
**Storage**: PostgreSQL (existing `ExtractionRecord`, `ExtractedFieldValue`, `RouteDelivery` models)
**Testing**: Vitest (frontend), pytest via Docker (backend)
**Target Platform**: Web application (React SPA + Django REST API)
**Performance Goals**: List loads ≤ 2s for 1,000 records (SC-002); record located via filter/search ≤ 30s (SC-001)
**Constraints**: Read-only feature (no write operations beyond existing `update_status`); no new models required (confirm via Phase 0 spike)
**Scale/Scope**: Workspace-scoped records; no cross-workspace views

---

## Constitution Check

### I. Protect Sensitive Data ✅
- Records are scoped to the authenticated user's organisation via `OrgQuerySetMixin` (existing).
- Email content (subject, sender) is already stored on `ExtractionRecord` — not re-fetching raw email.
- No PII in logs; existing audit logging applies.

### II. Respect the Architecture ✅
- All changes extend existing patterns: viewset filter/search follows the `document_type` / `history` param pattern; frontend filter panel follows document-types list page pattern.
- No new service boundaries introduced.

### III. Test What Matters ✅
- Backend: test each new filter param, search, and ordering in `ExtractionRecordViewSet`.
- Frontend: test filter state management and that filtered requests reach the API with correct params.

### IV. Ship Incrementally ✅
- Delivery is the delta only; the existing working UI is not regressed.
- Each sub-feature (filter, search, sort, extraction panel, confidence highlight) is independently testable.

### V. Make Failures Visible ✅
- "File unavailable" state renders explicitly in the UI (FR-016).
- Backend filter errors return standard DRF 400 responses.

**Gate result**: PASS — no violations.

---

## Project Structure

### Documentation (this feature)

```text
specs/003-de-view-records/
├── plan.md              ← this file
├── research.md          ← Phase 0 output
├── data-model.md        ← Phase 1 output (confirms overall_confidence field type)
├── contracts/
│   └── records-api-delta.md   ← Phase 1 output (new query params)
└── tasks.md             ← Phase 2 output (/speckit-tasks command)
```

### Source Code (delta files only)

```text
backend/django/apps/document_entries/
└── extraction_record_view_set.py    ← add filters, search, ordering

client/src/app/documentEntry/records/
├── ExtractionRecordListPage.tsx     ← add filter panel, search, enable sort
├── ExtractionRecordDetailPage.tsx   ← add extraction source panel, confidence highlight
├── components/
│   ├── RecordFilters.tsx            ← new: filter panel component
│   └── ExtractionSourcePanel.tsx    ← new: source viewer component
└── types.ts                         ← extend if new fields needed
```

No migrations required unless Phase 0 spike reveals `overall_confidence` is a `SerializerMethodField` (in that case: one migration to promote it to a stored field).

---

## Phase 0: Research Spikes ✅ Both Resolved

**Spike 1 — `overall_confidence` storage**: Confirmed stored `FloatField` on `ExtractionRecord`. No migration needed — filter via `overall_confidence__lte` directly.

**Spike 2 — Attachment source URLs**: GCS paths are persisted in `History.input` JSON. `History` is already `select_related()` on the retrieve queryset. Full inline viewer + download is feasible via a same-origin streaming proxy.

> ⚠️ **Superseded** — the signed-URL approach originally noted here was revised after PR review. See [tp_vwe1460_view_records.md §3.3–§3.4](../../technical_proposals/tp_vwe1460_view_records.md) for the agreed design: `default_storage.open()` streams bytes directly via the authenticated GCS client; `.url()` (the signed-URL method) is never called; no GCS URL is exposed to the client.

---

## Phase 1: Backend Delta

### 1.1 — Data Model Confirmation (`data-model.md`)

Document the confirmed `ExtractionRecord` field types (from Spike 1). No design changes unless `overall_confidence` requires a migration.

**Key entities** (as-built, confirmed):

- **ExtractionRecord**: `organisation` FK, `application` FK, `document_type` FK (SET_NULL), `history` FK, `status` (7-value TextChoices), `email_received_at`, `email_subject`, `email_sender`, `overall_confidence` (confirm type), `needs_review_reason`, `metadata`, `extraction_source`, `failed_attachment_names`, `classification_rationale`, `failure_stage`, `failure_message`
- **ExtractedFieldValue**: `record` FK, `document_type_field` FK (SET_NULL), `field_codename`, `raw_value`, `display_value`, `confidence`, `is_critical`, `group_item_index`
- **RouteDelivery**: `extraction_record` FK, `output_route` FK (nullable), `status` (6-value with `.aggregate()` method)

### 1.2 — API Contract (`contracts/records-api-delta.md`)

**New query parameters** on `GET /api/v1/document-entries/records/`:

| Param | Type | Description |
|-------|------|-------------|
| `status` | string (comma-separated) | Filter by processing status — values: `extracted`, `needs_review`, `failed`, `sent`, `skipped`, `send_failed`, `rejected` |
| `delivery_status` | string (comma-separated) | Filter by delivery status — values: `pending`, `warning`, `success`, `error` |
| `confidence_max` | float [0–1] | Return only records with `overall_confidence` ≤ this value |
| `date_from` | ISO 8601 date | Filter `email_received_at` ≥ this date |
| `date_to` | ISO 8601 date | Filter `email_received_at` ≤ this date |
| `search` | string | Case-insensitive match against `email_sender` OR `email_subject` |
| `ordering` | string | `email_received_at` (asc) or `-email_received_at` (desc, default) |

**No response shape changes** — same `ExtractionRecordListSerializer` output.

### 1.3 — Serializer Change: Attachment Metadata + Streaming Proxy

> ⚠️ **Superseded** — this section originally described a signed-URL approach. The agreed design uses a same-origin streaming proxy instead. See [tp_vwe1460_view_records.md §3.3–§3.4](../../technical_proposals/tp_vwe1460_view_records.md) for the canonical implementation. Key differences:
> - `attachments` field returns `{filename, mime_type, failed}` only — **no `download_url`**
> - `History.input` IS the email dict directly; attachments are at `History.input["attachments"]` (not `["email"]["attachments"]`)
> - Bytes are served by a new `attachment_download` `@action` on `ExtractionRecordViewSet` via `default_storage.open()` (authenticated GCS client — `.url()` never called)
> - A separate `email_body` field reads `History.input["content"]`
>
> See tasks.md T001–T003 for implementation tasks and T004–T005 for the updated test list.

### 1.4 — ViewSet Changes

File: `backend/django/apps/document_entries/extraction_record_view_set.py`

```
get_queryset() changes:
  - Default ordering: change -id → -email_received_at
  - Add: status filter (iexact / __in on comma-split)
  - Add: delivery_status filter (requires joining RouteDelivery aggregate —
         see notes below)
  - Add: confidence_max filter (overall_confidence__lte if stored field)
  - Add: date_from / date_to filter (email_received_at__date__gte / __lte)
  - Add: search filter (Q(email_sender__icontains=q) | Q(email_subject__icontains=q))
  - Add: ordering param (whitelist: email_received_at, -email_received_at)
```

> ⚠️ **Superseded** — the manual `get_queryset()` approach above has been replaced by `DjangoFilterBackend + ExtractionRecordFilterSet` (same pattern as `DocumentTypeViewSet`). See [tp_vwe1460_view_records.md §3.1](../../technical_proposals/tp_vwe1460_view_records.md) and tasks.md T013–T014. Also: `email_received_at` is a `DateTimeField` (not VARCHAR as previously noted).

**Delivery status filter — decided approach**: `MethodFilter` on `ExtractionRecordFilterSet` using `route_deliveries__status__in=[...]` + `.distinct()`. When `pending` is among the requested values, OR in `route_deliveries__isnull=True` (records with no `RouteDelivery` rows count as pending). See tasks.md T013.

> ⚠️ **Superseded** — the test names above were placeholders. See [tp_vwe1460_view_records.md §5](../../technical_proposals/tp_vwe1460_view_records.md) for the canonical test list and tasks.md T015–T017 for the implementation tasks.

---

## Phase 2: Frontend Delta

### 2.1 — List Page: Filter Panel + Search + Sort

File: `client/src/app/documentEntry/records/ExtractionRecordListPage.tsx`
New: `client/src/app/documentEntry/records/components/RecordFilters.tsx`

**Changes**:
- Add `RecordFilters` component above the table: contains `Select` (multi) for status, `Select` (multi) for delivery status, `NumberInput` for confidence threshold (%), `DatePickerInput` range for date range, `TextInput` for search.
- Persist filter state in URL search params (consistent with existing `document_type` param pattern).
- Wire filter values to `usePaginatedApiQuery` params.
- Enable date column sorting: set `enableSorting: true` on `email_received_at` column; map sort state to `ordering` param.
- Reset pagination to page 1 when any filter changes.

**i18n keys to add** (in `en.json` and fill same value in secondary languages):
- `documentEntry.records.filters.status`
- `documentEntry.records.filters.deliveryStatus`
- `documentEntry.records.filters.confidence`
- `documentEntry.records.filters.dateRange`
- `documentEntry.records.filters.search`
- `documentEntry.records.filters.clearAll`

### 2.2 — Detail Page: Extraction Source Panel

File: `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`
New: `client/src/app/documentEntry/records/components/ExtractionSourcePanel.tsx`

> ⚠️ **Superseded** — the UI description below originally referenced `download_url` from the serializer. The agreed design uses a same-origin streaming proxy instead. See [tp_vwe1460_view_records.md §4.2](../../technical_proposals/tp_vwe1460_view_records.md) for the canonical frontend spec and tasks.md T006–T008 for implementation tasks.

**Attachment source URLs available (Spike 2 resolved).**

`ExtractionSourcePanel` receives `email_body`, `extraction_source`, and `attachments` (metadata only — no `download_url`) from the detail API and renders:
- Extraction mode badge (`body_only` / `attachments_only` / `both`) from `extraction_source`.
- Email body rendered via DOMPurify when `extraction_source` includes body.
- For each attachment: `AttachmentPreview` sub-component builds the proxy URL (`/records/{id}/attachments/{index}/download/`) internally; renders `<iframe>` for PDFs, download link for others.
- For each attachment where `failed: true`: "file unavailable" badge, no URL rendered (FR-016).

**Side-by-side layout**: two-column layout — source panel on the left, extracted field values on the right. On narrow viewports, stack vertically. This satisfies FR-010 (source and extracted values visible without leaving the page).

### 2.3 — Detail Page: Low-Confidence Highlighting

File: `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx` (and/or `FieldTable` / `RenderGroupList` components)

**Change**: In the field value rendering, apply an amber/warning color to the confidence bar and field row when `value.confidence < threshold`, where threshold is:
- `field.confidence_threshold` if set on the `DocumentTypeField` (per-field override).
- `document_type.global_confidence_threshold` otherwise.

Both values are already available in the detail API response. No backend change needed.

**i18n key**: `documentEntry.records.detail.lowConfidence` (tooltip label for highlighted fields).

---

## Complexity Tracking

No constitution violations. No complexity justifications required.

---

## Open Questions (for /speckit-tasks)

1. **Delivery status filter complexity** — filtering by aggregated delivery status requires a subquery or queryset annotation (it's not a column on `ExtractionRecord`). Use `route_deliveries__status__in` with `distinct()` as the V1 approach; handle the "no deliveries = pending" edge case explicitly (records with no `RouteDelivery` rows count as pending).
