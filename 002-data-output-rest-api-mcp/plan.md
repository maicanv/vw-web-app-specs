# Implementation Plan: Data Output via REST API

**Branch**: `002-data-output-rest-api-mcp` | **Date**: 2026-05-26 | **Spec**: [spec.md](./spec.md)

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Phase Overview](#phase-overview)
- [Phase 1 — Data Model & Migrations](#phase-1--data-model--migrations)
- [Phase 2 — Output Route API (Django)](#phase-2--output-route-api-django)
- [Phase 3 — Delivery Orchestration (Django + FastAPI)](#phase-3--delivery-orchestration-django--fastapi)
- [Phase 4 — Record Detail & History APIs](#phase-4--record-detail--history-apis)
- [Phase 5 — Frontend: Document Type Wizard (Output Step)](#phase-5--frontend-document-type-wizard-output-step)
- [Phase 6 — Frontend: Record Detail Delivery Section](#phase-6--frontend-record-detail-delivery-section)
- [Phase 7 — Frontend: History Read-Only Delivery Section](#phase-7--frontend-history-read-only-delivery-section)
- [Complexity Tracking](#complexity-tracking)

---

## Summary

Route structured extraction results from Document Types to external REST endpoints by linking each Document Type to one or more output routes. Each route owns a dedicated `ApiEndpoint` (created inline, no reuse), assembles a `general / email / data` JSON payload using the grouped extraction shape from VWE-1496, applies per-field value transforms, and delivers via HTTP. Auto delivery fires on any transition to a deliverable `ExtractionRecord` status; manual approval and retry are exposed through dedicated record-level endpoints. A SSRF guard, a 1 MB payload cap, and a per-attempt delivery log are mandatory. The frontend adds an Output wizard step and a Delivery section to both the record detail page and the history read-only view.

---

## Technical Context

**Language/Version**: Python 3.12 (Django/FastAPI), TypeScript 5.x (React)
**Primary Dependencies**: Django 4.2, DRF 3.17, FastAPI 0.136+, React 18, Mantine UI, TanStack Query v5, Vitest
**Storage**: PostgreSQL (psycopg2); outbound HTTP via `httpx` (already used in the backbone)
**Testing**: pytest (Django + FastAPI), Vitest + React Testing Library (frontend)
**Target Platform**: Linux container (Docker Compose / GCP); local dev via Docker or Poetry venv
**Project Type**: Web service (Django management API) + async backbone (FastAPI) + React SPA
**Performance Goals**: Auto delivery within 30 s of status transition; outbound request timeout 30 s
**Constraints**: Max outbound payload 1 MB; SSRF guard blocks private IPs/localhost/metadata; no automatic retries; secret headers never in logs
**Scale/Scope**: Per-record, per-route delivery; O(N routes × records) delivery attempts; no bulk delivery in v1

---

## Constitution Check

| Principle | Assessment | Notes |
|-----------|------------|-------|
| I. Protect Sensitive Data | ✅ Pass | Payload hash only in logs; secret headers masked via existing `api_integrations` layer; SSRF guard prevents SSRF attacks |
| II. Respect the Architecture | ✅ Pass | New models in `document_entries` app; reuses `api_integrations` connection/endpoint pattern; delivery follows existing `ProviderAction.execute()` hook points |
| III. Test What Matters | ✅ Pass | Unit tests for SSRF guard, payload builder, and delivery service; integration tests for delivery trigger on status change |
| IV. Ship Incrementally | ✅ Pass | 4 independent user stories; MCP/XML/name-mapping deferred to v2; data model is backward-safe (new tables, no column removals) |
| V. Make Failures Visible | ✅ Pass | Every delivery failure captured in `DeliveryAttempt`; Sentry for unexpected exceptions in delivery code |

No violations. Proceeding.

---

## Project Structure

### Documentation (this feature)

```text
specs/002-data-output-rest-api-mcp/
├── plan.md                              # This file
├── research.md                          # Phase 0 research findings
├── data-model.md                        # Phase 1 — models, migrations, enums
├── quickstart.md                        # Dev setup notes
├── contracts/
│   ├── output-routes-api.md             # Output route CRUD contract
│   └── record-detail-api.md             # Delivery section in record detail + delivery actions
├── checklists/
│   └── requirements.md                  # Spec quality checklist
└── tasks.md                             # Phase 2 output (/speckit-tasks — not yet created)
```

### Source Code Layout

```text
backend/django/apps/document_entries/
├── models.py                            # + OutputRoute, DeliveryAttempt
├── migrations/
│   └── 000N_add_output_routes.py        # New tables only; backward-safe
├── serializers.py                       # + OutputRouteSerializer, extend ExtractionRecordDetailSerializer
├── document_type_view_set.py            # + output-routes nested route
├── extraction_record_view_set.py        # + deliver/ action endpoint
├── services.py                          # + DeliveryService, PayloadBuilder
└── tests/
    ├── test_output_routes/
    │   ├── test_output_route_viewset.py
    │   └── test_delivery_service.py
    └── test_extraction_records/
        └── test_delivery_trigger.py

backend/django/common/utils/
└── ssrf_guard.py                        # New — blocks private/loopback/metadata URLs

client/src/app/documentEntry/
├── documentTypes/steps/
│   └── OutputStep.tsx                   # New — Output step in wizard
├── components/
│   ├── OutputRouteList.tsx              # New — route list + add/edit dialog
│   ├── OutputRouteForm.tsx              # New — route create/edit form (inline endpoint)
│   └── DeliverySection.tsx              # New — shared delivery status section
├── records/
│   └── ExtractionRecordDetailPage.tsx   # Modified — add DeliverySection
└── history/
    └── actionRenderers/
        └── DocumentEntryRenderer.tsx    # Modified — add read-only DeliverySection

client/src/app/documentEntry/api.ts      # Modified — output routes CRUD + deliver action
client/src/types/documentEntry.ts        # Modified — OutputRoute, DeliveryAttempt types
```

---

## Phase Overview

| Phase | Scope | User Story | Key Deliverable |
|-------|-------|------------|-----------------|
| 1 | Django — models + migration | All | `OutputRoute`, `DeliveryAttempt` tables |
| 2 | Django — output route CRUD | US1 | Nested route API under DocumentType |
| 3 | Django — delivery orchestration | US2, US3 | `DeliveryService`, SSRF guard, trigger hooks |
| 4 | Django — record detail + history APIs | US3 | Delivery section in record serializer + deliver/ endpoint |
| 5 | Frontend — Output Step | US1, US4 | Wizard step with route list, form, preview |
| 6 | Frontend — Record Detail | US3 | `DeliverySection` with approve/retry |
| 7 | Frontend — History | US3 | Read-only `DeliverySection` |

---

## Phase 1 — Data Model & Migrations

See [`data-model.md`](./data-model.md) for full field specifications.

**New models**: `OutputRoute`, `DeliveryAttempt`
**Migration**: single `000N_add_output_routes.py` — new tables only; no existing column changes

**Key constraints**:
- `OutputRoute.api_endpoint` → `OneToOneField(ApiEndpoint, on_delete=PROTECT)` — route owns endpoint; viewset's `perform_destroy` deletes endpoint after route
- `DeliveryAttempt.output_route` → `ForeignKey(OutputRoute, on_delete=SET_NULL)` — preserves log if route deleted
- `DeliveryAttempt` stores `route_label_snapshot` and `endpoint_url_snapshot` for audit stability

---

## Phase 2 — Output Route API (Django)

See [`contracts/output-routes-api.md`](./contracts/output-routes-api.md) for request/response shapes.

**Endpoints** (nested under DocumentType):
```
GET    /api/v1/document-entries/document-types/{id}/output-routes/
POST   /api/v1/document-entries/document-types/{id}/output-routes/
GET    /api/v1/document-entries/document-types/{id}/output-routes/{route_id}/
PATCH  /api/v1/document-entries/document-types/{id}/output-routes/{route_id}/
DELETE /api/v1/document-entries/document-types/{id}/output-routes/{route_id}/
```

**Create flow** (single POST):
1. Validate route fields (label, delivery_mode, repeat_policy, value_transforms)
2. Create `ApiEndpoint` from `endpoint` sub-object (same validation as API integrations app)
3. Create `OutputRoute` linking `DocumentType → ApiEndpoint`
4. Return combined response

**Delete** — viewset `perform_destroy` deletes `ApiEndpoint` first, then `OutputRoute` (PROTECT constraint requires explicit ordering).

**Permissions**: requires `manage_integrations` permission on the organisation.

---

## Phase 3 — Delivery Orchestration (Django + FastAPI)

### SSRF Guard (`common/utils/ssrf_guard.py`)

Raises `SSRFBlockedError` if URL resolves to:
- Private IPv4 ranges (10.x, 172.16-31.x, 192.168.x)
- Loopback (127.x, ::1)
- Link-local (169.254.x, fe80::/10)
- Cloud metadata endpoint (169.254.169.254)
- Non-http(s) schemes

Uses `socket.getaddrinfo` for resolution (checks after DNS, before request).

### Payload Builder (`document_entries/services.py`)

`PayloadBuilder.build(record: ExtractionRecord, route: OutputRoute) → dict`

Assembles the `general / email / data` envelope:
- `general`: `date`, `time`, `record_id` (hashid), `document_type` (name), `trace_id` (from `record.metadata`)
- `email`: `id`, `from`, `to`, `subject`, `received_at` — from `ExtractionRecord` fields
- `data`: built by **iterating the `DocumentTypeField` tree** (TP VWE-1496 §2–4), not by iterating `ExtractedFieldValue` rows directly. The tree is the canonical structure; EFV rows are looked up per field:
  1. Fetch all `ExtractedFieldValue` rows for the record via `select_related("document_type_field__parent_field")`; bucket by `(document_type_field_id, group_item_index)`.
  2. Walk top-level `DocumentTypeField` rows (`parent_field=NULL`) ordered by `display_order`:
     - **Value field** → `data[codename] = transformed_value` (or missing-value if no EFV row)
     - **`single_object_group`** → walk child fields (`parent_field=this`); if any EFV row exists → `data[group_codename] = {child_codename: val, ...}`; if optional and no rows → key omitted; if not optional → all children rendered with missing-value behaviour
     - **`repeatable_group`** → collect child EFV rows grouped by `group_item_index` (sorted 0-based ascending) → `data[group_codename] = [{child_codename: val}, ...]`; no rows → `[]`
  3. **Group fields themselves have no EFV rows** (TP §4.3) — skip when looking up values; structural only.
  - Applies `route.value_transforms` per field codename

Value transforms applied in order: (1) date format, (2) strip non-alphanumeric, (3) missing-value behaviour.

**Payload size guard**: if `len(json.dumps(payload))` > 1 MB → raise `PayloadTooLargeError` (captured in `DeliveryAttempt.error_message`).

### DeliveryService (`document_entries/services.py`)

`DeliveryService.deliver(record: ExtractionRecord, route: OutputRoute, attempt_number: int) → DeliveryAttempt`

1. Build payload via `PayloadBuilder`
2. SSRF guard check on endpoint URL
3. Fetch connection credentials via existing `api_integrations` auth service
4. Make HTTP request (`httpx`, 30 s timeout) with merged headers (endpoint headers + connection headers)
5. Create `DeliveryAttempt` with outcome, HTTP status, error, payload SHA-256 hash

`DeliveryService.trigger_auto_delivery(record: ExtractionRecord) → None`

Called after any status transition to deliverable:
- Fetch enabled Auto routes for the record's DocumentType
- For each route, check repeat policy (skip if Prevent duplicates + already sent for this record)
- Call `deliver()` per route; exceptions are caught per-route (one failure doesn't block others)
- Sentry capture for unexpected exceptions

### Trigger Hook Points

**Initial extraction** — `DocumentEntryAction.execute()` in `document_entry_action.py`:
After `create_extraction_record(...)` returns with a non-blocked status, call `await sync_to_async(DeliveryService.trigger_auto_delivery)(record)`.

**Status change by reviewer** — `ExtractionRecordViewSet` status-update action:
After saving the updated status, if new status is deliverable, call `DeliveryService.trigger_auto_delivery(record)`.

---

## Phase 4 — Record Detail & History APIs

See [`contracts/record-detail-api.md`](./contracts/record-detail-api.md) for full shape.

### `ExtractionRecordDetailSerializer` extension

`to_representation()` appends a `delivery` list. Each entry:
```json
{
  "output_route_id": "abc123",
  "route_label": "ERP Inbound",
  "endpoint_url": "https://erp.example.com/inbound",
  "delivery_mode": "requires_approval",
  "repeat_policy": "allow_resend",
  "enabled": true,
  "delivery_status": "pending_approval",
  "can_confirm": true,
  "can_retry": false,
  "can_resend": false,
  "attempts": [...]
}
```

`delivery_status` is derived (not stored): computed from latest `DeliveryAttempt` + route state.

**Delivery status derivation logic**:
- No attempt + `requires_approval` + enabled → `pending_approval`
- No attempt + `auto` + enabled → `pending` (not yet triggered, record in transition)
- Latest attempt `success` + `allow_resend` → `sent`
- Latest attempt `success` + `prevent_duplicates` → `sent`
- Latest attempt `failed` → `send_failed`
- Route disabled → `not_configured`

### Deliver Action Endpoint

```
POST /api/v1/document-entries/records/{record_id}/deliver/
Body: { "output_route_id": "abc123" }
```

Service layer checks eligibility (delivery_mode, current status, repeat_policy) and calls `DeliveryService.deliver()`. Returns updated route delivery status.

### History Detail View

`DocumentEntryHistoryDetailSerializer` (or same serializer with `is_history=True` context flag) omits `can_confirm`, `can_retry`, `can_resend`. Attempts are included read-only.

---

## Phase 5 — Frontend: Document Type Wizard (Output Step)

**New step** appended to the existing multi-step wizard as the final step.

**`OutputStep.tsx`**:
- Lists `OutputRoute` entries fetched from `/document-types/{id}/output-routes/`
- "Manage API connections" link → full-page navigation to `/integrations/connections/` (browser back returns to wizard)
- "Add output route" button opens `OutputRouteForm` dialog

**`OutputRouteList.tsx`**:
- Table rows: label, endpoint URL, delivery mode, enabled toggle, edit/delete icons
- Empty state with primary CTA

**`OutputRouteForm.tsx`** (create + edit dialog):
- Step 1: Connection picker (existing `ApiConnection` dropdown, filtered by Connected status)
- Step 2: Endpoint form (URL, method, headers — same components as API integrations tab; `body_template`, AI input vars hidden)
- Value transforms accordion: per-field date format, strip toggle, missing-value select
- Payload preview (mock data by default; toggle to real record preview if records exist)
- "Test" button — calls existing endpoint test action, displays status + response snippet

**Payload Preview**: Calls a new read endpoint `GET /output-routes/{route_id}/payload-preview/?record_id={optional}` which returns sample payload JSON. Displayed in a `<Code>` block with syntax highlighting.

**Types** added to `documentEntry.ts`:
- `OutputRoute`, `OutputRouteCreate`, `OutputRouteUpdate`
- `ValueTransforms`, `DeliveryAttempt`, `DeliveryStatus`

---

## Phase 6 — Frontend: Record Detail Delivery Section

**`DeliverySection.tsx`** (shared, not embedded in `ExtractionRecordDetailPage`):

Props: `delivery: DeliveryRouteStatus[]`, `recordId: string`, `readonly?: boolean`

- Renders a row per route: label, endpoint URL, `StatusChip` (colour-coded by `delivery_status`)
- Expandable row → `DeliveryLog` (attempt list: timestamp, HTTP status, outcome, error message)
- Inline action buttons:
  - Confirm & Send (Requires approval + pending_approval + !readonly)
  - Retry (send_failed + !readonly)
  - Re-send (sent + allow_resend + !readonly)
- All mutate via `/records/{id}/deliver/` + `invalidateQueries` on success

**Integration in `ExtractionRecordDetailPage.tsx`**: append `<DeliverySection>` below the fields section.

---

## Phase 7 — Frontend: History Read-Only Delivery Section

**`DocumentEntryRenderer.tsx`**: append `<DeliverySection delivery={...} readonly />`.

Data comes from the existing history detail API response (`delivery` field in the record serializer, already read-only by context).

---

## Complexity Tracking

No constitution violations. No entries required.
