# Quickstart: Document Type Templates

**Feature**: 004-document-type-templates

How to exercise the feature end-to-end locally once implemented.

## Prerequisites

- Backend running (`docker compose up` from `backend/`), client running (`npm start` from `client/`).
- An org with the `document_entries` subscription feature enabled and a user holding `document_types:create-delete` (admin/owner).

## Seed / verify templates

The built-in global templates ship via a data migration. After `migrate`, confirm they exist:

- Django admin → **Document Type Templates** → you should see Invoice, Purchase Order, Delivery Note, Remittance Advice, Credit Note, Contract, HR Onboarding Form, Bill of Lading, all **active**, all **global** (no organisation).

To create an **org-scoped** template: add a new Document Type Template in admin and set its Organisation. To create a **global** one: leave Organisation blank.

## User Story 1 — create from a template (P1)

1. Go to **Document Entry → Document Types**.
2. Click **New Document Type** → the **template picker** modal appears (before the wizard).
3. Confirm: a grid of cards (icon, name, field count, blurb) plus a pre-selected **Start from scratch** option.
4. Select **Invoice** → continue. The wizard opens with description, instructions, and fields pre-filled; the **name field is blank**.
5. Edit a pre-filled field, add one, remove one — all should be freely editable.
6. Give it a name, save as draft → it appears in the list as a normal Document Type.
7. Re-open the picker, choose **Start from scratch** → wizard opens blank.
8. Back-navigate to the picker and pick a different template → wizard fully reflects the new template (no leftover state).

## User Story 2 — admin management (P2)

1. In Django admin, create a **global** active template with 3 fields → it appears in the picker for every org.
2. Create an **org-scoped** template for Org A → it appears only for Org A users; Org B users do not see it.
3. **Deactivate** a template → it disappears from the picker immediately; the row is not deleted.
4. **Reactivate** it → it reappears.

## API smoke checks

```bash
# List (needs auth cookie/header for a document_types:edit user)
curl -s .../api/v1/document-entries/document-type-templates/ | jq

# Retrieve full template (pre-fill payload)
curl -s .../api/v1/document-entries/document-type-templates/<hashid>/ | jq '.fields'

# Cross-org / inactive id → 404 (no existence leak)
```

## What to verify

- Org isolation: no cross-org template ever appears (SC-004).
- Deactivated templates never appear (SC-003).
- Name is never pre-filled (FR-005).
- Picker reachable in well under 2s (SC-001).
