# Quickstart: Extraction Review

## Run

```bash
# From backend/ — full stack
docker compose up -d --build
```

Services: Django http://127.0.0.1:8001, client http://localhost:5173 (`npm start` in `client/`).

## Exercise the feature

1. **Configure**: open a document type → set `global_confidence_threshold` (e.g. 80). Optionally add reviewers on the same config page (US7).
2. **Trigger**: run an email extraction (or create an `ExtractionRecord` via `factories.py` in a shell) whose field confidences fall below the threshold.
3. **Verify flagging**: record shows **Needs Review** in the records list with the reason; delivery does not fire.
4. **Review**: open the record → correct a value inline (before → after shown, edited badge) → **Confirm & Send** or **Reject**.
5. **Audit**: record detail → history tab shows actor/time/action/changes.
6. **Re-send**: correct a value on a `sent` record → re-send from the delivery section → new delivery attempt appears.

## Tests

```bash
# From backend/ — always via docker compose
docker compose exec django pytest tests/test_apps/test_document_entries/ -x -q
```

Focus areas: `evaluate_needs_review` rules (unit), correction endpoint + lock conflict (integration), queue filter + fallback visibility (integration), audit serializer mapping (unit).

## Open-question flags

The OQ seams (see [research.md](./research.md)) live in:
- `services.py::compute_overall_confidence` (OQ-1), `services.py::evaluate_needs_review` (OQ-2)
- correct endpoint / detail serializer (OQ-3)
- `ReviewerAssignment` model + `records_for_reviewer()` (OQ-5/OQ-6)
- `notify_reviewers()` (OQ-7), `ExtractionRecordLock` enforcement (OQ-8)

After the decision meeting: update `research.md` defaults that changed, then regenerate/trim tasks for the affected OQ tags only.
