# Implementation Plan: Improved Nesting (Arbitrary-Depth Field Groups)

**Branch**: `005-improved-nesting` | **Date**: 2026-06-30 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/005-improved-nesting/spec.md`

## Table of Contents

- [Summary](#summary)
- [Technical Context](#technical-context)
- [Constitution Check](#constitution-check)
- [Project Structure](#project-structure)
- [Complexity Tracking](#complexity-tracking)

## Summary

Lift the depth-2 grouping limit end-to-end so authors can define arbitrarily nested field
groups (groups-in-groups, repeatable-in-repeatable) to a depth cap of 5, and replace the fixed
`{general, email, data}` delivery envelope with a per-route author-written JSON body template.

The Document Type definition model is **already recursive** (`DocumentTypeField.parent_field`
self-FK, `related_name="children"`), so no new schema for the tree. The work is: (1) stop
rejecting nested groups + add a depth cap; (2) make the four consumers (validate/reconcile,
extraction write-back, review render, delivery builder) walk the tree recursively; (3) replace
the single `group_item_index: int` with an index **path** so nested arrays address per-level
item position; (4) add the Output Route body-template field + placeholder rendering.

## Technical Context

**Language/Version**: Python 3.12 (Django 4.2 + DRF; FastAPI backbone shares the ORM), TypeScript/React 18 (Vite, Mantine); vw-llm-app — Python 3.12 (LangGraph/LangChain, Pydantic v2)
**Primary Dependencies**: DRF serializers, `django-pydantic-field` (FieldConfig discriminated union), TanStack Query, `@hello-pangea/dnd` (Fields reorder), LangChain `with_structured_output(method="function_calling")`, Mistral OCR + `gpt-5.4-mini`
**Storage**: PostgreSQL (shared Django ORM). Migration on `document_entries` for `group_item_path`.
**Testing**: pytest (backend, via docker compose from `backend/`); vw-llm-app pytest. No new frontend tests (per user rule).
**Target Platform**: Linux containers (Docker Compose), GCP.
**Project Type**: Web — Django + FastAPI backend, React frontend, separate vw-llm-app LLM service.
**Performance Goals**: Nested extraction at depth 5 with zero cross-item leakage; deliver ≈10 cargo lines × 4 goods (~8k output tokens, observed ~36.5s) without spurious timeout.
**Constraints**: Byte-identical regression for existing flat/single-level Document Types; draft-first (no autonomous send); MAX_GROUP_DEPTH=5; value-field cap ~150 (groups uncapped); one-pass extraction; retry-once + review-flag on truncation; LLM timeout ≥60s.
**Scale/Scope**: DLG target tree depth 3 (headroom to 5); ~40 goods per document; ≤~150 value fields per Document Type.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Protect Sensitive Data | ✅ PASS | No new credential/PII surface. Body template is author-authored config, org-scoped via existing Output Route access. Placeholders resolve only against the record's own extracted data + provided metadata — no cross-org reach. |
| II. Respect the Architecture | ✅ PASS | Reuses the existing recursive `parent_field` model, `FieldConfig` discriminated union, the `_build_data_section`/`MockPayloadBuilder` pair, and the vw-llm-app side-effect handler. No new patterns invented. Replacing the fixed envelope with a template is a deliberate contract change (see Complexity Tracking). |
| III. Test What Matters | ✅ PASS | Behavior tests: serializer round-trip/reconcile at depth, extraction write-back index-path correctness + padding/capping, delivery deep-equals target shape, regression byte-identical. Skip exhaustive UI unit tests. |
| IV. Ship Incrementally; Enterprise-Grade | ✅ PASS | Five independently testable user stories (P1 author tree, P1 extract, P2 review, P2 deliver, P3 helper panel). **Breaking-change note**: the delivery payload shape changes for body-carrying routes — mitigated by FR-015 byte-identical regression for existing types + migration of `group_item_index`. |
| V. Make Failures Visible | ✅ PASS | Truncation → retry-once → Sentry-captured review flag (existing async error path). Template render errors blocked at save (hard error). |

**Result**: PASS. One justified contract change tracked below; no unjustified violations.

## Project Structure

### Documentation (this feature)

```text
specs/005-improved-nesting/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions (depth cap, field cap, index-path migration, envelope keys, quality gate)
├── data-model.md        # Phase 1 — entities + migration
├── quickstart.md        # Phase 1 — author→extract→review→deliver walkthrough
├── contracts/           # Phase 1 — API deltas (DocumentType fields, OutputRoute body template, record detail)
│   ├── document-type-fields.md
│   ├── output-route-body-template.md
│   └── extraction-record-detail.md
├── checklists/
│   └── requirements.md  # Already created by /speckit-specify
└── tasks.md             # /speckit-tasks output (not created here)
```

### Source Code (repository root)

```text
backend/django/apps/document_entries/
├── field_config.py      # + MAX_GROUP_DEPTH=5; bump value-field cap to ~150 (count leaves only)
├── serializers.py       # remove NESTED_GROUP_ERR (l.329); recurse _validate_group_field; recursive detail
├── models.py            # ExtractedFieldValue.group_item_path (JSONField); group_item_index retained transitionally then dropped
├── migrations/          # makemigrations (NEVER hand-write); data-migrate int → single-element path
├── services.py          # recursive reconcile_fields (l.190) + recursive _build_data_section (l.761) + Mock mirror (l.857); body-template render
└── tests/

vw-llm-app/src/
├── prompts/templates.py # render nested fields tree to arbitrary depth (today stops at 1 group level)
└── actions/document_entry_side_effect_handler.py  # recursive _ExtractionResult; write-back accumulates index path; keep min/max_items + unknown-codename per level

client/src/app/.../documentEntry (types + components)
├── types/documentEntry.ts          # recursive ExtractedFieldEntry / DocumentTypeField
├── FieldsStep.tsx                   # depth-2 block → depth-MAX block; add group-in-group
├── RepeatableGroupSection.tsx / GroupCard.tsx  # recurse into child groups
└── OutputRoute dialog               # body-template textarea (POST/PUT/PATCH) + helper panel + save validation
```

**Structure Decision**: Existing web layout (Django + FastAPI + React + vw-llm-app). No new top-level dirs; all changes land in the established `document_entries` app, the vw-llm-app extraction handler, and the client document-entry feature module.

## Complexity Tracking

| Change | Why Needed | Simpler Alternative Rejected Because |
|--------|------------|--------------------------------------|
| Delivery payload contract change (author-written body template replaces fixed `{general, email, data}` for body-carrying routes) | The DLG/TYSON customer requires the payload in their *exact* nested shape; a fixed envelope cannot produce it. | Reshaping flat values at delivery time was rejected — repeatable-in-repeatable means flat values can't recover which child belongs to which parent. The nesting must be captured at extraction and rendered by an author-controlled template. Existing routes keep byte-identical output (FR-015). |
| `group_item_index: int` → `group_item_path: list[int]` (+ migration) | A single int cannot express "Goods item 2 inside CargoLines item 0"; nested arrays need one index per repeatable ancestor. | Keeping a single int was rejected — it structurally cannot address nested arrays. A separate `ExtractedGroupItem` table was rejected (YAGNI): the tree is reconstructable from `parent_field`; only per-level indices need storing. |
