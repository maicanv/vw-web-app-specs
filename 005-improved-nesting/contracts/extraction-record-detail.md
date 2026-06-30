# Contract: Extraction record detail (nested render)

Delta over the existing record-detail response: the `fields` tree is emitted **recursively** to
full depth, preserving per-leaf confidence, including nested per-item structure.

## Response shape

```jsonc
{
  "id": "...",
  "fields": [
    {
      "codename": "cargo_lines",
      "type": "repeatable_group",
      "items": [
        {
          "group_item_path": [0],
          "fields": [
            {
              "codename": "goods",
              "type": "repeatable_group",
              "items": [
                {
                  "group_item_path": [0, 0],
                  "fields": [
                    { "codename": "description", "type": "string", "value": "Pallet A", "confidence": 0.93 },
                    {
                      "codename": "packing",
                      "type": "single_object_group",
                      "fields": [
                        { "codename": "weight", "type": "number", "value": "120", "confidence": 0.88 }
                      ]
                    }
                  ]
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

## Rules

- Every level renders correctly; arrays-in-arrays grouped by `group_item_path` (FR-009).
- Each leaf carries its `confidence`.
- Each cargo line's items contain only their own goods (no cross-item leakage) — the serializer
  groups children by the index at this group's depth in `group_item_path`.
- A flat / single-level record's response is byte-identical to today (FR-015).
