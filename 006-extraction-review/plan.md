# Implementation Plan: Extraction Review — Confidence & Control Step

**Branch**: `006-extraction-review` | **Date**: 2026-07-07 (rewritten after source-story revision) | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/006-extraction-review/spec.md`

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Implementation Phases](#implementation-phases)
- [Complexity Tracking](#complexity-tracking)

## Summary

Turn stored confidence data into an enforced, configurable review gate. Admins configure per-document-type review rules in a new **Review Records** wizard step (on/off toggle + OR-combined rules: all records, average below N, sender-domain list, critical field below N, any field below N, specific fields below N chosen via a multi-select field dropdown). Matching records are flagged **Needs Review** with structured reasons and held from delivery until a person Confirms, Corrects, or Rejects. A dedicated **Needs Review** sidebar queue (with pending count and Select-for-review) replaces the list filter. Corrections work on any record (before → after, edited marker, kept confidence) and corrected sent records can be re-sent subject to the route's resend policy. A platform-wide **Audit** section under the Organisation area exposes the extraction-record modification trail. Reviewer assignment adds a dedicated Reviewer role scoped per document type, a shared queue with a hard 5-minute record lock, and an admins/managers fallback for unassigned document types. Two existing delivery settings change: Repeat policy → **Resend policy** (Allow/Block, Block = no re-send after success), and **Delivery Mode removed** (manual = review-all rule). A reviewed (confirmed or corrected) record is delivered with the route's normal method and UID handling, unchanged — Record UID selection and a separate post-review method/UID were dropped from scope as too much work for now.

All previous open questions are resolved (spec D1–D9 + Clarifications); the prior plan's OQ branch seams are gone. Reviewer notifications are out of scope (A7). Design decisions and code grounding live in [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.12 (Django 4.2 + DRF), TypeScript / React 18 (Vite, Mantine, TanStack Query)
**Primary Dependencies**: DRF, django-pydantic-field (`SchemaField` for ReviewConfig), django-auditlog (extend registration to `ExtractedFieldValue`), django-filter, `apps/locks` (AbstractLock), `apps/roles` (dynamic Role + RoleAssignment)
**Storage**: PostgreSQL — 2 new tables (`ReviewerAssignment`, `ExtractionRecordLock`), a renamed column on `OutputRoute` (resend_policy), a new column on `DocumentType` (review_config), a new nullable `organisation` column on auditlog's `LogEntry` (org scoping for the audit API), and two column drops: `OutputRoute.delivery_mode` (preceded by a behaviour-preserving data migration) and `DocumentTypeField.confidence_threshold` (never read at flag time; the review gate lives only in `review_config`)
**Testing**: pytest via `docker compose exec django pytest` (run from `backend/`); no frontend tests unless asked
**Target Platform**: existing Docker Compose stack (Django :8001, client :5173)
**Project Type**: web application, extending `backend/django/apps/document_entries`, `apps/locks`, a new thin `apps/audit`, and `client/src/app/documentEntry` + layout/org area
**Performance Goals**: flag evaluation is O(fields) at extraction write-back; queue + count are single indexed queries through `records_for_reviewer`
**Constraints**: API changes additive except two coordinated renames/removals consumed only by our own client (`repeat_policy`→`resend_policy`, `delivery_mode` dropped) — client ships from the same repo, lossless data migrations; org/tenant scoping on every new query incl. the LogEntry-based audit endpoint; vw-llm-app untouched (confidence already arrives per field)
**Scale/Scope**: ~7 backend endpoint changes (3 new: audit list, reviewers sub-resource, claim; 4 extended: update_status transitions, correct action, queue filter/count, route serializers), 2 new models, 1 SchemaField config, 1 service-layer rule function, UI: 1 new wizard step, 1 new sidebar queue page, 1 new org-area Audit page, deltas on records list/detail and OutputRouteForm

## Constitution Check

*GATE: evaluated against constitution v5.1.2 — PASS (pre-research and post-design).*

- **I. Protect Sensitive Data**: PASS — every new query org-scoped: `ReviewerAssignment` carries `organisation`; queue predicate runs through existing org-scoped querysets; the audit endpoint scopes `LogEntry` directly via a new nullable `organisation` column populated by a `post_log`-signal receiver at write time, and excludes non-org / `is_staff` / null (system) actors so admin-panel and internal edits never leak to org users (tested explicitly). Registering `ExtractedFieldValue` with auditlog stores values already visible in the product; no new PII class.
- **II. Respect the Architecture**: PASS — follows established patterns: `SchemaField` config (as `PayloadConfig`), `AbstractLock` subclass (as `ApplicationLock`), dynamic Role + permission-string access policies, viewset `@action`s. One new pattern: a platform-wide audit read API over auditlog (`apps/audit`) — no prior art in the repo; approach agreed here in the plan phase per constitution II (thin read-only viewset over `LogEntry` with a direct org column + related-model filter; review-lifecycle actions logged explicitly via `LogEntryManager.log_create`, not derived from transitions).
- **III. Test What Matters**: PASS — tests target behaviour: rule evaluation incl. OR combination and confidence-unavailable, hold gating, new transitions (select-for-review, reject re-open), correction + lock conflict (409), resend-policy block, delivery-mode migration outcome, queue visibility + zero-assignment fallback, audit org-scoping and action mapping. No frontend tests (user preference).
- **IV. Ship Incrementally; Enterprise-Grade**: PASS — user stories ship independently in spec priority order. The two non-additive API changes (rename/removal) are consumed only by the same-repo client and land with lossless data migrations in the same deploy; noted here per constitution IV. The delivery-mode drop is preceded by a data migration that preserves manual-route behaviour via the review-all rule.
- **V. Make Failures Visible**: PASS — flag evaluation runs in the existing extraction write-back path already wired to Sentry; delivery failure surfacing unchanged; no new background jobs (notifications dropped).

**Agent autonomy**: no git operations without explicit per-operation request; migrations require user confirmation before running.

## Project Structure

### Documentation (this feature)

```text
specs/006-extraction-review/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions R1–R10 + as-built baseline
├── data-model.md        # Phase 1 — entities, transitions, validation
├── quickstart.md        # Phase 1 — run/verify/test
├── contracts/
│   └── api.md           # Phase 1 — endpoint contracts
└── tasks.md             # Phase 2 (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
backend/django/apps/document_entries/
├── models.py                        # + DocumentType.review_config; + ReviewerAssignment;
│                                    #   OutputRoute: repeat_policy→resend_policy, −delivery_mode;
│                                    #   −DocumentTypeField.confidence_threshold;
│                                    #   auditlog.register(ExtractedFieldValue)
├── review_config.py                 # ReviewConfig pydantic model (new, mirrors payload_config.py);
│                                    #   incl. specific_fields_below (int|None) + specific_fields (list[str]|None)
├── enums.py                         # RepeatPolicy→ResendPolicy(allow|block); −DeliveryMode;
│                                    #   ALLOWED_STATUS_TRANSITIONS += {extracted,rejected,sent,send_failed}→needs_review
├── services.py                      # + compute_overall_confidence, evaluate_needs_review (R2);
│                                    #   deliver(): resend-policy at eligibility; drop MANUAL parking
├── serializers.py                   # + correction, reviewers, lock/threshold surfacing, route setting fields
├── extraction_record_view_set.py    # + correct/, claim (or locks route), review_queue filter + count action
├── document_type_view_set.py        # + reviewers/ sub-resource; review_config in payload
├── filters.py                       # + review_queue predicate (records_for_reviewer)
└── migrations/                      # schema + data migrations (seed avg rule; manual-route→review-all; policy rename)

backend/django/apps/locks/
├── models.py                        # + ExtractionRecordLock(AbstractLock), TTL 5 min
└── base/views.py                    # ResourceLockViewSet split: MFA-free base mixin (acquire/release)
                                     #   + MFA/override layer for the existing application-lock use case

backend/django/apps/audit/           # NEW thin app: LogEntry read API (R6)
├── views.py / serializers.py / urls.py / receivers.py / migrations/
                                     #   org column on LogEntry (post_log receiver), model filter,
                                     #   staff/system-actor exclusion

backend/django/common/constants/permissions.py   # + DOCUMENT_ENTRIES_REVIEW
backend/django/common/access_policy/document_entry_type_access_policy.py  # reviewer statements
backend/django/tests/test_apps/test_document_entries/  # behaviour tests (+ test_apps/test_audit/)

client/src/app/layout/DefaultNav.tsx                   # + Needs Review entry (badge); + org-area Audit entry
client/src/app/documentEntry/
├── documentTypes/DocumentTypeCreatePage.tsx + steps/  # + Review Records step (stepper indices, validateStep)
├── records/NeedsReviewPage.tsx                        # queue page (reuses list components)
├── records/components/RecordFilters.tsx               # − needs_review status option
├── records/ExtractionRecordDetailPage.tsx             # review actions, blocked-send explainer, lock banner,
│                                                      #   select-for-review, threshold highlight (FieldTable)
├── records/FieldTable.tsx                             # inline correction, before→after, edited badge
└── components/OutputRouteForm.tsx                     # resend policy, −delivery mode
client/src/app/audit/AuditPage.tsx                     # org-area audit surface (subsystem filter)
```

**Structure Decision**: extend `document_entries` and the `documentEntry` client module in place; `apps/locks` gains one model per its subclass pattern; the platform-wide audit surface gets its own thin `apps/audit` app because it is deliberately not a document-entries concern (subsystems are pluggable, A8). vw-llm-app untouched.

## Implementation Phases

Build order follows spec story priorities; each phase ships independently.

1. **Review configuration (US1, FR-001..003, FR-032)** — `ReviewConfig` SchemaField (+ `specific_fields_below` / `specific_fields`) + serializer surface + data migration seeding `average_below` from `global_confidence_threshold`; Review Records wizard step (stepper indices, validation, multi-select field dropdown sourced from the wizard's in-progress field list); tests for persistence and defaults.
2. **Flagging (US2, FR-004..007, FR-032)** — `compute_overall_confidence` extraction; `evaluate_needs_review` with OR-combined rules (incl. specific-fields-below-N against the chosen field id set) + missing-critical + confidence-unavailable, structured reasons at extraction write-back; tests per rule and combination.
3. **Hold + review actions (US3, FR-008..010)** — hold falls out of existing `DELIVERABLE_STATUSES` gating → verify + test; Confirm/Reject via existing `update_status`; blocked-send explainer in detail UI.
4. **Needs Review queue (US4, FR-011..013)** — new transitions into `needs_review` (select-for-review, reject re-open); `review_queue` filter + count endpoint; sidebar entry + queue page; remove the list filter option.
5. **Correction (US5, FR-014..016)** — `correct/` endpoint (validation per FieldConfig, `is_manually_edited`, before → after from `raw_value`, never touches status); auditlog registration of `ExtractedFieldValue`; inline edit UI in FieldTable.
6. **Re-send + delivery-setting changes (US6, FR-017/018, FR-028/029)** — resend-policy rename + Block semantics at resend eligibility; delivery-mode removal with behaviour-preserving data migration; OutputRouteForm updates; delivery tests. Reviewed records reuse the route's normal delivery method and UID unchanged.
7. **Audit (US7, FR-019..021)** — `apps/audit` read API over LogEntry: `organisation` column + `post_log` receiver for direct org scoping, related-model filter, staff/system-actor exclusion; explicit `LogEntryManager.log_create` rows at the review-lifecycle moments (flag/confirm/correct/reject/select-for-review/resend); org-area Audit page with model filter; tests incl. org isolation and actor exclusion.
8. **UI signalling (US8, FR-022)** — `applied_threshold` + per-value `below_threshold` in detail serializer; highlight + recommendation banner.
9. **Reviewer assignment (US9, FR-023..027)** — `DOCUMENT_ENTRIES_REVIEW` permission + seeded Reviewer role; `ReviewerAssignment` + `reviewers/` sub-resource; `records_for_reviewer` queue scoping + zero-assignment fallback; `ExtractionRecordLock` (hard, 5 min) with claim/release mounted from the locks app's MFA-free base view mixin (split out of `ResourceLockViewSet` — no cloned lock logic) + 409 guards; assignment removal releases the user's active locks; config UI + lock banner.

Dependencies: 2→1 (rules read config), 3→2, 4→2, 6's post-review bits→2 (needs the flag marker), 7→3/5 (needs actions to audit), 9→4 (scopes the queue). 5 and 8 are independent after 2.

## Complexity Tracking

No constitution violations to justify. The one new pattern (platform audit read API) is declared in the Constitution Check; everything else reuses existing patterns. The two non-additive API changes are same-repo-client-only, with lossless data migrations.
