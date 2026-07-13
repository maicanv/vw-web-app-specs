# Quickstart: Extraction Review

## Run

```bash
# From backend/ — full stack
docker compose up -d --build
```

Services: Django http://127.0.0.1:8001, client http://localhost:5173 (`npm start` in `client/`).

## Exercise the feature

1. **Configure**: open a document type → new **Review Records** step → review toggle on (default) → enable rules, e.g. "average confidence below 80", "critical field below 90", and "specific fields below 85" (pick one or more fields from the multi-select dropdown). Optionally add reviewers in the same config (US9).
2. **Trigger**: run an email extraction (or create an `ExtractionRecord` via `factories.py` in a shell) whose confidences match an enabled rule.
3. **Verify flagging + queue**: the record appears under the sidebar **Needs Review** entry with a pending count and the triggering reason; delivery does not fire. The records-list status filter no longer offers "needs review".
4. **Review**: open the record from the queue (a 5-minute hard lock is claimed) → correct a value inline (before → after, edited badge, confidence kept) → **Confirm & Send** or **Reject**. A second user opening the same record sees "being reviewed by X" and gets 409 on edits.
5. **Select for review**: on an unflagged (or rejected) record, use **Select for review** and verify it joins the queue.
6. **Re-send**: correct a value on a `sent` record whose route's **Resend policy** is Allow → re-send → new delivery attempt with the corrected payload, using the route's normal delivery method and UID (unchanged). On a route with Block, the re-send is refused.
7. **Audit**: sidebar Organisation area → **Audit** → subsystem "Extraction Record modification" → verify actor, time, action type, before → after, and the originally-flagged marker for each step above.

## Tests

```bash
# From backend/ — always via docker compose
docker compose exec django pytest tests/test_apps/test_document_entries/ tests/test_apps/test_audit/ -x -q
```

Focus areas: `evaluate_needs_review` rules incl. OR combination and confidence-unavailable (unit); new transitions into needs_review (unit); correction endpoint + lock conflict (integration); resend-policy block (integration); delivery-mode removal data migration (migration test); queue scoping + zero-assignment fallback (integration); audit action mapping + org isolation (unit/integration).

## Migration notes

Three data migrations ship with the schema changes (run via the migration service / `manage.py migrate`, user-confirmed):
1. Seed `review_config.average_below` from `DocumentType.global_confidence_threshold` where set.
2. `repeat_policy` values → `resend_policy` (`allow_resend→allow`, `prevent_duplicates→block`).
3. Before dropping `delivery_mode`: document types owning a `manual` route get `review_config.all_records = True`.
