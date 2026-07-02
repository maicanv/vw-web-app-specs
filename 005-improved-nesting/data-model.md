# Phase 1 Data Model: Improved Nesting

Only one table changes shape (`ExtractedFieldValue`). The Document Type tree is already
recursive and needs **no** schema change — only validation/consumer logic.

## DocumentTypeField (unchanged schema; relaxed rules)

`document_entries/models.py:144`

- `parent_field` — self-FK, `null=True`, `related_name="children"`. **Already arbitrary-depth.**
- `config` — `FieldConfig` discriminated union (value types + `RepeatableGroupConfig` / `SingleObjectGroupConfig`). No new variant.
- **Rule changes (validation only, not schema)**:
  - A group child MAY itself be a group (remove `NESTED_GROUP_ERR`, `serializers.py:329`).
  - Nesting depth ≤ `MAX_GROUP_DEPTH` (5); deeper → blocked with a user-facing message.
  - Value (leaf) fields counted separately; ≤ 150. Group nodes uncapped (depth-bounded only).
  - `RepeatableGroupConfig.min_items` / `max_items` honored at **every** nesting level.

## ExtractedFieldValue (schema change)

`document_entries/models.py:280`

| Field | Before | After | Notes |
|-------|--------|-------|-------|
| `group_field` | FK to nearest group (frozen) | unchanged | still identifies the nearest group ancestor |
| `group_item_index` | `IntegerField(null=True)` | **removed** after backfill | could only address one array level |
| `group_item_path` | — | `JSONField(default=list)` | one 0-based index per **repeatable** ancestor, outer→inner; `[]` for top-level + single-object-group children |
| `confidence` | per-leaf | unchanged | preserved through nested render |

**Examples** (2 cargo lines × 2 goods each, with packing):

```
CargoLines[0].Goods[0].Packing.weight   → group_item_path = [0, 0]
CargoLines[0].Goods[1].weight            → group_item_path = [0, 1]
CargoLines[1].Goods[0].weight            → group_item_path = [1, 0]
Order.reference (top-level value)        → group_item_path = []
Order.Pickup.window (single-object grp)  → group_item_path = []
```

**Migration** (`makemigrations document_entries` — never hand-write):
1. Add `group_item_path` (`default=list`).
2. Data migration: existing rows → `[]` when no repeatable ancestor, else `[group_item_index]`.
3. Remove `group_item_index`.
(No nested-array rows exist in prod yet, so the convert is lossless.)

## OutputRoute (additive)

`document_entries/models.py:~325`

- **New**: a body-template field (author-written JSON with placeholders) used for body-carrying
  methods (POST/PUT/PATCH). When the method carries no body, the template is not required.
- Placeholders (sandboxed Jinja2, `StrictUndefined`): metadata under a single `metadata` root
  (`{{ metadata.id }}`, `{{ metadata.email.received_at }}`, …), whole extracted-data object
  (`{{ data }}`), and group-level references into it rendered with `| tojson`
  (`{{ data.CargoLines | tojson }}`).
- Validity is enforced at **save** (FR-014): malformed JSON / unresolvable placeholder = hard
  error (blocks save); unreferenced configured fields = non-blocking warning.
- Replaces the fixed `{general, email, data}` envelope for these routes. Existing routes keep
  byte-identical output (FR-015).

## Derived: nested delivery payload

Built recursively from the field tree + `group_item_path` (no stored shape):
- value field → transformed scalar (`_apply_field_transform`);
- single-object group → nested dict (omit if optional + all descendants missing — **existing**
  semantics in `_build_data_section`, made recursive);
- repeatable group → list, grouped by the index at *this* group's depth in `group_item_path`,
  recursing into each item's children.
