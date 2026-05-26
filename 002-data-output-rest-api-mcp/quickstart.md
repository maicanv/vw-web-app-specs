# Quickstart: Data Output via REST API

**Feature**: `002-data-output-rest-api-mcp`

---

## Prerequisites

- Docker Compose running (`docker compose up -d` from `backend/`)
- Frontend dependencies installed (`cd client && npm install`)
- `backend/.env` configured (copy from `.env.example` if not done)
- Field Grouping migrations applied — `parent_field` column on `DocumentTypeField` and `group_item_index` on `ExtractedFieldValue` must exist

## Run After Merging

```bash
# Apply new migrations
docker compose exec django python manage.py migrate

# Verify new tables exist
docker compose exec django python manage.py dbshell
\dt document_entries_outputroute
\dt document_entries_deliveryattempt
```

## Testing Delivery Locally

1. Start all backend services + frontend.
2. Create a Document Type with at least one field and activate it.
3. Create an API Connection pointing to a local test endpoint (e.g. `https://httpbin.org/post`).
4. Add an output route to the Document Type via the Output step.
5. Trigger an email or use the API to create an extraction record.
6. Check the record detail page — Delivery section should show route status.

## Running Tests

```bash
# Backend
cd backend
poetry run pytest django/tests/test_apps/test_document_entries/test_output_routes/ -v
poetry run pytest django/tests/test_apps/test_document_entries/test_delivery_service/ -v

# Frontend
cd client
npm run test -- --reporter=verbose src/app/documentEntry
```

## Key Files

| File | Purpose |
|------|---------|
| `backend/django/apps/document_entries/models.py` | `OutputRoute`, `DeliveryAttempt` models |
| `backend/django/apps/document_entries/services.py` | `DeliveryService`, `PayloadBuilder` |
| `backend/django/common/utils/ssrf_guard.py` | SSRF protection utility |
| `client/src/app/documentEntry/documentTypes/steps/OutputStep.tsx` | Output wizard step |
| `client/src/app/documentEntry/components/DeliverySection.tsx` | Shared delivery status UI |
