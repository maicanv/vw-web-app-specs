# Implementation Plan: Extraction Review — Confidence & Control Step

**Branch**: `006-extraction-review` | **Date**: 2026-07-06 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/006-extraction-review/spec.md`

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Open-Question Branch Strategy](#open-question-branch-strategy)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Implementation Phases](#implementation-phases)
- [Complexity Tracking](#complexity-tracking)

## Summary

Turn the already-stored confidence thresholds into an enforced review gate: flag low-confidence extraction records as Needs Review with a reason, hold them from delivery until a reviewer confirms/corrects/rejects, let users correct values on any record (before → after, manually-edited marker) and re-send corrected payloads, expose a user-facing audit trail, and (dependent story) assign reviewers per document type with a shared, locked queue and notifications.

Most of the machinery exists: status transitions (`needs_review → extracted/rejected/skipped` via `update_status`), delivery with retry/re-send, auditlog registration, per-value confidence, and the `is_manually_edited` column. The build is: flag-rule evaluation at extraction write-back, a correction endpoint, an audit read endpoint, serializer/UI surfacing, and the `ReviewerAssignment` + `ExtractionRecordLock` models.

The spec's open questions are resolved tomorrow; this plan isolates each behind a narrow code seam with a stated default so the chosen branch is kept and the rest deleted — see [Open-Question Branch Strategy](#open-question-branch-strategy) and [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.12 (Django 4.2 + DRF), TypeScript / React 18 (Vite, Mantine, TanStack Query)
**Primary Dependencies**: DRF, django-auditlog (already registered on the affected models), django-filter, `apps/locks` (AbstractLock), existing async task path (RabbitMQ) for notifications
**Storage**: PostgreSQL — 2 new tables (`ReviewerAssignment`, `ExtractionRecordLock`), no destructive migrations
**Testing**: pytest via `docker compose exec django pytest` (run from `backend/`); no frontend tests unless asked
**Target Platform**: existing Docker Compose stack (Django :8001, client :5173)
**Project Type**: web application (Django backend + React client), extending `backend/django/apps/document_entries` and `client/src/app/documentEntry`
**Performance Goals**: flag evaluation is O(fields) at extraction write-back — negligible; queue filter is one indexed join
**Constraints**: no breaking API changes (additive endpoints/fields only); org/tenant scoping on every new query; LLM-side (vw-llm-app) is untouched — confidence already arrives per field
**Scale/Scope**: ~6 backend endpoints (2 new, 4 extended), 2 models, 1 service-layer rule function, UI deltas on 2 existing pages + document-type config

## Open-Question Branch Strategy

Per-OQ option matrices with common core, deltas, defaults, and code seams live in [research.md](./research.md). Summary of seams:

| OQ | Topic | Seam (single change point) | Default until meeting |
|----|-------|---------------------------|----------------------|
| OQ-1 | Overall confidence formula | `compute_overall_confidence()` in services.py | keep mean |
| OQ-2 | Flag on any field vs critical | predicate in `evaluate_needs_review()` | critical only |
| OQ-3 | Edited-value confidence / auto-clear | detail serializer + correct endpoint | keep score + badge; explicit Confirm |
| OQ-5 | Reviewer routing axis | `ReviewerAssignment` FK + `records_for_reviewer()` | document type |
| OQ-6 | Role vs capability | `can_review()` predicate | assignment = capability |
| OQ-7 | Email cadence | `notify_reviewers()` body | immediate, throttled per reviewer |
| OQ-8 | Lock hardness/timeout | guard in correct endpoint; TTL constant | hard, 15 min |
| OQ-9 | Round-robin | none needed (layers on later) | out of scope |
| OQ-10 | Bulk-assign | additive endpoint | defer |

Rule for task generation (`/speckit-tasks`): tasks touching a seam carry the OQ tag; everything else is common core and unblocked. After the meeting, update the Default column + research.md, drop non-chosen deltas, and only OQ-tagged tasks get revised.

## Constitution Check

*GATE: evaluated against constitution v5.1.2 — PASS (pre-research and post-design).*

- **I. Protect Sensitive Data**: PASS — all new queries org-scoped (ReviewerAssignment carries `organisation`; queue predicate filters through existing org-scoped querysets). Audit endpoint exposes only same-org records via the existing access policy. No new PII stores; extracted values already exist.
- **II. Respect the Architecture**: PASS — extends `document_entries` app patterns (viewset `@action`s, service-layer rules, decorators/exceptions map), reuses `apps/locks` AbstractLock and auditlog instead of inventing parallel mechanisms. New pattern introduced: none.
- **III. Test What Matters**: PASS — tests target behaviour: flag rules, correction + lock conflicts, queue visibility/fallback, audit mapping. No frontend tests (user preference).
- **IV. Ship Incrementally; Enterprise-Grade**: PASS — user stories are independently shippable in spec priority order; all API changes additive (no versioning needed); OQ seams exist for decision stability, not speculative abstraction (each is a single function/field, justified by the scheduled decision meeting).
- **V. Make Failures Visible**: PASS — flag evaluation and notifications run in the existing async paths already wired to Sentry; delivery failure surfacing unchanged.

**Agent autonomy**: no git operations without explicit per-operation request; migrations require user confirmation before running.

## Project Structure

### Documentation (this feature)

```text
specs/006-extraction-review/
├── plan.md              # This file
├── research.md          # Phase 0 — OQ option matrices + as-built baseline
├── data-model.md        # Phase 1 — entities, transitions, validation
├── quickstart.md        # Phase 1 — run/verify/test
├── contracts/
│   └── api.md           # Phase 1 — endpoint contracts
└── tasks.md             # Phase 2 (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
backend/django/apps/document_entries/
├── models.py                        # + ReviewerAssignment
├── enums.py                         # (transitions unchanged)
├── services.py                      # + compute_overall_confidence, evaluate_needs_review, notify_reviewers
├── serializers.py                   # + correction, audit, reviewers, lock/threshold surfacing
├── extraction_record_view_set.py    # + correct/, audit/, claim/, my_queue filter, queue count
├── document_type_view_set.py        # + reviewers/ sub-resource
├── filters.py                       # + my_queue predicate
└── migrations/                      # + ReviewerAssignment

backend/django/apps/locks/
└── models.py                        # + ExtractionRecordLock(AbstractLock)

backend/django/tests/test_apps/test_document_entries/
└── (new test modules per behaviour area)

client/src/app/documentEntry/
├── records/ExtractionRecordListPage.tsx   # my-queue filter + pending count
├── records/ExtractionRecordDetailPage.tsx # review actions, blocked-send explainer, audit tab, lock banner
├── records/FieldTable.tsx                 # inline correction, before→after, edited badge, threshold highlight
└── documentType wizard/config             # threshold input surfacing + reviewers management (US7)
```

**Structure Decision**: extend the existing `document_entries` Django app and `documentEntry` client module in place; the only cross-app touch is one model in `apps/locks` following its established subclass pattern. vw-llm-app is untouched.

## Implementation Phases

Build order follows spec story priorities; each phase ships independently.

1. **Flagging (US1)** — extract `compute_overall_confidence` [OQ-1], add `evaluate_needs_review` with the four rules (missing-critical kept, unavailable, overall, per-field [OQ-2]) at extraction write-back; structured reasons; tests.
2. **Hold + review actions (US2)** — hold already falls out of existing delivery gating (`DELIVERABLE_STATUSES` excludes needs_review) → verify + test; surface Confirm/Reject (existing `update_status`) and blocked-send explanation in the detail UI.
3. **Correction (US3)** — `correct/` endpoint (validation, `is_manually_edited`, before → after from `raw_value`) [OQ-3]; inline edit UI in FieldTable.
4. **Re-send (US4)** — verify corrected payload flows through existing re-send; add test; UI affordance already exists.
5. **Audit (US5)** — `audit/` endpoint mapping auditlog entries to actions; history tab in detail page.
6. **UI signaling (US6)** — `applied_threshold` + `below_threshold` in detail serializer; highlight + recommendation banner.
7. **Reviewer assignment (US7)** — `ReviewerAssignment` model + `reviewers/` sub-resource [OQ-5, OQ-6]; `my_queue` filter + count; `ExtractionRecordLock` + `claim/` [OQ-8]; `notify_reviewers` [OQ-7]; config UI + queue UI.

Phases 1–6 have no dependency on tomorrow's meeting beyond seam defaults (OQ-1/2/3 are body-level swaps). Phase 7 is where OQ-5/6 could reshape a model — schedule its implementation after the meeting; its design is still fully plannable now (all options share the queue/permission/notification core).

## Complexity Tracking

No constitution violations to justify. The OQ seams are single functions/fields with a stated default, not speculative abstraction layers.
