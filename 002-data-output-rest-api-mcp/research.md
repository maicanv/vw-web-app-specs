# Phase 0 Research: Data Output via REST API

**Feature**: `002-data-output-rest-api-mcp`
**Date**: 2026-05-26

---

## TP VWE-1521 Deviations

The original plan was built without referencing TP VWE-1521. This section documents where the Phase 1 implementation diverges from TP, and the disposition for each divergence.

| # | TP Design | Implementation | Disposition |
|---|-----------|----------------|-------------|
| **DEV-1** | `RouteDelivery` (state tracker) + `DeliveryAttempt` (attempt log) — two models | ~~Single `DeliveryAttempt`~~ | ~~Keep merged approach~~ **Reverted — follows TP**. Both `RouteDelivery` and `DeliveryAttempt` are present in the code and migration. `RouteDelivery.status` is the authoritative stored status; `DeliveryAttempt` logs each attempt. D-007 "derived status" was incorrect. |
| **DEV-2** | `api_endpoint` FK → ApiEndpoint, SET_NULL, nullable | ~~`api_endpoint` OneToOneField → ApiEndpoint, PROTECT~~ | ~~Keep OneToOneField~~ **Reverted — follows TP**. Code uses `WithSingleEndpointMixin` which provides FK + SET_NULL for both `api_connection` and `api_endpoint`, matching TP §2.1. |
| **DEV-3** | `payload_config = SchemaField(schema=PayloadConfig)` Pydantic model | ~~`value_transforms = JSONField(default=dict)`~~ | ~~Keep JSONField~~ **Reverted — follows TP**. Code uses `django-pydantic-field` `SchemaField(schema=PayloadConfig)` per TP §2.2. `PayloadConfig` Pydantic model lives in `payload_config.py`. |
| **DEV-4** | `DeliveryMode.manual` | ~~`DeliveryMode.requires_approval`~~ | ~~Keep requires_approval~~ **Reverted — follows TP**. Code uses `MANUAL = "manual"` per TP. UI label "Manual" vs "Requires Approval" is a display-only concern; enum value follows TP. |
| **DEV-5** | `DeliveryAttempt.is_override` + `override=True` body param | ~~Not implemented~~ | ~~Keep omitted~~ **Reverted — follows TP**. `is_override` field is present on `DeliveryAttempt`. FR-013 updated to match. |
| **DEV-6** | `OutputRoute.display_order = IntegerField(default=0)` | ~~Not implemented~~ | ~~Defer~~ **Reverted — follows TP**. `display_order` field is present on `OutputRoute`. Ordering by `display_order, pk` already in viewset. |
| **DEV-7** | `OutputRoute.payload_format` (json/xml stub) | ~~Not implemented~~ | ~~Defer~~ **Reverted — follows TP**. `payload_format` field is present on `OutputRoute` with `PayloadFormat` choices. |
| **DEV-8** | `body_template = "{{content}}"` set on endpoint creation | `body_template=None` in `perform_create` | **Fixed** (T010b). `perform_create` now sets `body_template="{{content}}"`. |
| **DEV-9** | `WithSingleEndpointMixin` extracted to `api_integrations/mixins.py` | ~~Fields inline on `OutputRoute`~~ | ~~Defer~~ **Reverted — follows TP**. Mixin is implemented in `api_integrations/mixins.py` alongside `ApiIntegrationMixin`. `OutputRoute` inherits from it. |
| **DEV-10** | TP §3.1: manual send via `POST /route-deliveries/{id}/send/` (separate resource) | ~~Plan: `POST /records/{id}/deliver/`~~ | **Reverted — follows TP**. `RouteDelivery` exists as a model with its own ID. `POST /route-deliveries/{id}/send/` is the correct endpoint (TP §3.4). `RouteDeliveryViewSet` to be created in Phase 3. |
| **DEV-11** | TP §3.3: preview is `POST /output-routes/preview/` at collection level — accepts unsaved route config | ~~GET detail=True on saved route~~ | **Reverted — follows TP**. Code already has `@action(detail=False, methods=["post"], url_path="preview")`. Accepts unsaved config. |
| **DEV-12** | TP §3.1: flat URL `/api/v1/document-entries/output-routes/?document_type={id}` | ~~Nested URL~~ | **Reverted — follows TP**. `urls.py` registers `output-routes` as a flat router entry; `document_type` filter via `OutputRouteFilterSet`. |

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
| Output route models | Phase 1 complete — `OutputRoute`, `RouteDelivery`, `DeliveryAttempt` exist in `models.py` and migration `0008` |

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

**Decision**: `WithSingleEndpointMixin` provides `api_endpoint = ForeignKey(ApiEndpoint, SET_NULL, nullable)` per TP §2.1.
**Rationale**: TP is authoritative. SET_NULL means the route row survives endpoint deletion; `perform_destroy` in the viewset explicitly deletes the endpoint after deleting the route. One endpoint per route is enforced by application logic, not a DB unique constraint.
**Supersedes**: Earlier plan to use `OneToOneField + PROTECT` — reverted when TP was consulted (see DEV-2).

### D-003: Delivery trigger for status changes

**Decision**: Two explicit hook points per TP §4.3:
1. `DocumentEntryAction.execute()` → `RouteDeliveryService.initialize_and_dispatch(record)` — creates `RouteDelivery` rows + dispatches auto routes for the first time
2. `ExtractionRecordViewSet` status-update action → `DeliveryService.trigger_auto_delivery(record)` — re-dispatches auto routes on status transitions; `RouteDelivery` rows already exist

**Rationale**: Explicit call sites (no signals) keep the flow visible and debuggable. Signals make ordering implicit and harder to trace in async contexts.
**Alternative rejected**: Django post_save signal.

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

### D-008: SSRF guard utility

**Decision**: Use `validate_safe_url()` from `common/serializers/url_validators.py` for all outbound URL validation — at config time (serializers) and at send time (executor). No separate `ssrf_guard.py` module.
**Rationale**: TP §4.1/§6 is authoritative. `validate_safe_url` covers all required ranges (private, loopback, link-local, metadata IPs, non-HTTPS). The dev/CI bypass is acceptable per TP intent — the same pattern used by `api_integrations` throughout. The previously created `ssrf_guard.py` / `check_url()` was a spec-driven deviation from the TP and has been removed.
**Removed**: `common/utils/ssrf_guard.py`, `SSRFBlockedError`, `check_url()`, `tests/test_common/test_utils/test_ssrf_guard.py`.

### D-009: SSRF guard on update + re-enable

**Decision**: SSRF check also runs on PATCH when `enabled` changes from False → True, even without endpoint field changes.
**Rationale**: A route's provider DNS could have changed since creation. Re-enabling without re-checking could activate a route pointing at a newly-private IP.
**TP coverage**: TP §4.1 only specifies SSRF at send time; the viewset adds an earlier check at config time as defense-in-depth.

### D-007: `delivery_status` is stored, not derived

**Decision**: `RouteDelivery.status` is the authoritative stored status field per TP §2.3/§4.3. Serialisers read it directly.
**Rationale**: TP is authoritative. The delivery service updates `RouteDelivery.status` at every transition (`pending` → `sending` → `sent` / `send_failed`). Re-deriving from `DeliveryAttempt` rows would be redundant and potentially inconsistent.
**Supersedes**: Earlier D-007 "computed in serializer from latest attempt" — incorrect; reverted when TP §2.3 was consulted.

### D-010: `OutputRoute` uses `OrgQuerySetMixin`-equivalent via manual filter

**Decision**: `get_queryset` filters `organisation=self.request.organisation` directly instead of inheriting `OrgQuerySetMixin`.
**Rationale**: Flat URL already covered by org-scoped filter. Inline is explicit and avoids mixin ordering complexity. Same isolation guarantee as TP §3.1's `OrgQuerySetMixin scoped via document_type__organisation`.

### D-011: `ApiIntegrationAccessPolicy` (follows TP)

**Decision**: `OutputRouteViewSet` uses `ApiIntegrationAccessPolicy` directly per TP §3.1.
**Rationale**: TP is authoritative. Earlier plan to create a separate `OutputRouteAccessPolicy` was reverted (see DEV-11). `ApiIntegrationAccessPolicy` already gates write access via `API_INTEGRATIONS_EDIT`.
