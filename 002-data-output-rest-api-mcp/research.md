# Phase 0 Research: Data Output via REST API

**Feature**: `002-data-output-rest-api-mcp`
**Date**: 2026-05-26

---

## TP VWE-1521 Deviations

The original plan was built without referencing TP VWE-1521. This section documents where the Phase 1 implementation diverges from TP, and the disposition for each divergence.

| # | TP Design | Implementation | Disposition |
|---|-----------|----------------|-------------|
| **DEV-1** | `RouteDelivery` (state tracker) + `DeliveryAttempt` (attempt log) — two models | Single `DeliveryAttempt` with snapshot fields (`route_label_snapshot`, `endpoint_url_snapshot`) | **Keep merged approach**. `delivery_status` is derived from latest attempt (D-007). Simpler, no extra migration, fully sufficient for v1 sync delivery. |
| **DEV-2** | `api_endpoint` FK → ApiEndpoint, SET_NULL, nullable | `api_endpoint` OneToOneField → ApiEndpoint, PROTECT | **Keep OneToOneField**. One dedicated endpoint per route (spec FR-002). PROTECT + explicit cascade in `perform_destroy` achieves the same safety goal. |
| **DEV-3** | `payload_config = SchemaField(schema=PayloadConfig)` Pydantic model | `value_transforms = JSONField(default=dict)` with inline shape | **Keep JSONField**. Functionally equivalent for v1 (no schema_version needed). Pydantic wrapper can be added non-breakingly later. |
| **DEV-4** | `DeliveryMode.manual` | `DeliveryMode.requires_approval` | **Keep `requires_approval`**. Spec and UI copy use "Requires Approval"; spec overrides TP on user-facing terminology. |
| **DEV-5** | `DeliveryAttempt.is_override` + `override=True` body param | Not implemented | **Keep omitted**. Spec FR-013 explicitly removes the override path in v1: "prevent_duplicates has no override path in v1." Spec overrides TP. |
| **DEV-6** | `OutputRoute.display_order = IntegerField(default=0)` | Not implemented; routes ordered by `created_at, pk` | **Defer**. DnD reorder is a future enhancement. No impact on v1 behavior. |
| **DEV-7** | `OutputRoute.payload_format` (json/xml stub) | Not implemented | **Defer**. Spec: "Payload format is JSON only in v1." Enum stub adds complexity with no v1 value. |
| **DEV-8** | `body_template = "{{content}}"` set on endpoint creation | `body_template=None` in `perform_create` | **Fix in Phase 1 gap** (T010b). `TemporaryEndpointExecutor` (Phase 2) depends on this. No backfill needed — nothing is in production yet. One-line fix in `perform_create`. |
| **DEV-9** | `WithSingleEndpointMixin` extracted to `api_integrations/mixins.py` | Fields inline on `OutputRoute` | **Defer**. No second consumer of this mixin in v1. Inline is clear and avoids premature abstraction. |
| **DEV-10** | TP §3.1: manual send via `POST /route-deliveries/{id}/send/` (separate resource) | Plan uses `POST /records/{id}/deliver/` with `{"output_route_id": "..."}` body | **Keep existing plan approach**. Simpler — no additional state model or router registration needed. Same behavior. |
| **DEV-11** | TP §3.1: preview is `POST /output-routes/preview/` at collection level — accepts unsaved route config | Implemented as `GET detail=True` on a saved route (`/output-routes/{id}/payload_preview/`) | **Keep v1 scope cut**. Unsaved-route preview deferred to v2. Create-dialog preview can only call this after first save. No behavior change for saved routes. |

---

## Codebase Findings

### api_integrations App

| Item | Finding |
|------|---------|
| Models | `ApiProvider`, `ApiConnection`, `ApiEndpoint` in `backend/django/apps/api_integrations/models.py` |
| `ApiConnection` status choices | DRAFT, CONNECTED, DISCONNECTED, ERROR |
| `ApiEndpoint` key field | SlugField max 64 chars; `data_source` FK currently used for data source linkage |
| `ApiEndpoint.body_template` | TextField; set automatically by this feature — not exposed in route form |
| Auth service registry | `api_integrations/services/` — handles OAuth, static, basic auth credential fetch |
| Endpoint test action | Reused directly via the existing endpoint test action (no new implementation) |

### document_entries App

| Item | Finding |
|------|---------|
| `ExtractionRecord.status` | ExtractionRecordStatus enum; deliverable states = any status NOT IN {NEEDS_REVIEW, REJECTED, SKIPPED, FAILED} |
| `ExtractionRecord.metadata` | JSONField, default=dict; currently stores `trace_id` here |
| `ExtractedFieldValue` | Links to `DocumentTypeField` via FK; `group_item_index` (int, null for non-repeatable) determines repeatable group position |
| Record detail serializer | `ExtractionRecordDetailSerializer._build_fields_list()` already returns the nested grouped shape |
| No existing output route models | Confirmed — `OutputRoute` and `DeliveryAttempt` are net-new |

### DocumentEntryAction (ProviderAction)

| Item | Finding |
|------|---------|
| File | `backend/django/common/actions/document_entry_action.py` |
| `execute()` signature | `async execute(fields: dict, history: History, action: dict)` |
| Deferred | `action_deferred = True` — runs post-reply in the backbone async queue |
| Post-execute hook point | After `DocumentEntryActionService.create_extraction_record()` returns — safe to trigger delivery here |
| `trace_id` source | Passed in `action` dict to `execute()`; stored in `record.metadata["trace_id"]` |

### FastAPI Backbone

| Item | Finding |
|------|---------|
| HTTP client | `httpx` is used in the backbone; same library can be used for outbound delivery HTTP calls |
| Async DB access | `@async_db_operation` decorator on service methods using Django ORM |
| Delivery trigger | Sync Django ORM call wrapping needed when called from async FastAPI context (`sync_to_async`) |

### Field Grouping Integration

The payload builder must walk the `ExtractedFieldValue` rows using the `DocumentTypeField` tree:

| `parent_field` | `group_item_index` | Placement in `data` |
|---|---|---|
| NULL | NULL | `data[codename] = value` |
| `<group field>` (single_object) | NULL | `data[group_codename][codename] = value` |
| `<group field>` (repeatable) | 0, 1, 2… | `data[group_codename][idx][codename] = value` |

Group fields themselves do not have `ExtractedFieldValue` rows — they are structural.

---

## Decisions

### D-001: OutputRoute placement

**Decision**: `OutputRoute` and `DeliveryAttempt` in `document_entries` app.
**Rationale**: These models are document-entry-specific. The `api_integrations` app provides the connection/endpoint infrastructure and must remain generic. Coupling `OutputRoute` into `api_integrations` would add document-entry knowledge to a shared app.
**Alternative rejected**: Separate `output_routes` app — unnecessary overhead for a closely coupled extension of document entries.

### D-002: Endpoint ownership

**Decision**: `OutputRoute.api_endpoint = OneToOneField(ApiEndpoint, on_delete=PROTECT)`.
**Rationale**: Each route owns exactly one endpoint (no reuse per spec clarification). PROTECT prevents accidental orphan deletion. `perform_destroy` in the viewset deletes the endpoint explicitly before the route.
**Alternative rejected**: ForeignKey with unique constraint — same semantics but OneToOneField is more semantically precise.

### D-003: Delivery trigger for status changes

**Decision**: Call `DeliveryService.trigger_auto_delivery(record)` from two hook points: (1) `DocumentEntryAction.execute()` after record creation, (2) `ExtractionRecordViewSet` status-update action after status write.
**Rationale**: These are the only two places where a record's status can transition to deliverable. Using explicit call sites (no signals) keeps the flow visible and debuggable. Signals were rejected because they make call ordering implicit and harder to trace in async contexts.
**Alternative rejected**: Django post_save signal — invisible control flow, harder to test, ordering issues in async FastAPI context.

### D-004: Outbound HTTP library

**Decision**: `httpx` (sync client in Django service layer; async client if called from FastAPI).
**Rationale**: Already used in the backbone; consistent with existing patterns; supports timeout configuration cleanly.
**Alternative rejected**: `requests` — synchronous only, no async path; `aiohttp` — no existing usage.

### D-005: Payload hash algorithm

**Decision**: SHA-256 hex digest of the JSON-serialized payload (UTF-8, sorted keys).
**Rationale**: Industry-standard; sufficient collision resistance for audit purposes; deterministic with sorted keys.

### D-006: Delivery log retention

**Decision**: No retention limit in v1; all `DeliveryAttempt` rows are kept indefinitely.
**Rationale**: The spec and Confluence page define no retention policy. Simplicity over premature cleanup logic. A retention job can be added in v2.

### D-007: `delivery_status` derivation

**Decision**: Computed in the serializer from latest `DeliveryAttempt` + route config; not stored on the record or route.
**Rationale**: Storing derived state introduces update-in-two-places bugs. The derivation logic is simple and cheap (one latest-attempt query per route per record fetch).
