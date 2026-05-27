# Implementation Plan: Data Output via REST API (VWE-1521)

**Branch**: `VWE-1529-vwe-1521-task-group-1-foundation-route-crud-phases-1-3`
**Date**: 2026-05-26
**Spec**: [specs/002-data-output-rest-api-mcp/spec.md](./spec.md)
**TP**: [technical_proposals/tp_vwe1521_output_route_payload_shaping.md](../../technical_proposals/tp_vwe1521_output_route_payload_shaping.md)
**Input**: Feature specification from `/specs/002-data-output-rest-api-mcp/spec.md`

> **Authoritative sources**: TP VWE-1521 is the primary technical source of truth.
> Spec clarifications override TP on user-facing terminology and v1 scope (see research.md §Deviations).

---

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Phase Status](#phase-status)
- [Remaining Work](#remaining-work)
- [Complexity Tracking](#complexity-tracking)

---

## Summary

Users need to route extracted document data to external REST endpoints with payload shaping. The `OutputRoute` model links a `DocumentType` to an `ApiEndpoint`; `PayloadBuilder` assembles the `general / email / data` JSON envelope (TP §4.2); `TemporaryEndpointExecutor` (TP §4.1) dispatches the outbound request (SSRF-guarded, 1 MB cap, 30 s timeout); delivery state is **derived** from `DeliveryAttempt` rows — no separate state-machine model needed (D-007).

**Phase 1** (model + route CRUD + wizard Output step) is **complete**. This plan covers Phases 2 and 3 and one Phase 1 gap:
- **Phase 1 gap**: `body_template` is set to `None` in `perform_create` — must be `"{{content}}"` per TP §4.1 so the executor can inject the payload via Jinja.
- **Phase 2**: `PayloadBuilder`, `TemporaryEndpointExecutor`, delivery orchestration (inline in FastAPI action + status-change hook), preview + test endpoints.
- **Phase 3**: `DeliverySection` component, manual send/retry action on ExtractionRecord, record detail + history surfaces.

---

## Technical Context

**Language/Version**: Python 3.12 (Django), TypeScript 5 (React)
**Primary Dependencies**: Django 4.2, DRF, django-pydantic-field, FastAPI, Mantine UI, TanStack Query, Vitest
**Storage**: PostgreSQL (via Django ORM)
**Testing**: pytest (Django), Vitest + @testing-library/react (frontend)
**Target Platform**: Linux server (Docker Compose), browser (React SPA)
**Project Type**: Full-stack web service — Django REST API + React frontend
**Performance Goals**: Auto-delivery within 30 s of record reaching deliverable status (FR, SC-002); synchronous v1
**Constraints**: 1 MB payload cap; 30 s request timeout; SSRF guard on all outbound URLs; no raw payload content in logs
**Scale/Scope**: Per-org; bounded by extraction record volume; single endpoint per route (OneToOneFK)

---

## Constitution Check

*GATE: Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. Protect Sensitive Data** | ✅ PASS | Raw payloads never persisted (hash only). Credentials stay in api_integrations encrypted layer. `ExecutionResult` never carries request headers. SSRF guard on every outbound call. |
| **II. Respect the Architecture** | ✅ PASS | Reuses `api_integrations` app for connections/endpoints/auth. Follows OrgQuerySetMixin, HashidGetObjectMixin, UserInContextMixin patterns. New viewset per domain resource. |
| **III. Test What Matters** | ✅ PASS | Unit tests for PayloadBuilder transforms; integration tests for delivery flow, SSRF guard, idempotency. Frontend tests for DeliverySection state chips and action visibility. |
| **IV. Ship Incrementally** | ✅ PASS | TP's three-phase split maintained. TemporaryEndpointExecutor is the sole throwaway; the rest is permanent. No premature abstraction (XML deferred). |
| **V. Make Failures Visible** | ✅ PASS | DeliveryAttempt logs every attempt. `TemporaryEndpointExecutor` errors are logged. Route failures are isolated (per-route isolation). |

---

## Project Structure

### Documentation (this feature)

```text
specs/002-data-output-rest-api-mcp/
├── plan.md              ← this file
├── research.md          ← Phase 0 output (design decisions, deviations)
├── data-model.md        ← Phase 1 output (models, relationships, gaps)
├── contracts/
│   ├── preview-endpoint.md
│   ├── test-endpoint.md
│   ├── send-retry-endpoint.md
│   └── extraction-record-detail-update.md
└── tasks.md             ← Phase 2 output (/speckit.tasks — NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
backend/django/apps/document_entries/
├── models.py                         # OutputRoute ✅, RouteDelivery ✅, DeliveryAttempt ✅
├── serializers.py                    # OutputRoute serializers ✅; delivery serializer extension ❌
├── output_route_view_set.py          # CRUD ✅; preview (stub) ✅; test action ❌; preview body ❌
├── route_delivery_view_set.py        # send/ action ❌ (to create — TP §3.4)
├── extraction_record_view_set.py     # status-update hook ❌
├── services.py                       # PayloadBuilder ❌; MockPayloadBuilder ❌; RouteDeliveryService ❌; DeliveryService ❌; TemporaryEndpointExecutor ❌
├── factories.py                      # OutputRouteFactory ✅; DeliveryAttemptFactory ✅
└── urls.py                           # ✅ (needs RouteDeliveryViewSet registration)

backend/fastapi/backbone/
└── [DocumentEntryAction.execute — orchestration hook ❌]

backend/django/common/utils/
└── output_route_executor.py          # TemporaryEndpointExecutor ❌ (TP §4.1 — throwaway)

client/src/app/documentEntry/
├── components/
│   ├── OutputRouteForm.tsx           # CRUD form ✅; value-transform fields ❌; preview panel ❌
│   ├── OutputRouteList.tsx           # Route list ✅
│   └── DeliverySection.tsx           # ❌ (to create)
├── api.ts                            # output-route hooks ✅; delivery hooks ❌; preview hook ❌
└── steps/OutputStep.tsx              # ✅

client/src/types/documentEntry.ts     # OutputRoute ✅; DeliveryAttempt ❌; DeliveryRouteStatus ❌

backend/django/tests/test_apps/test_document_entries/
├── test_output_route_viewset.py      # basic CRUD tests ✅; preview/test tests ❌ (T040)
├── test_payload_builder.py           # ❌ (to create — T021, T039)
├── test_delivery_service.py          # ❌ (to create — T026)
├── test_route_delivery_viewset.py    # ❌ (to create — T032; send/ action, override, prevent_duplicates)
└── test_extraction_record_viewset.py # delivery section tests ❌ (T031)
```

**Structure Decision**: Web application (Option 2). Django app at `backend/django/apps/document_entries/`; React feature at `client/src/app/documentEntry/`. Services live inside the Django app (not a separate services/ directory at the repo root), following the project's established pattern.

---

## Phase Status

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Model + Route CRUD + Output wizard step | ✅ Complete (one gap: `body_template`) |
| **Phase 1 gap** | Set `body_template="{{content}}"` in `perform_create` | ❌ Not started |
| **Phase 2** | PayloadBuilder + TemporaryEndpointExecutor + delivery orchestration + preview/test | ❌ Not started |
| **Phase 3** | DeliverySection + RouteDeliveryViewSet send/ action + history surfaces | ❌ Not started |

---

## Remaining Work

### Phase 1 Gap — body_template Fix

> **Why needed**: TP §4.1 requires `body_template = "{{content}}"` so `TemporaryEndpointExecutor` can render the payload via Jinja (`{"content": json.dumps(payload)}`). Current `perform_create` sets `body_template=None`, which means Phase 2 executor will send an empty body.

**Fix** (single line in `output_route_view_set.py:perform_create`):
```python
# before
body_template=None,
# after
body_template="{{content}}",
```

No migration needed. No backfill needed — nothing is in production yet.

---

### Phase 2 — Payload + Delivery

#### 2a. PayloadBuilder (`services/payload_builder.py`)

`build(record: ExtractionRecord, route: OutputRoute, *, sample: bool = False) -> dict`

Assembles `{ general, email, data }`:
- **`general`**: `date`, `time`, `record_id` (hashid), `document_type` (name), `trace_id` (from `record.metadata["trace_id"]`)
- **`email`**: `id`, `from`, `to`, `subject`, `received_at` — from ExtractionRecord email fields
- **`data`**: built from grouped `ExtractedFieldValue` rows (see data-model.md §PayloadBuilder Data Assembly Rules):
  - Top-level fields → flat keys in `data`
  - Single-object group children → nested object keyed by group codename
  - Repeatable group items → array of objects keyed by group codename, ordered by `group_item_index`
- `value_transforms` applied per field: `date_format` (default ISO 8601), `strip_non_alphanumeric`
- `missing_value` from route config: `null` or `""` for absent fields
- `sample=True`: mock placeholders (type-appropriate; 2 items per repeatable group)

**Hash**: SHA-256 of `json.dumps(payload, sort_keys=True, ensure_ascii=False).encode()` — canonical, deterministic.

#### 2b. TemporaryEndpointExecutor (`services/output_route_executor.py`)

Per TP §4.1 — the single swappable connection point. Replaced wholesale when the `api_integrations` execution wrapper lands; nothing else changes.

```python
@dataclass
class ExecutionResult:
    outcome: str                       # "success" | "failure"
    http_status: int | None
    error_message: str
    endpoint_url: str

class TemporaryEndpointExecutor:
    def execute(self, endpoint: ApiEndpoint, connection: ApiConnection, payload: dict) -> ExecutionResult
```

Steps (TP §4.1, steps 1–7):
1. Resolve auth service from `connection.provider.auth_type` via `connection_registry.get()`
2. `service.build_request(connection, endpoint)` → resolved URL + merged headers
3. `validate_safe_url(url)` (SSRF — `common/serializers/url_validators`) — reject non-`http(s)` and private/internal addresses (TP §4.1/§6)
4. Payload size check: `len(json.dumps(payload).encode("utf-8")) > 1_048_576` → return `ExecutionResult(failure, …)`
5. Render `body_template` via Jinja2: `{"content": json.dumps(payload)}`
6. HTTP request via `httpx` (sync), `endpoint.method`, 30 s timeout
7. Return `ExecutionResult` — request headers never included in result

Note: `DeliveryService.deliver()` (T022) wraps this and creates the `DeliveryAttempt` row.

#### 2c. Delivery Orchestration (FastAPI backbone)

In `DocumentEntryAction.execute()`, after `ExtractionRecord` is persisted (T024):

```python
RouteDeliveryService.initialize_and_dispatch(record)
```

`RouteDeliveryService.initialize_and_dispatch(record)` (TP §4.3):
1. Fetch all enabled `OutputRoute` rows for `record.document_type`
2. For each route: get-or-create a `RouteDelivery` row (snapshotting `route_label`)
3. For `delivery_mode=manual`: set `RouteDelivery.status = pending_approval`; do not dispatch
4. For `delivery_mode=auto` (per-route isolation — exception never propagates):
   - `PayloadBuilder.build()` → payload + SHA-256 hash
   - Re-check `prevent_duplicates` against `route_delivery.last_successful_payload_hash`; skip if duplicate
   - Set `route_delivery.status = sending`; `attempt_count += 1`
   - `TemporaryEndpointExecutor.execute(endpoint, connection, payload)` → `ExecutionResult`
   - Create `DeliveryAttempt` row (`outcome`, `http_status`, `error_message`, `endpoint_url`, `payload_hash`, `attempt_number`)
   - Success: `route_delivery.status = sent`, `last_successful_payload_hash = hash`; Failure: `route_delivery.status = send_failed`

**Re-delivery on status transitions** (T025): Wire `trigger_auto_delivery` into `ExtractionRecordViewSet` status-update action — after writing the new status, if deliverable, call `DeliveryService.trigger_auto_delivery(record)`. `RouteDelivery` rows already exist; idempotency via `prevent_duplicates` check (`last_successful_payload_hash`) handles repeated transitions.

#### 2d. Preview + Test Endpoints

Per TP §3.3 — two custom actions on `OutputRouteViewSet`:

**Preview** (`@action(detail=False, methods=["POST"], url_path="preview")` — already stubbed, returns 501):
- Body: `{ "document_type": "<hashid>", "payload_config": {...}, "record_id": "<hashid>" (optional) }`
- Accepts **unsaved** route config so the create dialog can call it before first save
- Logic: `MockPayloadBuilder.build(config, document_type)` (no record_id) or `PayloadBuilder.build(record, route)` (with record_id); record must belong to same DocumentType + org
- Response: `{ "format": "json", "payload": {...} }`
- Auth: `ApiIntegrationAccessPolicy` (same as viewset)

**Test** (`@action(detail=True, methods=["POST"], url_path="test")`):
- Body: `{}` (optional `record_id`)
- Logic: build payload → reuse `BaseConnectionService.test_endpoint(connection, endpoint, body)` from `api_integrations` — no `DeliveryAttempt` written; secret headers redacted by `test_endpoint()` internally
- Response: `{ http_status, response_snippet, timestamp }` — no header values
- Auth: `ApiIntegrationAccessPolicy` (same as viewset)

#### 2e. Frontend — Value Transform UI + Preview Panel

In `OutputRouteForm.tsx`:
- Add per-field transform section: accordion per DocumentType field; date_format input, strip_non_alphanumeric checkbox, missing_value select
- Add payload preview panel (right column or expandable): calls `POST .../output-routes/preview/` on debounce; JSON syntax highlight

Add `getPayloadPreview`, `testOutputRoute` query/mutation functions to `api.ts`.

---

### Phase 3 — Review Surfaces

#### 3a. send/retry Endpoint (TP §3.4)

New `RouteDeliveryViewSet` with a `send` action:

`POST /api/v1/document-entries/route-deliveries/{id}/send/`

Body: `{ "override": true }` (optional — bypasses `prevent_duplicates`)

- Validates: `RouteDelivery` belongs to caller's org; route enabled; delivery mode + repeat policy eligibility
- `prevent_duplicates` without `override`: 409 Conflict
- `prevent_duplicates` with `override: true`: proceed, set `DeliveryAttempt.is_override = True`
- `allow_resend`: always permitted without override
- Calls same `DeliveryService.deliver(record, route, route_delivery)` flow as Phase 2
- Returns updated `RouteDelivery` status + latest attempt

#### 3b. ExtractionRecord Detail — route_deliveries

Extend `ExtractionRecordDetailSerializer` to include (TP §3.5):

```json
"route_deliveries": [
  {
    "id": "...",
    "route_label": "...",
    "delivery_status": "sent",
    "delivery_mode": "auto",
    "repeat_policy": "prevent_duplicates",
    "enabled": true,
    "can_confirm": false,
    "can_retry": false,
    "can_resend": true,
    "attempts": [
      { "attempt_number": 1, "endpoint_url": "...", "http_status": 200, "outcome": "success", "error_message": "", "payload_hash": "...", "created_at": "..." }
    ]
  }
]
```

`delivery_status` = `route_delivery.status` (stored field — not derived). `endpoint_url` per attempt from executor. No raw payload content.

#### 3c. DeliverySection Component (`components/DeliverySection.tsx`)

- Props: `delivery: DeliveryRouteStatus[]`, `readOnly?: boolean`
- Renders one row per route delivery entry with:
  - Label + endpoint URL (from latest attempt's `endpoint_url`)
  - Status chip (`MantineColor` typed): `pending` → yellow, `pending_approval` → blue, `sent` → green, `send_failed` → red, `not_configured` → gray
  - Actions (if `!readOnly`): `can_confirm` → "Confirm & Send"; `can_retry` → "Retry"; `can_resend` → "Re-send" (Re-send on `prevent_duplicates` routes shows confirm dialog + passes `override: true`)
  - Expandable row → attempt log (timestamp, HTTP status, outcome, error, payload hash — never raw content)
- Always renders `<ApiError>` and loading `<Skeleton>`

Add `sendRouteDelivery(routeDeliveryId, override?)` mutation to `api.ts` — calls `POST .../route-deliveries/{id}/send/`.

#### 3d. Record Detail + History Integration

- Render `<DeliverySection delivery={...} />` at bottom of extraction record detail page
- Render `<DeliverySection delivery={...} readOnly />` in Document Entry history tab

#### 3e. TypeScript Types

Add to `client/src/types/documentEntry.ts`:

```typescript
export interface DeliveryRouteStatus {
  id: string                          // RouteDelivery hashid — used as param for send/ action
  output_route_id: string | null      // null when route was deleted (SET_NULL)
  route_label: string                 // snapshot from RouteDelivery.route_label
  delivery_mode: DeliveryMode
  repeat_policy: RepeatPolicy
  enabled: boolean
  delivery_status: 'not_configured' | 'pending' | 'pending_approval' | 'sent' | 'send_failed'
  can_confirm: boolean
  can_retry: boolean
  can_resend: boolean
  attempts: DeliveryAttemptEntry[]
}

export interface DeliveryAttemptEntry {
  id: string
  attempt_number: number
  created_at: string
  outcome: 'success' | 'failure'      // matches DeliveryOutcome enum
  http_status: number | null           // matches DeliveryAttempt.http_status field name
  error_message: string
  endpoint_url: string
  payload_hash: string
}
```

---

## Complexity Tracking

> No constitution violations. No deferred complexity justifications needed.

| Decision | Rationale |
|----------|-----------|
| Derived `delivery_status` (D-007), no RouteDelivery model | Simpler; no extra migration; derivation from latest `DeliveryAttempt` is sufficient for v1 sync delivery. Same pattern as data-model.md §Delivery Status Derivation. |
| `TemporaryEndpointExecutor` as throwaway | Deliberately narrow — replaced wholesale when `api_integrations` execution wrapper lands. Single swap point; nothing else moves. |
| Inline sync delivery in FastAPI worker | v1 per TP decision; no Celery dependency. Acceptable because worker is async background context. |
