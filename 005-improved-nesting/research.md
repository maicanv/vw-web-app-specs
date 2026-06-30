# Phase 0 Research: Improved Nesting

All Technical-Context unknowns resolved below. No `NEEDS CLARIFICATION` remain.

## R1 — Maximum group depth

- **Decision**: `MAX_GROUP_DEPTH = 5` (a new constant in `field_config.py`, replacing the implicit depth-2 bound).
- **Rationale**: DLG/TYSON needs depth 3 (`Order → CargoLines[] → Goods[] → Packing`). The stress test extracted depth-5 arrays-in-arrays at 100% structural correctness with zero cross-item leakage, so 5 is safe headroom without inviting unbounded prompt growth.
- **Alternatives**: Unbounded (rejected — no guardrail against prompt bloat / pathological trees); depth 3 (rejected — no margin, a single future customer change forces a code edit).

## R2 — Value-field cap vs. group nodes (code discrepancy)

- **Finding**: Code today is `MAX_FIELDS_PER_DOCUMENT_TYPE = 50` (`field_config.py:8`), counted across the **whole tree** (`_count_fields_tree`, `serializers.py:375`). Spec FR-003 wants ~150 **value (leaf)** fields with **group nodes uncapped** (bounded only by depth).
- **Decision**: Raise the leaf cap to **150** and change counting to count **value fields only**; group nodes (single-object / repeatable) are excluded from the cap and bounded solely by `MAX_GROUP_DEPTH`.
- **Rationale**: A deep tree spends nodes on structure; counting structural nodes against the same 50 budget would block legitimate nested shapes well before 150 real fields. 150 matches the observed DLG field volume with margin.
- **Alternatives**: Keep 50 tree-wide (rejected — DLG tree exceeds it once groups count); cap groups too (rejected — depth is the meaningful structural limit, not node count).

## R3 — Nested per-item indexing: `group_item_index` → `group_item_path`

- **Decision**: Replace `ExtractedFieldValue.group_item_index: IntegerField` (`models.py:311`) with `group_item_path: JSONField(default=list)` holding one 0-based index per **repeatable** ancestor, ordered outer→inner. Top-level fields and single-object-group children = `[]`. Keep `group_field` (frozen nearest-group pointer) as-is.
- **Rationale**: A single int cannot say "Goods item 2 inside CargoLines item 0". A path list is the minimal store; the full nesting is reconstructable by walking `parent_field`, so only per-level indices need persisting.
- **Migration**: `makemigrations document_entries` (never hand-write). Data-migrate existing rows: `null`/value `n` → `[]` if no repeatable ancestor, else `[n]`. Drop `group_item_index` after backfill (cleanest; no transitional dual-write — there are no nested-array rows in prod yet, so the convert is lossless).
- **Alternatives**: New `ExtractedGroupItem` relational table (rejected — YAGNI; add only if cross-item queries later demand it). Keep both columns transitionally (rejected — dual-write complexity for zero benefit at current data scale).

## R4 — Static envelope keys (TYSON `SenderIdentifier`, `Message.DateTime`)

- **Decision**: Static wrapper keys, constants, and metadata live **in the author-written body template**, not in a fixed data section. The template references the extracted-data object via `{{data}}` / `{{data.CargoLines}}` and provided metadata via `{{id}}`, `{{email_processed_at}}`, etc. Constant strings are typed directly into the template JSON.
- **Rationale**: This is exactly why the fixed `{general, email, data}` envelope is being replaced (FR-010/011) — the customer's contract dictates the wrapper, so the wrapper belongs to the route, authored per target.
- **Alternatives**: Author-configured envelope fields in the Document Type (rejected — conflates extraction schema with delivery shape; the same extraction may feed differently-shaped routes); computed `general` block (rejected — that *is* the legacy envelope being removed).

## R5 — Extraction strategy & timeout

- **Decision**: One-pass extraction (nested tree rendered inline to the model); no pre-chunking at DLG scale (~40 goods ≈ 8k output tokens, no truncation). On truncated/invalid output: retry once, then flag the record for review (no silent partial persist). Raise the LLM call timeout for this flow to **≥60s** (nested docs observed ~36.5s vs. the 20s default).
- **Rationale**: Staged (outer-array-then-children) extraction gave no structural benefit at +30–40% tokens and 6–11 calls in the stress test. The largest in-scope docs are the ones most likely to spuriously time out, so the timeout must clear the observed worst case with margin.
- **Alternatives**: Staged extraction (rejected — cost with no accuracy gain); keep 20s timeout (rejected — spurious timeouts on exactly the documents motivating the feature).

## R6 — Pre-rollout quality gate

- **Decision**: Reuse the VWE-1496 multi-document fixture-gate pattern: a fixture set asserting correct cargo-line count, correct goods-per-line, zero cross-item leakage (SC-002), and zero invented optional groups (SC-003) before rollout.
- **Rationale**: Confidence is per-leaf and a structurally-wrong grouping can still score high (spec Edge Cases), so a structural-correctness fixture gate — not confidence — is the acceptance signal.
- **Alternatives**: Per-field accuracy only (rejected — misses array-boundary corruption that poisons every nested child).

## Recursive-model notes (implementation guardrails)

- **Pydantic v2**: `_ExtractionResult` / `_RepeatableGroupResult` / `_SingleObjectGroupResult` already exist in `document_entry_side_effect_handler.py`; make them mutually recursive (forward refs + `model_rebuild()`). `with_structured_output(method="function_calling")` already in use; the existing `dict[str, Any]` per-subtree escape hatch (handler l.434) covers arbitrary depth without exploding the tool schema.
- **DRF**: `DocumentTypeFieldSerializer` already nests via `source="children"`; verify it recurses without a hard depth stop. `_validate_group_field` (l.288) and `reconcile_fields` (services.py:190) must recurse rather than flatten at one level.
- **Write-back**: carry the existing `min_items` padding / `max_items` capping / unknown-codename handling (handler l.532–590) at **every** level, accumulating the index path down each repeatable ancestor.
