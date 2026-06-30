# Quickstart: Improved Nesting (author → extract → review → deliver)

End-to-end walkthrough for the DLG/TYSON shape (`Order → CargoLines[] → Goods[] → Packing`).

## 1. Author the nested Document Type (US1)

1. Document Types → create → **Fields** step.
2. Add a repeatable group `CargoLines`. Inside it, add a repeatable group `Goods` (now allowed —
   group-in-group). Inside `Goods`, add value fields + a single-object group `Packing`.
3. Save. Reopen — the tree round-trips. Re-parent / delete a node and re-save to confirm reconcile.
4. Limits: nesting blocked beyond depth 5; > 150 value fields rejected (groups don't count).

## 2. Run extraction (US2)

- Send a document with 2 cargo lines × 2 goods (each with packing) through the EMAIL pipeline.
- Each good is stored against its own cargo line via `group_item_path` (`[0,0]`, `[0,1]`,
  `[1,0]`, `[1,1]`) — zero cross-item leakage.
- Absent optional groups → no invented values.
- Truncated output → one retry, then review flag (no silent partial).

## 3. Review (US3)

- Open record detail → the full nested structure renders at every level with per-leaf confidence.

## 4. Configure delivery (US4 + US5)

- Output Route → choose POST → a JSON body-template field appears.
- Write the customer's exact shape using `{{id}}`, `{{email_processed_at}}`, `{{data}}`,
  `{{data.CargoLines}}`.
- Helper panel lists metadata + all configured fields, highlighting referenced ones.
- Save: malformed JSON / bad placeholder = blocked; unreferenced fields = warning only.

## 5. Deliver

- Delivered payload deep-equals the target shape — each cargo line owns only its own goods; each
  good its own packing. Draft-first preserved.

## Verify (tests — run via docker compose from `backend/`)

```bash
# backend
docker compose run --rm django pytest apps/document_entries/tests -k "nested or reconcile or build_data or regression"
# vw-llm-app
docker compose run --rm <llm-service> pytest -k "extraction and (nested or group_item_path)"
```

1. **Serializer**: full TYSON tree validates, round-trips, reconciles on PATCH; depth>5 rejected; 150-leaf cap holds; groups uncapped.
2. **Extraction write-back**: `group_item_path` values correct (`[0]`,`[1]`,`[0,0]`,`[0,1]`,`[1,0]`…); padding/capping per level.
3. **Delivery**: data section deep-equals `TYSON.json` shape.
4. **Regression**: flat / single-level type → byte-identical output.
5. **Manual**: author TYSON type, run extraction end-to-end, diff delivered payload vs `Downloads/TYSON.json`.
