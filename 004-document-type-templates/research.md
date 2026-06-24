# Research: Document Type Templates

**Feature**: 004-document-type-templates | **Date**: 2026-06-24

This document records the design decisions resolved before Phase 1. All inputs come from the existing `document_entries` app, the VWE-1297/1298/1328 proposals, and the clarified spec.

---

## D1. How to store a template's fields

**Decision**: Store the template's fields as a single Pydantic-validated JSON column (`fields`) on the `DocumentTypeTemplate` model, reusing the existing `FieldConfig` discriminated union. Do **not** create a second relational `DocumentTypeTemplateField` model.

**Rationale**:
- A template is a **write-once snapshot** authored by superadmins and read as a whole to pre-fill a form. Its fields are never queried by column, joined, reconciled, or linked to `ExtractedFieldValue` — none of the reasons `DocumentTypeField` is relational apply.
- The codebase already stores rich structured config as Pydantic JSON via `django-pydantic-field` `SchemaField` (`DocumentTypeField.config`, `OutputRoute.payload_config`, `FeatureInPlan.control_value`). A JSON `fields` blob is consistent with that established pattern, not a deviation.
- The frontend pre-fill wants the fields as nested JSON anyway (`DocumentTypeFieldFormValues[]`). A relational model would only be re-serialized back into that same JSON.
- Far less code: one model, one migration, no inline-field admin, no reconciliation, no per-row serializer.

**Alternatives considered**:
- *Relational `DocumentTypeTemplateField` mirroring `DocumentTypeField`* — better Django-admin authoring (one inline form per field) but adds a model, migration, inline admin, and a re-nesting serializer for data that is never queried relationally. Rejected for v1.
- *Hardcode templates in Python/fixtures only (no model)* — fails the deactivate + org-scope + admin-edit requirements (FR-007, FR-009, FR-010). Rejected.

**Upgrade path (ponytail)**: if superadmin JSON authoring in Django admin proves painful, promote `fields` to a relational `DocumentTypeTemplateField` + writable inline. The API contract (nested JSON) stays identical, so the frontend is unaffected.

---

## D2. Global vs org-scoped visibility

**Decision**: A nullable `organisation` FK on `DocumentTypeTemplate`. `NULL` = global (visible to all orgs); set = org-scoped (visible to that org only). Picker queryset: `Q(organisation__isnull=True) | Q(organisation=request.organisation)`, filtered to `is_active=True`.

**Rationale**: Mirrors the subscriptions app's global-vs-custom plan pattern (`custom_for_organisations`). `OrgQuerySetMixin` cannot be reused as-is because it filters strictly by a required org FK; a small custom `get_queryset` expresses the global-OR-own-org rule.

**Alternatives considered**: A separate `is_global` boolean plus org FK — redundant; the nullable FK already encodes both states. Rejected.

---

## D3. API surface and permissions

**Decision**: Read-only API — `GET /list` and `GET /retrieve` only — via a `ReadOnlyModelViewSet`. No create/update/delete endpoints (admin-only management satisfies FR-007). Gate on `Permissions.DOCUMENT_TYPES_EDIT` (held by Editor and, implicitly, Admin/Owner), reusing the org-level access-policy pattern of `DocumentEntryTypeAccessPolicy`.

**Rationale**: The only runtime consumer is the create-flow picker, which reads templates. Document Type creation itself requires `document_types:create-delete` (admin-only), so anyone who can reach the picker already clears an `document_types:edit` check. A read-only viewset is the minimum surface.

**Alternatives considered**: Full CRUD viewset gated to superadmin — unnecessary; Django admin already covers management and the spec explicitly scopes UI management out of v1. Rejected.

---

## D4. Where the picker lives (frontend)

**Decision**: A **modal on `DocumentTypesListPage`**, opened by the existing "New Document Type" entry points (header button + table toolbar icon). On selection, navigate to `/document-entries/types/new` passing the chosen template id via React Router location state. The create page reads `location.state.templateId`, fetches the template, and pre-fills the form (name left blank). "Start from scratch" navigates with no state → blank wizard.

**Rationale**:
- The Figma design shows the picker as a distinct "Choose a template" dialog with Cancel / Start-from-scratch actions — a modal, not a wizard step.
- The wizard *is* the `/types/new` route, so "picker before the wizard" maps cleanly to "modal before navigation".
- Router state (not a query param) keeps the URL clean and is sufficient for a one-time pre-fill. Direct navigation to `/types/new` (bookmark/refresh) yields no state → blank wizard, which is the correct default.
- Re-picking after going back remounts the create page → fresh pre-fill, satisfying the "fully replaced" clarification with zero extra logic.

**Alternatives considered**:
- *Picker as step 0 inside the create page* — handles direct navigation but bolts a non-step choice onto the Mantine `Stepper` and diverges from the design. Rejected.
- *Query param `?template=id`* — survives refresh but pollutes the URL and invites stale-state edge cases. Rejected in favour of router state.

---

## D5. Icon handling

**Decision**: Store `icon` as a short string identifier on the template. The frontend maps it to a Tabler icon via a small lookup with a sensible fallback (e.g. a generic document icon) for unknown identifiers.

**Rationale**: Matches the platform's existing icon-by-identifier convention; avoids file upload (explicitly an assumption in the spec). A fallback keeps the picker robust to identifiers the frontend doesn't yet know.

---

## D6. Field-count display on cards

**Decision**: Expose `field_count` as a read-only serializer field = total number of fields in the template tree (top-level + children).

**Rationale**: The design cards show "10 fields", "9 fields". Computing it server-side keeps the card dumb. Total-tree count matches the Figma example where a group ("Line Items (4 fields)") contributes its children to the headline number.

---

## D7. Seeding the initial template set

**Decision**: Seed the built-in templates (Invoice, Purchase Order, Delivery Note, Remittance Advice, Credit Note, Contract, HR Onboarding Form, Bill of Lading) as **global, active** templates via a data migration that is idempotent (`get_or_create` by name + null org).

**Rationale**: The spec assumes templates are seeded, not hardcoded. A data migration ships them with the feature so the picker is populated on deploy, and lets superadmins edit/deactivate them afterwards. Field payloads reuse the validated `FieldConfig` shapes.
