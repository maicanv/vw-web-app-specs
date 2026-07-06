# Research: Extraction Review — Confidence & Control Step

**Date**: 2026-07-06 | **Spec**: [spec.md](./spec.md)

## Approach: planning around open questions

The spec carries deliberately open questions (OQ-1..OQ-10) that will be resolved in a team meeting after this plan is written. Instead of blocking, this plan:

1. Identifies for each OQ the **common core** — work that is identical under every answer. This is safe to build immediately.
2. Isolates the **variant delta** behind a narrow seam (one function, one field, one endpoint) so that when the OQ closes, we keep the chosen branch and delete the others from the plan — no rework of the core.
3. States a **default branch** (recommended answer) so tasks can be drafted; tasks touching a variant are tagged with the OQ id.

Legend per OQ below: **Common core** (build now) / **Options** (A/B/C with delta) / **Default** (what the plan assumes until overridden) / **Seam** (where the branch point lives in code).

---

## As-built baseline (verified in code)

- `ExtractionRecord.overall_confidence` = mean of field confidences × 100 (`services.py:510-513`). `needs_review_reason` free-text exists.
- Needs Review currently fires **only** on missing critical fields (`services.py:588-593`); the stored thresholds (`DocumentType.global_confidence_threshold`, `DocumentTypeField.confidence_threshold`) are never read at runtime.
- `ALLOWED_STATUS_TRANSITIONS` (enums.py) already models reviewer outcomes: `needs_review → {extracted, skipped, rejected}`. `update_status` action on `ExtractionRecordViewSet` applies them and triggers auto-delivery on → extracted. **Confirm = transition to extracted; Reject = transition to rejected. These endpoints exist.**
- `ExtractedFieldValue.is_manually_edited` exists, is never set — no edit endpoint exists. `raw_value` / `standardized_value` / `display_value` columns exist per value.
- Delivery (`RouteDelivery`, `DeliveryAttempt`, retry, `ALLOW_RESEND`, `pending_approval`) is complete; re-send of a corrected record needs **no delivery-side changes**, only a corrected payload at send time.
- `django-auditlog` registered on the document_entries models — admin-only surface today; no user-facing audit endpoint.
- `apps/locks` provides `AbstractLock` + `AbstractAccessOverride` (used for `ApplicationLock`) — a ready-made pattern for record locking.
- `apps/roles` + document_entries RBAC constants (tp_vwe1297) provide the permission layer to extend.
- Frontend: records list with status/confidence filters and detail page with field table, confidence display, delivery section already shipped (tp_vwe1460).

---

## OQ-1 — Overall confidence calculation

**Common core**: threshold comparison, flag + reason persistence, UI display. None of it cares how the number is computed.

**Seam**: one pure function `compute_overall_confidence(field_values) -> int | None` in `services.py` (extract the existing inline mean into it). All options share the signature.

| Option | Delta | Notes |
|--------|-------|-------|
| A. Mean (current) | zero — extract-to-function only | Hides one bad field; that risk is mitigated by OQ-2 field-level flagging |
| B. Min of fields | one-line body change | Overall score becomes redundant with field-level flagging |
| C. Critical-weighted | ~10 lines: weighted mean using `is_critical` | Needs a weight constant; slightly harder to explain to users |

**Default**: A (keep mean) — field-level flagging (FR-002) already catches the single-bad-field case, so changing the aggregate adds little. Decision is a body swap either way.

## OQ-2 — Needs Review trigger: any field vs critical only

**Common core**: a per-field threshold check loop over `field_values` at extraction write-back time, producing a structured flag reason ("field X at 42% below threshold 80%"). The loop is needed under both answers.

**Seam**: `evaluate_needs_review(record, field_values) -> str | None` (returns reason or None) in `services.py`, with a single filter predicate inside.

| Option | Delta |
|--------|-------|
| A. Critical fields only | predicate: `f.is_critical` |
| B. Any field | predicate: `True` |
| C. Per-field opt-in (use `DocumentTypeField.confidence_threshold` presence as the signal) | predicate: field has own threshold OR is_critical |

**Default**: A (critical only) — consistent with the existing missing-critical-field rule and avoids flag fatigue on noisy optional fields. B is a one-token change.

## OQ-3 — Corrected value confidence + does correcting clear Needs Review

**Common core** (identical under all answers): correction endpoint saves new value, preserves the original (before → after), sets `is_manually_edited=True`, writes an audit entry. Confirm stays a separate explicit action (`update_status` already exists).

**Seam 1 (display)**: serializer decides what confidence to show for edited values; stored confidence is never mutated (lossless under every option).

| Option | Delta |
|--------|-------|
| A. Keep original score + "edited" badge | zero backend, badge in UI |
| B. Show "n/a" for edited values | serializer returns null confidence when `is_manually_edited` |
| C. Show 100% | serializer override |

**Seam 2 (status)**: does saving corrections auto-transition needs_review → extracted?

| Option | Delta |
|--------|-------|
| A. No — explicit Confirm required | zero; correction endpoint never touches status |
| B. Yes — auto-clear when all flagged fields edited | correction endpoint calls the existing transition after save |

**Default**: A + A — never mutate stored confidence (badge only), keep Confirm explicit. Matches the spec's "the Needs Review status drives the hold" and keeps one path that sends data downstream.

## OQ-4 — Threshold location: **resolved** (document type)

Already stored on `DocumentType.global_confidence_threshold` and per-field. No work beyond enforcement (OQ-1/OQ-2 core).

## OQ-5 — Reviewer routing axis (document type / data source / both / pool)

**Common core** (identical under all axes): a `ReviewerAssignment` row linking a user to *something*, queue filtering in the existing records list ("my queue" filter + pending count), permission grant to act on matching records, fallback visibility for admins/managers, notification fan-out to assigned reviewers.

**Seam**: the FK(s) on `ReviewerAssignment` and one queryset function `records_for_reviewer(user) -> QuerySet`.

| Option | Schema | Queue predicate |
|--------|--------|-----------------|
| A. Document type (working assumption) | FK `document_type` | `record.document_type__reviewer_assignments__user=user` |
| B. Data source / mailbox | FK `data_source` | `record.application/data_source` join |
| C. Both | two nullable FKs + check constraint (exactly one set) | OR of A and B |
| D. Workspace pool | FK `workspace` only | all workspace needs_review records |

**Default**: A — matches the spec's working assumption and the Confluence discussion. Design note: name the model `ReviewerAssignment` (not `DocumentTypeReviewer`) and route all queue logic through `records_for_reviewer()` so B/C/D change the model + one function, not the viewsets/UI. If C becomes likely tomorrow, A's migration extends (add nullable FK) rather than rewrites.

## OQ-6 — Permission model: reviewer role vs capability

**Common core**: an authorization check "can this user act on this record" used by the correct/confirm/reject endpoints and the queue.

**Seam**: one predicate `can_review(user, record) -> bool` next to the existing document_entries access policy.

| Option | Delta |
|--------|-------|
| A. Capability via assignment: having a `ReviewerAssignment` (OQ-5) *is* the grant; admins/managers keep full access via existing RBAC | no roles-app changes at all |
| B. Dedicated Reviewer role in `apps/roles` | new role constant + admin UI + interaction rules with existing roles |

**Default**: A — the assignment row already encodes exactly who reviews what; a separate role would duplicate it. Who can *assign* reviewers: reuse the existing document-type manage permission (same people who edit document type config).

## OQ-7 — Email cadence: per record vs digest

**Common core**: an event emitted when a record enters Needs Review (`notify_reviewers(record)`), in-app unread count derived from the queue itself (no new table needed for in-app: the pending count *is* the notification).

**Seam**: the body of `notify_reviewers` — send now vs enqueue for batch.

| Option | Delta |
|--------|-------|
| A. Immediate email per record | send in the existing async task path when the flag fires |
| B. Batched digest | store pending-notification marker + periodic Celery beat task (cron pattern exists, cf. context-memory cron in data_sources) |
| C. Immediate, throttled (max 1 email per reviewer per N minutes, links to queue) | A + a debounce timestamp per reviewer — debounce pattern already exists (Klaviyo sync debounce) |

**Default**: C — per-record mails spam high-volume mailboxes on day one; a full digest needs schedule UX. Throttled-immediate reuses the existing debounce pattern. A→C→B is an additive path.

## OQ-8 — Lock behaviour: hard vs soft, timeout

**Common core**: a claim on a record while a reviewer edits, surfaced in the UI ("being reviewed by X"), released on action or expiry. `apps/locks.AbstractLock` already implements holder + expiry + override semantics for Applications.

**Seam**: new `ExtractionRecordLock(AbstractLock)` + enforcement level in the correction endpoint.

| Option | Delta |
|--------|-------|
| A. Soft flag: UI warns, server does not reject concurrent saves | lock model + UI banner only |
| B. Hard lock: server rejects corrections from non-holders (409) | A + one guard in the correction endpoint |
| Timeout | either way: TTL field on the lock; default 15 min, refreshed on activity |

**Default**: B with 15-min TTL — the whole point is preventing colliding edits (FR-019); a warning users can ignore doesn't deliver that. `AbstractAccessOverride` gives admins a takeover path for free.

## OQ-9 — Round-robin auto-assignment

Out of scope (spec). No seam needed: it would layer on top of `ReviewerAssignment` without schema change.

## OQ-10 — Bulk-assign reviewers across document types

Additive endpoint over the same model; zero impact on the core design. Defer to tasks only if answered "yes" tomorrow.

---

## Cross-cutting decisions (no OQ attached)

- **Correction persistence — Decision**: store the original in the existing columns, never overwrite. `raw_value` keeps the extracted original; corrections write `standardized_value`/`display_value` and set `is_manually_edited`. Add `original_display_value` snapshot column only if raw→display derivation proves lossy for some field types. *Rationale*: before → after (FR-009) with minimal schema. *Alternative rejected*: a `FieldValueRevision` history table — auditlog already captures per-save diffs; a second history store duplicates it.
- **User-facing audit trail — Decision**: expose the existing `django-auditlog` entries (already registered on `ExtractionRecord` + `ExtractedFieldValue`) through a read-only `audit` sub-endpoint on the record, serialized into actor / timestamp / action / changes, plus explicit action log entries for confirm/reject/re-send (auditlog `LogEntry.changes` on status transitions covers these; the serializer maps them to action types). *Rationale*: FR-012/FR-013 without a new audit model. *Alternative rejected*: bespoke `ReviewAction` table — duplicates auditlog; add later only if auditlog's shape proves too lossy for the UI.
- **"Originally flagged for low confidence" (FR-012)**: `needs_review_reason` is already persisted and never cleared on transition — expose it in the audit serializer; no new field.
- **Missing confidence (FR-004)**: `overall_confidence is None` already occurs (no scored fields); add it as a flag rule in `evaluate_needs_review` with reason "confidence unavailable".
- **Frontend**: extend existing pages (records list, record detail) rather than new routes; correction UI is inline editing in the existing `FieldTable`, review actions surface next to the existing status controls.
