# Feature Specification: Document Type Templates

**Feature Branch**: `VWE-1584-story-document-entry-templates`

**Created**: 2026-06-24

**Status**: Draft

**Input**: User description: "[DE] US: Document Type Templates — template picker for Document Type creation and editing, with superadmin template management via Django admin."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Choose Template When Creating Document Type (Priority: P1)

A workspace user clicks "New Document Type" and is shown a template picker before the wizard opens. They browse a flat grid of pre-configured templates (Invoice, Purchase Order, Contract, etc.) and choose one. The wizard then opens with the description, instructions, and fields pre-filled from the template. The name field is always blank — the user must provide their own. They can edit, add, or remove any pre-filled value before saving.

A "Start from scratch" option is always visible and pre-selected by default. Selecting it opens the wizard with a blank configuration.

**Why this priority**: Core value proposition — reduces setup time for common document types; unlocks all downstream functionality for first-time users.

**Independent Test**: Create a new Document Type via the template picker using the "Invoice" template. Confirm the wizard opens with description, instructions, and fields populated from the template, the name field is blank, all values are editable, and the resulting Document Type is fully usable.

**Acceptance Scenarios**:

1. **Given** a user with permissions to create Document Types, **When** they click "New Document Type", **Then** a template picker screen is shown before the wizard.
2. **Given** the template picker is open, **When** no template is selected, **Then** "Start from scratch" is pre-selected and the wizard opens with a blank form.
3. **Given** the template picker is open, **When** the user selects a template (e.g. Invoice), **Then** the card is highlighted and a detail panel appears alongside the card grid showing the full field tree — top-level fields plus child fields of any groups, indented under their parent; each card also shows an icon, name, field count, and short description.
4. **Given** a template is selected and the user clicks to continue, **When** the wizard opens, **Then** description, instructions, and fields (including types and configurations) are pre-filled from the template, and the name field is blank.
5. **Given** the wizard is pre-filled from a template, **When** the user edits any field, **Then** the change is applied and the pre-filled value is not restored.
6. **Given** up to 20 templates are active, **When** the template picker opens, **Then** all templates are visible in a single view without horizontal scrolling or pagination.

---

### User Story 2 - Superadmin Manages Templates via Django Admin (Priority: P2)

A superadmin uses Django admin to create, edit, deactivate, and scope templates. Each template has a name, short description (blurb), icon, long description, instructions, and a set of pre-configured fields. A template can be global (visible to all organisations) or org-scoped (visible only to a specific organisation). Deactivating a template removes it from the picker without deleting it.

**Why this priority**: Foundational — no templates in the picker without this. Sequenced after the picker in user-facing value, but in practice the model + seed data must land first so the picker has content to display.

**Independent Test**: In Django admin, create a global template "Test Invoice" with 3 fields and activate it. Confirm it appears in the template picker. Deactivate it. Confirm it no longer appears in the picker.

**Acceptance Scenarios**:

1. **Given** a superadmin in Django admin, **When** they create a template with name, blurb, icon, description, instructions, and fields, **Then** the template is saved and visible in the picker (if active).
2. **Given** an active global template exists, **When** any user opens the template picker, **Then** the template appears.
3. **Given** an active org-scoped template exists for Organisation A, **When** a user from Organisation A opens the picker, **Then** the template appears; users from other organisations do not see it.
4. **Given** an active template, **When** a superadmin deactivates it, **Then** it no longer appears in the picker; it is not deleted.
5. **Given** a deactivated template, **When** a superadmin reactivates it, **Then** it reappears in the picker.

---

### Edge Cases

- What happens when all templates are deactivated — picker shows only "Start from scratch".
- What happens when an org-scoped template's organisation is later deleted — template is hidden (handled by org FK constraint or cascade).
- What happens if a user deselects a template and re-selects "Start from scratch" in the picker — wizard opens blank with no pre-fill.
- If a user navigates back from the wizard to the template picker and selects a different template, the wizard state is fully replaced by the new selection — no merge of previous pre-fill.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST display a template picker screen before the Document Type creation wizard opens.
- **FR-002**: Template picker MUST include a "Start from scratch" option that is pre-selected by default.
- **FR-003**: Each template card MUST display: icon, name, field count, and short description.
- **FR-003a**: When a template card is selected in the picker, a persistent detail panel MUST appear (alongside the card grid) showing the full field tree: top-level fields (name and type) plus any child fields of group-type fields, indented under their parent group, so the user can preview the complete structure that will be pre-filled into the wizard.
- **FR-004**: Selecting a template and proceeding MUST pre-fill the wizard with the template's description, instructions, and fields (including field types and all configurations).
- **FR-005**: The name field in the wizard MUST always be blank, regardless of which template is selected.
- **FR-006**: All values pre-filled by a template MUST be fully editable by the user before saving.
- **FR-007**: Templates MUST be creatable and manageable by superadmins via Django admin only (v1).
- **FR-008**: Each template MUST store: name, short description (blurb), icon, long description, instructions, and a set of pre-configured fields with types and configurations.
- **FR-009**: Templates MUST support global visibility (all organisations) or org-scoped visibility (specific organisation).
- **FR-010**: Templates MUST be deactivatable without deletion; deactivated templates MUST NOT appear in the picker.
- **FR-011**: The template picker MUST show global templates plus any org-scoped templates for the current user's organisation, sorted alphabetically by template name. All templates MUST be shown at once with no pagination (v1). The picker layout is designed to accommodate up to approximately 20 templates visible in a single view.

### Key Entities

- **DocumentTypeTemplate**: Represents a reusable configuration blueprint. Stores name, blurb, icon, long description, instructions, visibility scope (global or org-scoped), and active/inactive status.
- **TemplateField**: A field definition attached to a template, mirroring the structure of DocumentTypeField (type, codename, configuration, ordering). Not a live DocumentTypeField — copied on application.
- **Organisation** (existing): Optional FK on DocumentTypeTemplate for org-scoped templates.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can select a template and reach a pre-filled wizard in under 2 seconds from clicking "New Document Type".
- **SC-002**: Template application (pre-filling wizard fields) completes without page reload and produces no data errors.
- **SC-003**: Deactivated templates never appear in any user-facing picker, confirmed immediately after deactivation.
- **SC-004**: Org-scoped templates are only visible to users belonging to the correct organisation — zero cross-org leakage.

## Clarifications

### Session 2026-06-24

- Q: In what order are templates displayed in the picker? → A: Alphabetical by template name.
- Q: If a user goes back from the wizard to the picker and picks a different template, what happens? → A: Wizard state is fully replaced by the new template.
- Q: Should the template picker paginate? → A: No pagination in v1 — all templates shown at once.

> **Scope change (post-clarification):** The "Load from template" edit flow (originally User Story 2 — Replace/Merge/Cancel into an existing Document Type) has been **dropped from this feature**. Templates apply only at create time. Earlier clarifications about merge field ordering and loading into an active Document Type are therefore void.

### Session 2026-06-24 (amendment)

- Templates apply at create time only — loading a template into an existing Document Type is fully out of scope and will not be revisited in this iteration.
- The picker layout is designed for up to ~20 templates visible in a single window with no pagination.
- When a template is selected in the picker, the full field list MUST be shown so users can preview the template's content before committing to it.
- Q: Where does the full field list appear when a template is selected? → A: Side panel — a persistent detail panel alongside the card grid, populated when a card is clicked.
- Q: For group-type fields in the side panel, should child fields be shown? → A: Yes — child fields listed indented below their parent group so the full field tree is visible.

## Assumptions

- Existing Document Type creation wizard (multi-step: Name & Description → Fields → Review) remains unchanged; the template picker is an additional step inserted before it.
- The icon field on a template accepts a standard icon identifier consistent with the existing icon system in the platform (not an arbitrary upload).
- Template field configurations follow the same `FieldConfig` discriminated union already defined for `DocumentTypeField` (string/number/boolean/currency/date/enum and group variants from VWE-1328/VWE-1496).
- The initial set of built-in templates (Invoice, Purchase Order, Delivery Note, Remittance Advice, Credit Note, Contract, HR Onboarding Form, Bill of Lading) will be seeded via a data migration or fixture — they are not hardcoded in code.
- Access control: only users with the existing "create/edit Document Type" permission see the template picker; no new permission is required.
- Template management (create/edit/deactivate) is superadmin-only in v1 — org admins do not have access.

## Out of Scope

- **Load template into an existing Document Type** (the "Load from template" button, Replace/Merge confirmation, and merge-dedup logic). Templates apply at create time only. May be revisited in a future iteration.
- Self-service template creation by org admins from the UI.
- Category grouping, search, and filtering in the picker.
- Saving an existing Document Type as a template.
- Template versioning / change history.
