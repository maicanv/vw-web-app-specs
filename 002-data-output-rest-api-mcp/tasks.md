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
backend/django/common/utils/            Shared utilities (ssrf_guard.py goes here)
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
- [x] T003 Add `OutputRoute` model (fields: `document_type` FK CASCADE, `organisation` FK CASCADE, `api_endpoint` OneToOneField PROTECT, `label` CharField(255), `delivery_mode`, `repeat_policy`, `value_transforms` JSONField default=dict, `enabled` BooleanField default=True, `created_at`, `updated_at`) to `backend/django/apps/document_entries/models.py`
- [x] T004 Add `DeliveryAttemptStatus` TextChoices enum and `DeliveryAttempt` model (fields: `extraction_record` FK CASCADE, `output_route` FK SET_NULL null=True, `route_label_snapshot` CharField(255), `endpoint_url_snapshot` URLField(2000), `status`, `http_status_code` IntegerField null=True, `error_message` TextField blank=True, `payload_hash` CharField(64), `attempt_number` PositiveIntegerField default=1, `created_at`) to `backend/django/apps/document_entries/models.py`
- [x] T005 Create migration `backend/django/apps/document_entries/migrations/000N_add_output_routes.py` — two new tables only, no existing column changes; add composite indexes `(document_type, enabled)` on OutputRoute and `(extraction_record, output_route)` + `(extraction_record, output_route, created_at)` on DeliveryAttempt
- [x] T006 [P] Create `SSRFBlockedError` exception class and `check_url(url: str) -> None` function in `backend/django/common/utils/ssrf_guard.py` — block private IPv4 ranges (10.x, 172.16-31.x, 192.168.x), loopback (127.x, ::1), link-local (169.254.x, fe80::/10), cloud metadata endpoint (169.254.169.254), and non-http(s) schemes; use `socket.getaddrinfo` for DNS resolution before range check
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
- [ ] T010b **[Phase 1 gap]** Fix `body_template=None` → `body_template="{{content}}"` in `output_route_view_set.perform_create` (TP §4.1 requirement — executor renders payload via Jinja); no backfill needed, nothing is in production
- [x] T011 [US1] Implement `perform_destroy` in `OutputRouteViewSet`: delete `api_endpoint` first (bypasses PROTECT), then route; preserve `DeliveryAttempt` rows (FK already SET_NULL)
- [x] T012 [US1] Register nested routes under DocumentType in `backend/django/apps/document_entries/urls.py` (or the app router) — pattern: `/document-types/{document_type_pk}/output-routes/` and `/{pk}/`
- [x] T013 [P] [US1] Write tests for output route CRUD (create with inline endpoint, list, patch, delete with endpoint cleanup, permission gate, blocked-connection validation) in `backend/django/tests/test_apps/test_document_entries/test_output_route_viewset.py`
- [ ] T013b [P] [US1] Add test verifying `body_template == "{{content}}"` on the `ApiEndpoint` after `OutputRoute` creation (after T010b fix) in the same file

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

**Independent Test**: Configure an Auto route, trigger an extraction, verify a `DeliveryAttempt` row is created with `status=success` and the external endpoint received the correct payload. Also: update a Needs Review record to an approved status, verify delivery fires again (per clarification Q1).

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
  3. `validate_safe_url(url)` from `common/serializers/url_validators` (SSRF — TP §4.1/§6) — raises on private IPs, loopback, link-local, metadata IPs, non-http(s); covers DNS rebinding; more complete than `check_url`
  4. Payload size check: `len(json.dumps(payload).encode("utf-8")) > 1_048_576` → return failure result
  5. Render `body_template` via Jinja2: `Template(endpoint.body_template).render({"content": json.dumps(payload)})` → final request body string
  6. `httpx` sync request, `endpoint.method`, **30 s timeout**
  7. Return `ExecutionResult(outcome, http_status, error_message, endpoint_url)` — request headers never in result
  - `ExecutionResult` is a dataclass defined alongside the executor; reuse or align with `api_integrations.ExecutionResult` if it exists
  - Then implement `DeliveryService.deliver(record, route, attempt_number) -> DeliveryAttempt` in the same file:
    1. `PayloadBuilder.build(record, route)` → payload + SHA-256 hash (catch `PayloadTooLargeError` → failed attempt)
    2. `TemporaryEndpointExecutor().execute(route.api_endpoint, route.api_connection, payload)` → `ExecutionResult`
    3. Create and return `DeliveryAttempt` with outcome, `http_status_code`, `error_message`, `payload_hash`, `attempt_number`; snapshot `route_label_snapshot = route.label` and `endpoint_url_snapshot = result.endpoint_url`
    4. Catch ALL exceptions per-route (unexpected → `logger.exception` + Sentry capture, create failed attempt)
- [ ] T023 [US2] Implement `DeliveryService.trigger_auto_delivery(record: ExtractionRecord) -> None` in `backend/django/apps/document_entries/services.py`:
  - Fetch enabled Auto routes for `record.document_type` via `(document_type, enabled=True, delivery_mode=auto)` index
  - For each route: skip if `prevent_duplicates` and latest attempt for `(record, route)` is `success`
  - Call `deliver(record, route, attempt_number=next_attempt_number)` — failures on one route never raise to the caller
- [ ] T024 [US2] Wire `trigger_auto_delivery` into `DocumentEntryAction.execute()` in `backend/django/common/actions/document_entry_action.py` — call after `create_extraction_record()` returns with a non-blocked status; use `sync_to_async` wrapper since execute() is async
- [ ] T025 [US2] Wire `trigger_auto_delivery` into the `ExtractionRecord` status-update action in `backend/django/apps/document_entries/extraction_record_view_set.py` — after writing the new status, if new status is deliverable, call `DeliveryService.trigger_auto_delivery(record)` (synchronous call from sync DRF viewset)
- [ ] T026 [P] [US2] Write DeliveryService + TemporaryEndpointExecutor tests in `backend/django/tests/test_apps/test_document_entries/test_delivery_service.py` — cover:
  - **DeliveryService.deliver**: success → attempt with status=success; HTTP 4xx → failed attempt; SSRF blocked → failed attempt with `http_status_code=None`; `prevent_duplicates` skips already-sent record; one route failure does not block others
  - **TemporaryEndpointExecutor.execute** (TP §4.1): payload > 1 MB → failure result (no HTTP fired); 30 s timeout → failure result with timeout error; `body_template="{{content}}"` correctly renders `json.dumps(payload)` via Jinja into the request body; request headers never present in `ExecutionResult`
  - **SC-003 invariant**: assert `DeliveryAttempt.error_message`, `route_label_snapshot`, `endpoint_url_snapshot` never contain raw extracted field values or document content — only metadata and the SHA-256 payload hash
  - **FR-012 negative**: assert `trigger_auto_delivery` calls `deliver()` exactly once per route per invocation — no automatic retry on failure (failed route stays `send_failed`; caller is not re-invoked)
- [ ] T027 [P] [US2] Write SSRF guard tests in `backend/django/tests/test_common/test_url_validators.py` — test `validate_safe_url` from `common/serializers/url_validators` (TP §6 — this is the function used by the executor): private IPv4 ranges blocked (10.x, 172.16-31.x, 192.168.x), loopback (127.x) blocked, link-local (169.254.x, fe80::) blocked, AWS/GCP metadata IP (169.254.169.254) blocked, public HTTPS URL passes, non-http(s) scheme blocked, DNS rebinding guard (hostname resolves to private IP) blocked

**Checkpoint**: Extraction records auto-deliver to configured routes; delivery attempts logged; failures isolated per route.

---

## Phase 5: US3 — Manual Approve / Retry + Delivery UI (P2)

**Goal**: Users can see per-route delivery status on the record detail page, manually approve pending deliveries, retry failures, and view the delivery log. History detail shows read-only delivery status.

**Independent Test**: Configure a Requires Approval route, open a processed record — Delivery section shows "Pending approval". Click Confirm & Send — status changes to "Sent" and a `DeliveryAttempt` row exists. In history detail, same record shows read-only Sent status.

### Backend — Record Detail Delivery Extension

- [ ] T028 [US3] Add `derive_delivery_status(route: OutputRoute, latest_attempt: DeliveryAttempt | None) -> str` function in `backend/django/apps/document_entries/services.py` — computes `"pending_approval"` | `"pending"` | `"sent"` | `"send_failed"` | `"not_configured"` from latest `DeliveryAttempt` + route config per data-model.md §Delivery Status Derivation (no DB column; pure derivation; business logic belongs in services not serializers)
- [ ] T029 [US3] Extend `ExtractionRecordDetailSerializer.to_representation()` in `backend/django/apps/document_entries/serializers.py` to append `delivery` list — each entry: `output_route_id`, `route_label`, `endpoint_url`, `delivery_mode`, `repeat_policy`, `enabled`, `delivery_status` (call `derive_delivery_status()` from T028), `can_confirm`, `can_retry`, `can_resend`, `attempts` (list of attempt objects: id, created_at, status, http_status_code, error_message, payload_hash, attempt_number); when `output_route` is NULL (deleted route) populate label/url from snapshot fields on the latest attempt; if `output_route` is NULL AND no attempts exist → omit the entry from the list entirely; `can_confirm/retry/resend` always False in history context (pass `is_history` via serializer context)
- [ ] T030 [US3] Add `deliver` action (`@action(detail=True, methods=["post"])`) to `ExtractionRecordViewSet` in `backend/django/apps/document_entries/extraction_record_view_set.py` — accepts `{"output_route_id": "..."}`. Per `contracts/record-detail-api.md`:
  - Validate: route belongs to record's document type, route enabled, eligibility per delivery_mode + repeat_policy
  - Call `DeliveryService.deliver()` — returns `{output_route_id, delivery_status, attempt: {...}}`
  - Error responses:
    - `400` — invalid/missing `output_route_id` or route not on this record's document type
    - `403` — missing `manage_integrations` permission
    - `409` — `prevent_duplicates` policy blocks re-send (latest attempt was success)
    - `422` — route disabled OR connection not in CONNECTED status
- [ ] T031 [P] [US3] Write tests for delivery section in record detail response (all status variants, deleted-route snapshot, can_confirm/retry/resend flags, history context flags) in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`
- [ ] T032 [P] [US3] Write tests for `deliver/` action endpoint (confirm & send success, retry success, prevent_duplicates 409, disabled route 422, wrong org 404) in `backend/django/tests/test_apps/test_document_entries/test_extraction_record_viewset.py`

### Frontend — Delivery Section

- [ ] T033 [US3] Add `DeliveryRouteStatus`, `DeliveryAttemptEntry` TypeScript types to `client/src/types/documentEntry.ts`; add `delivery: DeliveryRouteStatus[]` to `ExtractionRecordDetail` type — do not reintroduce `field_values` (already removed)
- [ ] T034 [P] [US3] Add `deliverRoute(recordId: string, outputRouteId: string)` mutation to `client/src/app/documentEntry/api.ts`
- [ ] T035 [US3] Create `DeliverySection.tsx` in `client/src/app/documentEntry/components/DeliverySection.tsx` — props: `delivery: DeliveryRouteStatus[]`, `recordId: string`, `readonly?: boolean`; renders per-route rows with label, endpoint URL, `StatusChip` colour-coded by `delivery_status` using the canonical five-value enum: `not_configured` (grey) / `pending` (yellow, auto route awaiting first trigger) / `pending_approval` (blue, requires-approval route awaiting Confirm & Send) / `sent` (green) / `send_failed` (red); expandable row shows `DeliveryLog` (attempt list: timestamp, HTTP status, outcome, error message — payload hash shown, raw content never shown); inline action buttons (Confirm & Send, Retry, Re-send) gated by `can_confirm`, `can_retry`, `can_resend` and `!readonly`; all mutations call `deliverRoute` + `invalidateQueries` on success
- [ ] T036 [US3] Integrate `<DeliverySection>` at the bottom of `client/src/app/documentEntry/records/ExtractionRecordDetailPage.tsx`, below the `fields` section
- [ ] T037 [US3] Integrate `<DeliverySection delivery={...} readonly />` at the bottom of `client/src/app/application/history/actionRenderers/DocumentEntryRenderer.tsx` — read-only; no action buttons rendered

**Checkpoint**: Users can approve, retry, and inspect delivery log from the record detail page; history shows read-only delivery status.

---

## Phase 6: US4 — Payload Preview + Endpoint Testing (P2)

**Goal**: When creating or editing a route, users can preview the shaped payload (with current transforms) and send a test request to the endpoint.

**Independent Test**: Open route edit dialog, change a date format transform, verify the preview JSON updates. Click Test — result shows HTTP status, response snippet, timestamp. Verify no secret headers appear in the test result.

### Backend — Preview Endpoint

- [ ] T038 [US4] Add TWO custom actions to `OutputRouteViewSet` in `backend/django/apps/document_entries/output_route_view_set.py`:
  - **`payload_preview`** (`@action(detail=True, methods=["get"])`) — accepts optional `record_id` query param; without `record_id` → `MockPayloadBuilder.build(route, document_type)` using type-appropriate placeholder values (string → `"example"`, date → current date in route's date format, repeatable groups → 2 mock items); with `record_id` → `PayloadBuilder.build(record, route)` (must belong to same DocumentType + org); returns `{"format": "json", "payload": {...}}`. **v1 scope**: saved routes only (detail=True). Unsaved-route preview from the create dialog is deferred — frontend can only call this after first save.
  - **`test`** (`@action(detail=True, methods=["post"])`) — TP §3.3: build payload via `PayloadBuilder` (or `MockPayloadBuilder` if no record_id), render `body_template` via Jinja with `{"content": json.dumps(payload)}`, fire request via `BaseConnectionService.test_endpoint(connection, endpoint, rendered_body)` (reuses existing api_integrations test action — handles secret header redaction internally). Returns `{ "status_code": int, "response_body": str (truncated 500 chars), "timestamp": ISO 8601 }`. No `DeliveryAttempt` is written.
- [ ] T039 [US4] Implement `MockPayloadBuilder.build(route: OutputRoute, document_type: DocumentType) -> dict` in `backend/django/apps/document_entries/services.py` — generates placeholder values per field type: `string → "example"`, `number → 0`, `boolean → false`, `currency → "0.00"`, `date → current date in route's date_format`, `enum → first available choice or "EXAMPLE"`; repeatable groups always show exactly 2 items; single_object_groups always present in mock (never omitted)
- [ ] T040 [P] [US4] Write tests for both preview and test actions in `backend/django/tests/test_apps/test_document_entries/test_output_route_viewset.py`:
  - **`payload_preview`**: mock mode (no record_id) returns placeholder data with 2 items per repeatable group; real-record mode uses real EFV data; wrong-org record returns 404; record from different DocumentType returns 400
  - **`test` action**: returns `status_code`, `response_body` snippet (truncated to 500 chars), `timestamp`; secret header values absent from response; no `DeliveryAttempt` row created after test call; connection in non-CONNECTED status → 422

### Frontend — Preview + Test Button

- [ ] T041 [US4] Add `getPayloadPreview(documentTypeId, routeId, recordId?)` query function to `client/src/app/documentEntry/api.ts`
- [ ] T042 [US4] Add payload preview panel to `OutputRouteForm.tsx` — fetches preview via `getPayloadPreview` when form is open; updates on transform changes (debounced 500 ms); toggle "Preview from selected record" switches to real data picker (record selector showing records for this DocumentType); renders JSON in Mantine `<Code>` block with syntax highlighting; shows loading skeleton while fetching
- [ ] T043 [US4] Wire "Test" button in `OutputRouteForm.tsx` — calls the new `test` action on `OutputRouteViewSet` (`POST /document-types/{id}/output-routes/{route_id}/test/`) added in T038; displays result panel: HTTP status code, response body snippet (truncated to 500 chars), timestamp; never display header values in result

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
- T031 and T032 (delivery serializer tests and deliver/ action tests) can run in parallel — same test file, different test classes

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
- **Developer B**: Phase 5 BE (deliver/ action, serializer delivery extension)

---

## Notes

- [P] = safe to parallelize (different files, no cross-task dependency)
- [US1]–[US4] = traceability to spec.md user stories
- **PayloadBuilder (T020)**: iterate the `DocumentTypeField` tree, NOT `ExtractedFieldValue` rows directly. Group fields have no EFV rows. Absent optional single_object_group → omit key from `data` (do not render as null or {}).
- Secret headers never appear in delivery logs, test results, or payload previews — enforced in DeliveryService (T022) and OutputRouteForm test button (T043).
- All test file paths follow the existing pattern in `backend/django/tests/test_apps/test_document_entries/`.
- **body_template fix (T010b)**: Must be done before Phase 4 (T020–T027). `TemporaryEndpointExecutor` expects `body_template="{{content}}"`. One-line fix — no migration, no backfill (not in production).
