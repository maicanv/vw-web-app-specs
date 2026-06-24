# Data Model: Document Type Templates

**Feature**: 004-document-type-templates | **Date**: 2026-06-24

One new model. Fields are stored as a Pydantic-validated JSON column (see research D1), reusing the existing `FieldConfig` union from `apps/document_entries/field_config.py`.

---

## DocumentTypeTemplate

Inherits `BaseModel` (provides `created_at`, `updated_at`, hashid-encoded `id` in API responses).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | bigint | PK, auto | Hashid-encoded in API responses |
| `organisation` | FK → Organisation | **nullable**, CASCADE, `related_name="document_type_templates"` | `NULL` = global; set = org-scoped |
| `name` | varchar(80) | NOT NULL | Display name; 3–80 chars, trimmed |
| `blurb` | varchar(300) | NOT NULL | Short card description |
| `icon` | varchar(50) | blank, default `""` | Icon identifier; frontend maps to a Tabler icon with fallback |
| `description` | text | blank, default `""` | Long description → pre-fills DocumentType.description |
| `instructions` | text | blank, default `""` | → pre-fills DocumentType.instructions |
| `fields` | jsonb (SchemaField) | NOT NULL, default `[]` | `list[TemplateField]`; Pydantic-validated (see below) |
| `is_active` | bool | default `True` | Deactivated templates are hidden from the picker; not deleted |
| `created_at` | datetime | auto | From `BaseModel` |
| `updated_at` | datetime | auto | From `BaseModel` |

**Constraints / indexing**:
- Partial unique constraint on `(organisation, name)` is **not** required — duplicate template names are tolerable across scopes and within a scope (templates are picked by id, not name). Keep it simple; add later only if a real need appears. *(ponytail: skip the constraint, add when duplicate names actually confuse superadmins.)*
- No functional index needed — templates are listed in full (no pagination), filtered only by `is_active` and the global-OR-own-org predicate.

**Validation**:
- `name`: trimmed, 3–80 chars (mirror DocumentType).
- `blurb`: required, max 300.
- `fields`: validated by the `TemplateField` schema below at the `SchemaField` boundary (Pydantic raises on save / serializer validation).

---

## TemplateField (Pydantic schema, not a DB model)

Mirrors `DocumentTypeField`'s authorable shape, minus persistence/runtime concerns (`display_order` is implicit array order; no `id`, no soft-delete). Depth-2 only: children cannot have children — same as `DocumentTypeField`.

```python
class TemplateFieldChild(BaseModel):
    codename: str                       # regex CODENAME_REGEX (reuse existing constant)
    name: str = ""
    config: FieldConfig                 # reuse existing discriminated union
    description: str = ""
    is_critical: bool = False
    confidence_threshold: int | None = None   # 0–100 or None

class TemplateField(TemplateFieldChild):
    children: list[TemplateFieldChild] = []    # only group-type fields populate this
```

**Validation rules (reuse existing constants where possible)**:
- `codename`: matches `CODENAME_REGEX` (`^[a-zA-Z][a-zA-Z0-9_]*$`), max `CODENAME_MAX_LENGTH` (100); unique within the template's full field set (top-level + children).
- Total field count ≤ `MAX_FIELDS_PER_DOCUMENT_TYPE` (50).
- Group configs (`repeatable_group`, `single_object_group`) follow the same bounds as `FieldConfig` (e.g. `max_items` ≤ `MAX_ITEMS_PER_GROUP`).
- Non-group fields MUST NOT have `children`; group fields MAY.

> Reuse the logic in the existing `_validate_fields_payload` helper where practical, or factor the shared rules so both the DocumentType field payload and the template `fields` schema validate identically. Avoid duplicating the codename/dedup/bounds rules.

---

## Relationships

```
Organisation ||--o{ DocumentTypeTemplate : "org-scoped (nullable)"
                    DocumentTypeTemplate : "global when organisation IS NULL"
```

`DocumentTypeTemplate` has **no** relationship to `DocumentType`, `DocumentTypeField`, `ExtractionRecord`, or `Application`. Applying a template is a one-way, create-time copy performed entirely in the frontend form — there is no persistent link back to the source template.

---

## Lifecycle

- **Created / edited / deactivated**: by superadmins in Django admin (or via the seed data migration for the built-in set).
- **Read**: by the create-flow picker (`GET` list / retrieve), filtered to `is_active=True` and visible to the current org.
- **Deactivate ≠ delete**: `is_active=False` hides a template from the picker but preserves the row.
