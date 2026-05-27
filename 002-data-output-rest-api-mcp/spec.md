# Feature Specification: Data Output via REST API

**Feature Branch**: `002-data-output-rest-api-mcp`

**Created**: 2026-05-26

**Status**: Draft

**Source**: [Confluence — [DE] US: Data Output via REST API / MCP Server](https://frontiergroup.atlassian.net/wiki/spaces/VW/pages/293077038)

---

## Overview

Users need to route structured data extracted from Document Types to external systems (ERPs, backends) via REST API endpoints. Customers migrating from other systems have existing integrations expecting specific payload shapes. The platform must let users configure output routes per Document Type — controlling payload format, field value transforms, and delivery rules — so downstream systems receive records in the exact format they already expect, without changes on either side.

MCP Server output is v2 and out of scope for this release.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Configure Output Routes for a Document Type (Priority: P1)

A user configuring a Document Type reaches the Output step in the wizard and can add one or more output routes, each linking the Document Type to an existing API endpoint with delivery settings and value transforms.

**Why this priority**: Core capability — without output routes, no data can be delivered to external systems.

**Independent Test**: Create a Document Type, navigate to the Output step, add an output route pointing to an existing endpoint, save — route is listed and persists on reload.

**Acceptance Scenarios**:

1. **Given** I am editing a Document Type with the required subscription feature, **When** I reach the Output step, **Then** I see a list of configured routes (or an empty state with "Add output route" CTA).
2. **Given** I click "Add output route", **When** I complete the form (label, connection, endpoint), **Then** the route is saved and appears in the list showing label, endpoint key + URL, delivery mode, and enabled status.
3. **Given** I do not have permission to manage integrations, **When** I open the Output step, **Then** the section is hidden or read-only.
4. **Given** I try to save a route without a label or target endpoint, **When** I click Save, **Then** the save is blocked with an inline validation message.
5. **Given** the selected connection is not in Connected status, **When** I try to save, **Then** save is blocked with a message linking to the connection management page.

---

### User Story 2 — Deliver Extracted Data Automatically (Priority: P1)

After a record is successfully extracted, the system automatically delivers the shaped payload to all enabled output routes configured in Auto delivery mode.

**Why this priority**: Auto delivery is the primary value driver — reduces manual work to zero for non-sensitive records.

**Independent Test**: Configure a route in Auto mode, trigger an extraction, verify the delivery log shows a successful attempt and the external endpoint received the correct payload.

**Acceptance Scenarios**:

1. **Given** a record completes extraction with status not Needs Review / Rejected / Skipped / Failed, **When** all enabled Auto routes are triggered, **Then** each route receives the `general / email / data` payload.
2. **Given** delivery succeeds, **Then** the delivery log records: timestamp, route reference, outcome (success), HTTP status, payload hash, attempt number.
3. **Given** delivery fails, **Then** route status becomes Send failed and a manual Retry action is available; other routes and other records are unaffected.
4. **Given** repeat policy is Prevent duplicates, **When** re-send is attempted for an already-sent record, **Then** the send is blocked unless explicitly overridden.

---

### User Story 3 — Review and Manually Approve/Retry Deliveries (Priority: P2)

From the extraction record detail page, a user can see the delivery status per route, approve pending deliveries, retry failed ones, and inspect the delivery log.

**Why this priority**: Provides operational visibility and control for sensitive or failed deliveries.

**Independent Test**: Configure a route in Requires Approval mode, open a processed record, confirm the Delivery section shows Pending approval, click Confirm & Send, verify status changes to Sent and a log entry appears.

**Acceptance Scenarios**:

1. **Given** I open a record detail page, **Then** a Delivery section shows each configured route with label, endpoint URL, and status chip (Not configured / Pending approval / Sending / Sent / Send failed).
2. **Given** a route is in Requires approval mode and has not yet been sent, **Then** a Confirm & Send button appears inline for that route.
3. **Given** a route is in Send failed or Sent (re-send allowed) state, **Then** a Retry / Re-send button appears inline.
4. **Given** I expand a route row, **Then** I see the delivery log: attempt list with timestamp, HTTP status, outcome, and error message; raw payload content is never shown, only the payload hash.
5. **Given** I am viewing the history detail view (Document Entry tab), **Then** a read-only Delivery section shows route labels, endpoint URLs, and final status chips; no send/retry/approve actions are available.

---

### User Story 4 — Payload Preview and Endpoint Testing (Priority: P2)

When creating or editing a route, a user can preview the shaped payload (with value transforms applied) and send a test request to the endpoint.

**Why this priority**: Reduces integration errors before going live.

**Independent Test**: Open a route edit form, adjust a date format transform, verify the preview reflects the new format; click Test, verify the result shows status code, response snippet, and timestamp.

**Acceptance Scenarios**:

1. **Given** I am viewing or editing a route, **Then** a payload preview renders the `general / email / data` envelope with current transform settings using mock/sample data.
2. **Given** I toggle "Preview from selected record", **Then** the preview uses real extracted data from an existing record of this Document Type.
3. **Given** I click Test on a REST route, **Then** the current preview payload is sent to the endpoint; the result shows HTTP status code, response body snippet, and timestamp.
4. **Given** the endpoint uses secret header values, **Then** those values do not appear in test results or delivery logs.

---

### Edge Cases

- What happens when a record's Document Type has no output routes? — No delivery section shown (or section shows empty state).
- What happens when the external endpoint returns a non-2xx response? — Delivery fails, status → Send failed, attempt is logged, no retry.
- What happens when the payload exceeds 1 MB? — Delivery is blocked; route shows an error status.
- What happens when the outbound request times out (> 30 s)? — Request is aborted, delivery attempt logged as failed.
- What happens when the target URL resolves to a private IP / localhost / cloud metadata endpoint? — Request is blocked by the SSRF guard before sending.
- What happens when a route is disabled? — Auto delivery is skipped for that route; manual approve/retry is also blocked.

---

## Requirements *(mandatory)*

### Functional Requirements

**Output Route Management**

- **FR-001**: Users MUST be able to add, edit, enable/disable, and delete output routes on a Document Type.
- **FR-002**: Each output route MUST have: a label, a dedicated API endpoint (created inline via connection + endpoint form as part of route setup — no endpoint reuse across routes), payload format (JSON, v1 only), and delivery mode (Auto or Requires approval).
- **FR-003**: Output routes MUST reuse the existing API integrations layer for connection, endpoint, auth, and credential storage — no re-entry of credentials in the route form. Each route owns its endpoint; endpoints are not shared between routes or Document Types.
- **FR-004**: The Output step MUST provide access to the API connections management screen. In v1, this is surfaced via the "Manage" button inside the API connection picker in the route create/edit form. Full-page navigation — no modal or drawer.
- **FR-005**: Route creation MUST be blocked if label or target endpoint is missing, or if the selected connection is not in Connected status.

**Payload Generation**

- **FR-006**: The system MUST assemble a `general / email / data` JSON envelope for every delivery attempt.
  - `general`: `date`, `time`, `record_id`, `document_type`, `trace_id`.
  - `email`: `id`, `from`, `to`, `subject`, `received_at`.
  - `data`: extracted fields at the top level; single-object groups as nested objects; repeatable groups as arrays of objects.
- **FR-007**: Users MUST be able to configure per-field value transforms per route: date format (default ISO 8601), strip-non-alphanumeric (boolean), missing-value behaviour (null or empty string).

**Delivery Orchestration**

- **FR-008**: In Auto mode, the system MUST deliver to all enabled routes on any transition of a record into a deliverable status (not Needs Review / Rejected / Skipped / Failed). This includes the initial extraction completion AND any subsequent status change (e.g., a reviewer approving a Needs Review record). Idempotency for repeat transitions is governed by the route's repeat policy (FR-013).
- **FR-009**: In Requires approval mode, delivery MUST be held until a user explicitly triggers Confirm & Send from the record detail page.
- **FR-010**: The system MUST log every delivery attempt with: timestamp, route reference, outcome, HTTP status, error message (if any), payload hash, attempt number.
- **FR-011**: Failures on one route MUST NOT affect other routes or other records.
- **FR-012**: There are NO automatic retries; failed deliveries MUST offer a manual Retry action.
- **FR-013**: When repeat policy is Prevent duplicates, re-send to the same record + endpoint MUST be blocked unless the user explicitly overrides it. Override sends are logged via a per-attempt `is_override` flag. The Re-send action for Allow re-send routes is always available without override.

**Security**

- **FR-014**: Outbound requests MUST be blocked for: private IP ranges, localhost, link-local addresses, cloud metadata endpoints (e.g. 169.254.169.254), and non-http(s) protocols (SSRF guard).
- **FR-015**: Maximum outbound payload size: 1 MB. Outbound request timeout: 30 s.
- **FR-016**: Raw document content MUST NOT be persisted in delivery logs — only a payload hash.
- **FR-017**: Secret header values MUST NOT appear in delivery logs or test results.

**UI — Record Detail Page**

- **FR-018**: A Delivery section MUST appear at the bottom of the extraction record detail page, listing each configured route with label, endpoint URL, and status chip.
- **FR-019**: Status chips MUST be one of: Not configured / Pending (auto route, awaiting first delivery trigger) / Pending approval (requires-approval route, awaiting Confirm & Send) / Sent / Send failed. "Sending" is not a state — delivery is synchronous in v1.
- **FR-020**: Expanding a route row MUST reveal the delivery log (attempt list: timestamp, HTTP status, outcome, error message). Payload hash visible; raw content never shown.

**UI — History Detail View**

- **FR-021**: The Document Entry tab in history detail MUST show a read-only Delivery section with route labels, endpoint URLs, and final status chips at snapshot time.
- **FR-022**: No send, retry, or approve actions MUST be available in the history detail view.

### Key Entities

- **OutputRoute**: links a Document Type to an API endpoint; holds label, delivery mode, repeat policy, value transforms per field, enabled flag.
- **DeliveryAttempt**: per-record per-route attempt log; holds timestamp, route reference, outcome, HTTP status, error message, payload hash, attempt number.
- **ValueTransform**: per-field per-route transform config; holds date format, strip-non-alphanumeric flag, missing-value behaviour.
- **ApiEndpoint** _(existing)_: endpoint URL, headers, auth; owned by the `api_integrations` app.
- **ApiConnection** _(existing)_: credential store + status; owned by the `api_integrations` app.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can configure a complete output route (connection → endpoint → transforms) without leaving the Document Type wizard, completing the setup in under 3 minutes.
- **SC-002**: Extracted records are delivered to all enabled Auto routes within 30 seconds of reaching a deliverable status under normal load.
- **SC-003**: Zero extracted field values or raw document content appear in delivery logs — only payload hashes.
- **SC-004**: 100% of outbound requests to private / disallowed addresses are blocked before leaving the platform (SSRF guard verified by test).
- **SC-005**: Users can inspect the full delivery log for any record at any time, including after the record has been archived in history.
- **SC-006**: A failed delivery on one route does not delay or block delivery on any other route for the same or different records.

---

## Clarifications

### Session 2026-05-26

- Q: Does Auto delivery trigger only at initial extraction completion, or also on any subsequent transition to a deliverable status (e.g. after a reviewer approves a Needs Review record)? → A: Any transition to deliverable status triggers Auto delivery; idempotency is handled by repeat policy.
- Q: Can a user pick an existing API endpoint when creating an output route, or must they always create a new one inline? → A: Always create new inline — no endpoint reuse. Each route owns its own dedicated endpoint.
- Q: Does "Manage API connections" open in a new tab, navigate away, or open in a side drawer? → A: Full-page navigation away; user returns via browser back or breadcrumb.
- Analysis fix I1: FR-019 status chip values corrected — "Sending" removed; `pending` (auto route) and `pending_approval` (requires-approval route) are the two distinct pre-delivery states.
- Analysis fix F3: ~~FR-013 "unless explicitly overridden" clause removed~~ — **Reverted 2026-05-27**: override path restored per TP §3.4. `is_override` flag tracks override sends. See proposals.md ALIGN-8.
- Analysis fix F4: T026 test task extended to assert SC-003 invariant (no raw content in DeliveryAttempt fields).

---

## Assumptions

- Field Grouping (prerequisite) is deployed and the extraction pipeline produces grouped payloads before this feature is released.
- The `api_integrations` app provides connection CRUD, OAuth flows, endpoint CRUD, endpoint testing, credential encryption, and header merging — this feature only adds the linking layer.
- Payload format is JSON only in v1; XML and additional formats are v2.
- MCP Server output routes are out of scope for v1.
- Field and group name mapping (e.g. `uti_code` → `ContainerNumber`) and envelope section toggles are out of scope for v1.
- Email trigger metadata is always available as the source; other trigger types (Chat, API) are out of scope for the email section of the payload.
- The Document Type wizard already has a multi-step structure; "Output" is added as a new step.
- Permissions follow the existing RBAC pattern — users without integration management permissions cannot configure or view routes.
