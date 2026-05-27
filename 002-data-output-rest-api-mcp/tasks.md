---
description: "Task list for Data Output via REST API (VWE-1521)"
---

# Tasks: Data Output via REST API

**Spec**: `specs/002-data-output-rest-api-mcp/spec.md`
**Plan**: `specs/002-data-output-rest-api-mcp/plan.md`
**TP**: `technical_proposals/tp_vwe1521_output_route_payload_shaping.md` (authoritative technical source of truth)

## Table of Contents

- [Format](#format-id-p-story-description)
- [Path Conventions](#path-conventions)
- [Phase 1: Setup](#phase-1-setup)
- [Phase 2: Foundational (Data Models + SSRF Guard)](#phase-2-foundational-data-models--ssrf-guard)
- [Phase 3: US1 — Configure Output Routes (P1)](#phase-3-us1--configure-output-routes-p1)
- [Phase 4: US2 — Auto Delivery (P1)](#phase-4-us2--auto-delivery-p1)
- [Phase 5: US3 — Manual Approve / Retry + Delivery UI (P2)](#phase-5-us3--manual-approve--retry--delivery-ui-p2)
- [Phase 6: US4 — Payload Preview + Endpoint Testing (P2)](#phase-6-us4--payload-preview--endpoint-testing-p2)
- [Phase 7: Polish & Cross-Cutting Concerns](#phase-7-polish--cross-cutting-concerns)
- [Dependencies & Execution Order](#dependencies--execution-order)
- [Implementation Strategy](#implementation-strategy)

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no shared dependencies)
- **[Story]**: Maps to user story from spec.md (US1–US4)
- All task descriptions include exact file paths

## Path Conventions

```
backend/django/apps/document_entries/   Django app (models, serializers, viewsets, services)
backend/django/common/utils/            Shared utilities
backend/django/tests/test_apps/         Django tests
client/src/app/documentEntry/          Frontend feature module
client/src/types/documentEntry.ts       Shared TypeScript types
```

---

## Phase 1: Setup

**Purpose**: Verify field grouping prerequisites are in place before any implementation starts.

- [x] T001 Verify field grouping is in place — run `docker compose exec django python manage.py showmigrations document_entries` and confirm the field-grouping migration is present (`parent_field` on `DocumentTypeField`, `group_item_index` on `ExtractedFieldValue`)

---

## Phase 2: Foundational (Data Models + SSRF Guard)

**Purpose**: Core DB tables and security utility that ALL user stories depend on.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [x] T002 Add `DeliveryMode` and `RepeatPolicy` TextChoices enums to `backend/django/apps/document_entries/models.py`
- [x] T003 Add `OutputRoute` model — **implemented per TP §2.1**: inherits `WithSingleEndpointMixin` (FK `api_connection` + FK `api_endpoint`, both SET_NULL); fields: `document_type` FK CASCADE, `label` CharField(100), `payload_format`, `delivery_mode`, `repeat_policy`, `is_enabled`, `payload_config` SchemaField(PayloadConfig), `display_order` IntegerField default=0
- [x] T004 Add `RouteDelivery` and `DeliveryAttempt` models — **implemented per TP §2.3–2.4**: `RouteDelivery` (extraction_record FK, output_route FK SET_NULL, route_label snapshot, status, attempt_count, last_successful_payload_hash); `DeliveryAttempt` (route_delivery FK, attempt_number, endpoint_url, outcome, http_status, error_message, payload_hash, is_override)
- [x] T005 Create migration `0008_outputroute_routedelivery_deliveryattempt_and_more.py` — three new tables; composite index `(document_type, is_enabled)` on OutputRoute; partial unique constraint `(extraction_record, output_route)` where output_route IS NOT NULL on RouteDelivery
- [x] T006 ~~[P] Create `SSRFBlockedError` / `check_url()` in `ssrf_guard.py`~~ — **removed**; TP §6 is authoritative: use `validate_safe_url` from `common/serializers/url_validators.py` only. `ssrf_guard.py` deleted (see research.md D-008).
- [x] T007 [P] Add `OutputRouteFactory` and `DeliveryAttemptFactory` to `backend/django/apps/document_entries/factories.py`

**Checkpoint**: Migration applied, models importable, SSRF guard importable — user story work can begin.

---

## Phase 3: US1 — Configure Output Routes (P1)

**Goal**: Users can add, edit, enable/disable, and delete output routes on a Document Type via the wizard Output step.

**Independent Test**: POST a new output route (with inline endpoint creation) to `/api/v1/document-entries/document-types/{id}/output-routes/`, verify 201 response; PATCH label; DELETE — confirm endpoint is also deleted; open wizard Output step in browser and add a route from start to finish.

### Backend — Output Route CRUD

- [x] T008 [US1] Add `OutputRouteListSerializer` and `OutputRouteDetailSerializer` to `backend/django/apps/document_entries/serializers.py` — list serializer returns id, label, endpoint_url, endpoint_method, delivery_mode, repeat_policy, enabled, value_transforms, created_at; detail adds nested endpoint editable fields
- [x] T009 [US1] Create `OutputRouteViewSet` (ModelViewSet) in `backend/django/apps/document_entries/output_route_view_set.py` — scoped to `document_type_id` URL kwarg, org-scoped queryset, `manage_integrations` permission gate
- [x] T010 [US1] Implement `perform_create` in `OutputRouteViewSet`: atomically create `ApiEndpoint` from `endpoint` sub-object (set `body_template` automatically, hide AI-specific fields), then create `OutputRoute` linking both — raise 400 if connection not in CONNECTED status
- [x] T010b **[Phase 1 gap]** Fix `body_template=None` → `body_template="{{content}}"` in `output_route_view_set.perform_create` (TP §4.1 requirement — executor renders payload via Jinja); no backfill needed, nothing is in production — **done in PR #745 line 100**
- [x] T011 [US1] Implement `perform_destroy` in `OutputRouteViewSet`: delete `api_endpoint` first (bypasses PROTECT), then route; preserve `DeliveryAttempt` rows (FK already SET_NULL)
- [x] T012 [US1] Register nested routes under DocumentType in `backend/django/apps/document_entries/urls.py` (or the app router) — pattern: `/document-types/{document_type_pk}/output-routes/` and `/{pk}/`
- [x] T013 [P] [US1] Write tests for output route CRUD (create with inline endpoint, list, patch, delete with endpoint cleanup, permission gate, blocked-connection validation) in `backend/django/tests/test_apps/test_document_entries/test_output_route_viewset.py`
- [x] T013b [P] [US1] Add test verifying `body_template == "{{content}}"` on the `ApiEndpoint` after `OutputRoute` creation (after T010b fix) in the same file

### Frontend — Wizard Output Step

- [x] T014 [US1] Add `OutputRoute`, `OutputRouteCreate`, `OutputRouteUpdate`, `ValueTransforms` TypeScript types to `client/src/types/documentEntry.ts`
- [x] T015 [P] [US1] Add `getOutputRoutes`, `createOutputRoute`, `updateOutputRoute`, `deleteOutputRoute` API functions to `client/src/app/documentEntry/api.ts` — use `useApiQuery` / `useApiMutation` pattern from existing code
- [x] T016 [US1] Create `OutputRouteList.tsx` in `client/src/app/documentEntry/components/OutputRouteList.tsx` — table of routes (label, endpoint URL, delivery mode, enabled toggle, edit/delete icons), empty state with "Add output route" CTA, "Manage API connections" link (full-page navigation via `<Link>`, NOT new tab)
- [x] T017 [US1] Create `OutputRouteForm.tsx` in `client/src/app/documentEntry/components/OutputRouteForm.tsx` — dialog with: connection picker (existing ApiConnection filtered to CONNECTED status), endpoint form (URL, method, headers — reuse existing API integrations components, hide body_template and AI fields), value transforms accordion (per-field date format, strip toggle, missing-value select), Save/Cancel buttons; blocks save when no label or connection not connected
- [x] T018 [US1] Create `OutputStep.tsx` in `client/src/app/documentEntry/documentTypes/steps/OutputStep.tsx` — wraps `OutputRouteList` + `OutputRouteForm`, fetches routes via `getOutputRoutes`, shows loading/error states
- [x] T019 [US1] Add `OutputStep` to the Document Type wizard step array in `client/src/app/documentEntry/documentTypes/DocumentTypeWizard.tsx` (or equivalent wizard container) as the final step

**Checkpoint**: Users can configure output routes end-to-end in the wizard; routes persist and appear in the list on reload.

---

## Phase 4: US2 — Auto Delivery (P1)

**Goal**: After a record transitions to a deliverable status (including after reviewer approval), all enabled Auto routes for that Document Type receive the `general / email / data` payload automatically.

**Independent Test**: Configure an Auto route, trigger an extraction, verify a `RouteDelivery` row has `status=sent` and a `DeliveryAttempt` row has `outcome=success` and the external endpoint received the correct payload. Also: update a Needs Review record to an approved status, verify delivery fires again (per clarification Q1).

### Backend — Payload Builder

- [ ] T020 [US2] Implement `PayloadBuilder.build(record: ExtractionRecord, route: OutputRoute) -> dict` in `backend/django/apps/document_entries/services.py`:
  - `general` section: `date`, `time`, `record_id` (hashid), `document_type` (name), `trace_id` (from `record.metadata`)
  - `email` section: `id` (`email_message_id`), `from` (`email_sender`), `to`, `subject` (`email_subject`), `received_at` (`email_received_at`)
  - `data` section: **iterate the `DocumentTypeField` tree** (not EFV rows directly): fetch all `ExtractedFieldValue` rows via `select_related("document_type_field__parent_field")`; bucket by `(document_type_field_id, group_item_index)`; walk top-level fields ordered by `display_order`; group fields have no EFV rows (structural only); single_object_group absent + optional → omit key from `data`; repeatable_group no rows → `[]`
  - Apply `route.value_transforms` per field codename: (1) date format, (2) strip_non_alphanumeric, (3) missing_value_behaviour
  - Raise `PayloadTooLargeError` if `len(json.dumps(payload, default=str)) > 1_048_576`
- [ ] T021 [P] [US2] Write PayloadBuilder unit tests in `backend/django/tests/test_apps/test_document_entries/test_payload_builder.py` — cover: flat fields, repeatable group (multi-item), single_object_group (present and absent/optional), value transforms (date format, strip, missing=null vs empty), payload exceeding 1 MB

### Backend — Delivery Service

- [ ] T022 [US2] Implement `TemporaryEndpointExecutor` class (TP §4.1 — deliberately narrow, single swappable unit) in `backend/django/apps/document_entries/services.py`:
  - `execute(endpoint: ApiEndpoint, connection: ApiConnection, payload: dict) -> ExecutionResult`
  1. `connection_registry.get(connection.provider.auth_type)` → auth service
  2. `service.build_request(connection, endpoint)` → resolved URL + merged headers
  3. `validate_safe_url(url)` from `common/serializers/url_validators` (SSRF — TP §4.1/§6)
  4. Payload size check: `len(json.dumps(payload).encode("utf-8")) > 1_048_576` → return failure result
  5. Render `body_template` via Jinja2: `Template(endpoint.body_template).render({"content": json.dumps(payload)})` → final request body string
  6. `httpx` sync request, `endpoint.method`, **30 s timeout**
  7. Return `ExecutionResult(outcome, http_status, error_message, endpoint_url)` — request headers never in result
  - `ExecutionResult` is a dataclass defined alongside the executor; reuse or align with `api_integrations.ExecutionResult` if it exists
  - Then implement `RouteDeliveryService.initialize_and_dispatch(record: ExtractionRecord) -> None` in the same file (TP §4.3):
    1. Fetch all enabled `OutputRoute` rows for `record.document_type` (`is_enabled=True`)
    2. For each route, get-or-create a `RouteDelivery` row (`extraction_record=record, output_route=route`) snapshotting `route_label=route.label`
    3. For `delivery_mode=manual`: set `route_delivery.status = RouteDeliveryStatus.PENDING_APPROVAL`; do not dispatch
    4. For `delivery_mode=auto`: call `deliver(record, route, route_delivery)` (see below) — per-route isolation, exceptions never propagate
  - Then implement `DeliveryService.deliver(record, route, route_delivery) -> DeliveryAttempt` in the same file:
    1. `PayloadBuilder.build(record, route)` → payload + SHA-256 hash (catch `PayloadTooLargeError` → create failed attempt, set `route_delivery.status = send_failed`, return)
    2. If `repeat_policy=prevent_duplicates` and `route_delivery.last_successful_payload_hash` is not None → skip (do not send); return early
    3. Set `route_delivery.status = RouteDeliveryStatus.SENDING`; `route_delivery.attempt_count += 1`
    4. `TemporaryEndpointExecutor().execute(route.api_endpoint, route.api_connection, payload)` → `ExecutionResult`
    5. Create `DeliveryAttempt` with `outcome=result.outcome`, `http_status=result.http_status`, `error_message=result.error_message`, `endpoint_url=result.endpoint_url`, `payload_hash=hash`, `attempt_number=route_delivery.attempt_count`
    6. On success: set `route_delivery.status = sent`, `route_delivery.last_successful_payload_hash = hash`
    7. On failure: set `route_delivery.status = send_failed`
    8. Catch ALL exceptions per-route: `logger.exception` + Sentry capture, create failed attempt, set `route_delivery.status = send_failed`
- [ ] T023 [US2] Implement `DeliveryService.trigger_auto_delivery(record: ExtractionRecord) -> None` in `backend/django/apps/document_entries/services.py` — called on status transitions to re-dispatch routes for records already initialized (TP §4.3 re-delivery on status change):
  - Fetch enabled Auto routes for `record.document_type` via `(document_type, is_enabled=True, delivery_mode=auto)` index
  - For each route: fetch its `RouteDelivery` row; skip if `prevent_duplicates` and `route_delivery.last_successful_payload_hash` is not None
  - Call `deliver(record, route, route_delivery)` — failures on one route never raise to the caller
- [ ] T024 [US2] Wire `initialize_and_dispatch` into `DocumentEntryAction.execute()` in `backend/django/common/actions/document_entry_action.py` — call `RouteDeliveryService.initialize_and_dispatch(record)` after `create_extraction_record()` returns; use `sync_to_async` wrapper since execute() is async. This is the first-time initialization path (creates `RouteDelivery` rows + dispatches auto routes).
- [ ] T025 [US2] Wire `trigger_auto_delivery` into the `ExtractionRecord` status-update action in `backend/django/apps/document_entries/extraction_record_view_set.py` — after writing the new status, if new status is deliverable, call `DeliveryService.trigger_auto_delivery(record)` (synchronous call from sync DRF viewset). `RouteDelivery` rows already exist from T024; this re-dispatches auto routes respecting `prevent_duplicates`.
- [ ] T026 [P] [US2] Write DeliveryService + TemporaryEndpointExecutor tests in `backend/django/tests/test_apps/test_document_entries/test_delivery_service.py` — cover:
  - **DeliveryService.deliver**: success → `RouteDelivery.status=sent`, `last_successful_payload_hash` set, `DeliveryAttempt.outcome=success`; HTTP 4xx → `RouteDelivery.status=send_failed`, `DeliveryAttempt.outcome=failure`; SSRF blocked → failed attempt with `http_status=None`; `prevent_duplicates` skips route when `last_successful_payload_hash` already set; one route failure does not block others
  - **TemporaryEndpointExecutor.execute** (TP §4.1): payload > 1 MB → failure result (no HTTP fired); 30 s timeout → failure result with timeout error; `body_template="{{content}}"` correctly renders `json.dumps(payload)` via Jinja into the request body; request headers never present in `ExecutionResult`
  - **SC-003 invariant**: assert `DeliveryAttempt.error_message` and `DeliveryAttempt.endpoint_url` never contain raw extracted field values or document content — only metadata and the SHA-256 payload hash
  - **FR-012 negative**: assert `trigger_auto_delivery` calls `deliver()` exactly once per route per invocation — no automatic retry on failure (failed route stays `send_failed`; caller is not re-invoked)
- [ ] T027 [P] [US2] Write `validate_safe_url` integration tests in `backend/django/tests/test_common/test_serializers/test_url_validators.py` — cover: private IPv4 ranges blocked (10.x, 172.16-31.x, 192.168.x), loopback blocked, link-local blocked, metadata IP (169.254.169.254) blocked, public HTTPS URL passes, non-https scheme blocked in prod, DNS rebinding blocked; all checks skipped in dev/CI per existing behaviour

**Checkpoint**: Extraction records auto-deliver to configured routes; delivery attempts logged; failures isolated per route.

---

## Phase 5: US3 — Manual Approve / Retry + Delivery UI (P2)

**Goal**: Users can see per-route delivery status on the record detail page, manually approve pending deliveries, retry failures, and view the delivery log. History detail shows read-only delivery status.

**Independent Test**: Configure a Requires Approval route, open a processed record — Delivery section shows "Pending approval". Click Confirm & Send — status changes to "Sent" and a `DeliveryAttempt` row exists. In history detail, same record shows read-only Sent status.

### Backend — Record Detail Delivery Extension

- [ ] T028 [US3] ~~Add `derive_delivery_status()` function~~ — **not needed**. `RouteDelivery.status` is the stored, authoritative status field (TP §2.3, §4.3). Serialise it directly; do not re-derive from attempts. The five UI chip values (`pending`, `pending_approval`, `sent`, `send_failed`) map 1-to-1 to `RouteDeliveryStatus` enum values. `not_configured` is the UI state for an `OutputRoute` that has **no `RouteDelivery` row** for this record (route was added after the record was processed) — surface this in the serializer by checking for a missing row, not by deriving from attempts.
- [ ] T029 [US3] Extend `ExtractionRecordDetailSerializer.to_representation()` in `backend/django/apps/document_entries/serializers.py` to append `route_deliveries` list (TP §3.5 key name) — prefetch `route_deliveries__attempts` on the queryset; for each `RouteDelivery` row: `output_route_id` (hashid of `route_delivery.output_route_id`), `route_label` (from `route_delivery.route_label` snapshot), `endpoint_url` (from latest attempt's `endpoint_url` if present, else None), `delivery_mode` / `repeat_policy` / `enabled` (from live `output_route` if not NULL, else omit), `delivery_status` (`route_delivery.status`), `can_confirm` / `can_retry` / `can_resend` (derive from `status`, `delivery_mode`, `repeat_policy`, and `is_history` context flag), `attempts` (list: id, created_at, `outcome`, `http_status`, `error_message`, `payload_hash`, `attempt_number`); if `output_route` is NULL (deleted) populate mode/policy/enabled from most recent attempt snapshot or omit; if `output_route` is NULL AND no attempts exist → omit the entry entirely; pass `is_history=True` via serializer context from the history endpoint → `can_confirm/retry/resend` all False
- [ ] T030 [US3] Add `send` action (`@action(detail=True, methods=["post"], url_path="send")`) to a new `RouteDeliveryViewSet` in `backend/django/apps/document_entries/` — registered at `route-deliveries/{id}/send/` (TP §3.4). Takes the `RouteDelivery` hashid as the URL param (no body required for confirm/retry; optional `{"override": true}` body to bypass `prevent_duplicates`):
  - Validate: `route_delivery` belongs to caller's org; route enabled; eligibility per `delivery_mode` + `repeat_policy`
  - `prevent_duplicates` without override → 409; with `override=True` → proceed and set `DeliveryAttempt.is_override=True`
  - Call `DeliveryService.deliver(record, route, route_delivery)` — returns updated `RouteDelivery` status + latest attempt
  - Error responses: `400` bad request, `403` missing permission, `409` prevent_duplicates blocked, `422` route disabled or connection not CONNECTED
- [ ] T031 [P] [US3] Write tests for delivery section in record detail response (`route_deliveries` key, all status variants, deleted-route snapshot, can_confirm/retry/resend flags, history context flags) in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`
- [ ] T032 [P] [US3] Write tests for `route-deliveries/{id}/send/` action (confirm & send success, retry success, prevent_duplicates 409, override bypasses 409, disabled route 422, wrong org 404) in a new `backend/django/tests/test_apps/test_document_entries/test_route_delivery_viewset.py`

### Frontend — Delivery Section

- [ ] T033 [US3] Add `DeliveryRouteStatus`, `DeliveryAttemptEntry` TypeScript types to `client/src/types/documentEntry.ts`; add `delivery: DeliveryRouteStatus[]` to `ExtractionRecordDetail` type — do not reintroduce `field_values` (already removed)
- [ ] T034 [P] [US3] Add `sendRouteDelivery(routeDeliveryId: string, override?: boolean)` mutation to `client/src/app/documentEntry/api.ts` — calls `POST /api/v1/document-entries/route-deliveries/{id}/send/` with optional `{"override": true}` body (TP §3.4)
- [ ] T035 [US3] Create `DeliverySection.tsx` in `client/src/app/documentEntry/components/DeliverySection.tsx` — props: `delivery: DeliveryRouteStatus[]`, `readonly?: boolean`; renders per-route rows with label, endpoint URL, `StatusChip` colour-coded by `delivery_status` using the canonical five-value enum: `not_configured` (grey) / `pending` (yellow, auto route awaiting first trigger) / `pending_approval` (blue, requires-approval route awaiting Confirm & Send) / `sent` (green) / `send_failed` (red); expandable row shows `DeliveryLog` (attempt list: timestamp, HTTP status, outcome, error message — payload hash shown, raw content never shown); inline action buttons (Confirm & Send, Retry, Re-send) gated by `can_confirm`, `can_retry`, `can_resend` and `!readonly`; all mutations call `sendRouteDelivery(routeDeliveryId)` + `invalidateQueries` on success; Re-send after `prevent_duplicates` prompts confirm dialog and passes `override: true`
- [ ] T036 [US3] Integrate `<DeliverySection>` at the bottom of `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`, below the `fields` section
- [ ] T037 [US3] Integrate `<DeliverySection delivery={...} readonly />` at the bottom of `client/src/app/application/history/actionRenderers/DocumentEntryRenderer.tsx` — read-only; no action buttons rendered

**Checkpoint**: Users can approve, retry, and inspect delivery log from the record detail page; history shows read-only delivery status.

---

## Phase 6: US4 — Payload Preview + Endpoint Testing (P2)

**Goal**: When creating or editing a route, users can preview the shaped payload (with current transforms) and send a test request to the endpoint.

**Independent Test**: Open route edit dialog, change a date format transform, verify the preview JSON updates. Click Test — result shows HTTP status, response snippet, timestamp. Verify no secret headers appear in the test result.

### Backend — Preview Endpoint

- [ ] T038 [US4] The `preview` action stub already exists (`detail=False, POST, url_path="preview"` per TP §3.3). Implement its body and add the `test` action:
  - **`preview`** (`@action(detail=False, methods=["post"], url_path="preview")` — already registered, returning 501) — body: `{"document_type": "<hashid>", "payload_config": {...}}` plus optional `"record_id"`; without `record_id` → `MockPayloadBuilder.build(route_config, document_type)` using type-appropriate placeholder values (string → `"example"`, date → current date in route's date format, repeatable groups → 2 mock items); with `record_id` → `PayloadBuilder.build(record, route)` (record must belong to same DocumentType + org); returns `{"format": "json", "payload": {...}}`. Accepts unsaved route config so the create dialog can call it before first save (TP §3.3).
  - **`test`** (`@action(detail=True, methods=["post"])`) — TP §3.3: build payload via `PayloadBuilder` (or `MockPayloadBuilder` if no record_id), render `body_template` via Jinja with `{"content": json.dumps(payload)}`, fire request via `BaseConnectionService.test_endpoint(connection, endpoint, rendered_body)` (reuses existing api_integrations test action — handles secret header redaction internally). Returns `{ "status_code": int, "response_body": str (truncated 500 chars), "timestamp": ISO 8601 }`. No `DeliveryAttempt` is written.
- [ ] T039 [US4] Implement `MockPayloadBuilder.build(route: OutputRoute, document_type: DocumentType) -> dict` in `backend/django/apps/document_entries/services.py` — generates placeholder values per field type: `string → "example"`, `number → 0`, `boolean → false`, `currency → "0.00"`, `date → current date in route's date_format`, `enum → first available choice or "EXAMPLE"`; repeatable groups always show exactly 2 items; single_object_groups always present in mock (never omitted)
- [ ] T040 [P] [US4] Write tests for both preview and test actions in `backend/django/tests/test_apps/test_document_entries/test_output_route_viewset.py`:
  - **`preview`** (collection-level POST): mock mode (no record_id) returns placeholder data with 2 items per repeatable group; real-record mode uses real EFV data; wrong-org record returns 404; record from different DocumentType returns 400
  - **`test` action**: returns `status_code`, `response_body` snippet (truncated to 500 chars), `timestamp`; secret header values absent from response; no `DeliveryAttempt` row created after test call; connection in non-CONNECTED status → 422

### Frontend — Preview + Test Button

- [ ] T041 [US4] Add `getPayloadPreview(documentTypeId, routeId, recordId?)` query function to `client/src/app/documentEntry/api.ts`
- [ ] T042 [US4] Add payload preview panel to `OutputRouteForm.tsx` — fetches preview via `getPayloadPreview` when form is open; updates on transform changes (debounced 500 ms); toggle "Preview from selected record" switches to real data picker (record selector showing records for this DocumentType); renders JSON in Mantine `<Code>` block with syntax highlighting; shows loading skeleton while fetching
- [ ] T043 [US4] Wire "Test" button in `OutputRouteForm.tsx` — calls the `test` action on `OutputRouteViewSet` (`POST /api/v1/document-entries/output-routes/{route_id}/test/`) added in T038; displays result panel: HTTP status code, response body snippet (truncated to 500 chars), timestamp; never display header values in result

**Checkpoint**: Users can preview shaped payload and test the endpoint connection without leaving the route form.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T044 [P] Complete integration checklist: (1) verify `delivery` key appears in `ExtractionRecordDetail` API response for records whose DocumentType has configured routes, (2) verify `delivery` key is absent when DocumentType has no routes, (3) verify history detail renders delivery section read-only (no action buttons), (4) verify no secret header values in delivery log or test result responses, (5) smoke-test prevent_duplicates blocks second delivery attempt for same (record, route) pair
- [ ] T045 [P] Add `staleTime: 5 * 60 * 1000` to `useApiQuery` calls for output route list in `OutputStep.tsx` and delivery attempts in `DeliverySection.tsx` to prevent background refetches on every render (per project code review rule)
- [ ] T046 [P] Verify `DeliverySection` renders `<ApiError error={error} />` unconditionally (not wrapped in conditional — per project frontend rules) in `DeliverySection.tsx`
- [ ] T047 Smoke-test full flow using `quickstart.md`: apply migration, create doc type, add route pointing to `https://httpbin.org/post`, trigger extraction, verify delivery log shows success in record detail

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 — **blocks all user stories**
- **Phase 3 (US1)**: Depends on Phase 2 — BE tasks can start when models+migration done; FE can start when BE CRUD is ready
- **Phase 4 (US2)**: Depends on Phase 2 — PayloadBuilder and DeliveryService depend only on models
- **Phase 5 (US3)**: Depends on Phase 2 + Phase 4 — serializer extension needs DeliveryAttempt; deliver/ endpoint needs DeliveryService
- **Phase 6 (US4)**: Depends on Phase 3 (OutputRouteViewSet exists) + Phase 4 (PayloadBuilder exists)
- **Phase 7 (Polish)**: Depends on all preceding phases

### User Story Dependencies

- **US1 (P1)**: After Phase 2. BE and FE can proceed in parallel once models are done.
- **US2 (P1)**: After Phase 2. Can run in parallel with US1 — shares no files except models.
- **US3 (P2)**: After Phase 4 (DeliveryService). BE delivery extension + FE `DeliverySection`.
- **US4 (P2)**: After Phase 3 (OutputRouteViewSet) + Phase 4 (PayloadBuilder).

### Parallel Opportunities Within Phases

- T006 (SSRF guard) and T007 (factories) can run in parallel with T002–T005 (models + migration) — different files
- T013 (TS types) and T015 (API functions) can run in parallel — different files
- T021 (PayloadBuilder tests) and T027 (SSRF tests) can run in parallel — different test files
- T031 (delivery serializer tests) and T032 (RouteDelivery send/ action tests) can run in parallel — different test files

---

## Implementation Strategy

### MVP First (Phase 2 + Phase 3 only)

1. Complete Phase 1 (prerequisite check)
2. Complete Phase 2 (models + migration + SSRF guard)
3. Complete Phase 3 (output route CRUD + wizard Output step)
4. **STOP and VALIDATE**: Users can configure routes, routes persist, wizard step renders
5. Proceed to Phase 4 (delivery) once routes are verified

### Incremental Delivery

1. Phases 1–3 → Route configuration working (no delivery yet)
2. Phase 4 → Auto delivery working (routes fire on extraction)
3. Phase 5 → Delivery visible in UI; manual approve/retry working
4. Phase 6 → Payload preview and test working in route form
5. Phase 7 → Checklist + polish

### Parallel Team Strategy

Once Phase 2 is complete:
- **Developer A**: Phase 3 BE (OutputRouteViewSet, serializer, inline endpoint creation)
- **Developer B**: Phase 4 BE (PayloadBuilder, DeliveryService, trigger hooks)
- **Developer C**: Phase 3 FE (OutputStep, OutputRouteList, OutputRouteForm)

After Phase 3 + 4 complete:
- **Developer A/C**: Phase 5 FE (DeliverySection, record detail integration, history integration)
- **Developer B**: Phase 5 BE (RouteDeliveryViewSet send/ action, serializer delivery extension)

---

## Notes

- [P] = safe to parallelize (different files, no cross-task dependency)
- [US1]–[US4] = traceability to spec.md user stories
- **PayloadBuilder (T020)**: iterate the `DocumentTypeField` tree, NOT `ExtractedFieldValue` rows directly. Group fields have no EFV rows. Absent optional single_object_group → omit key from `data` (do not render as null or {}).
- Secret headers never appear in delivery logs, test results, or payload previews — enforced in DeliveryService (T022) and OutputRouteForm test button (T043).
- All test file paths follow the existing pattern in `backend/django/tests/test_apps/test_document_entries/`.
- **body_template fix (T010b)**: Must be done before Phase 4 (T020–T027). `TemporaryEndpointExecutor` expects `body_template="{{content}}"`. One-line fix — no migration, no backfill (not in production).
