# Contract: Document Type nested fields (create / update / detail)

Delta over the existing DocumentType endpoints. The field payload is the existing recursive
`fields[]` shape (`children[]` per node) — this contract only relaxes validation and lifts depth.

## Request — nested `fields[]` (create & update)

A group node's `children[]` may now contain group nodes (single-object or repeatable), to depth 5.

```jsonc
{
  "fields": [
    {
      "codename": "cargo_lines",
      "name": "Cargo Lines",
      "config": { "type": "repeatable_group", "min_items": 1, "max_items": 200 },
      "children": [
        {
          "codename": "goods",
          "name": "Goods",
          "config": { "type": "repeatable_group", "min_items": 1, "max_items": 200 },
          "children": [
            { "codename": "description", "name": "Description", "config": { "type": "string" } },
            {
              "codename": "packing",
              "name": "Packing",
              "config": { "type": "single_object_group", "optional": true },
              "children": [
                { "codename": "weight", "name": "Weight", "config": { "type": "number" } }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

## Validation rules

| Rule | Outcome on violation |
|------|----------------------|
| Group child may be a group (nested groups allowed) | (was rejected) now **accepted** |
| Nesting depth ≤ 5 | `400` — clear user-facing message naming the depth limit |
| Value (leaf) field count ≤ 150 | `400` — `MAX_FIELDS_ERR` |
| Group nodes do not count toward the 150 cap | n/a (uncapped except by depth) |
| `repeatable_group` honors `min_items`/`max_items` at every level | `400` on invalid bounds |
| Value field with non-empty `children` | `400` — "Value fields cannot have nested fields." |

## Update semantics (reconcile)

`reconcile_fields` MUST recursively create / update / delete / **re-parent** nodes at any depth.
A saved nested tree MUST round-trip: GET after PATCH returns the same structure.

## Detail response

`DocumentTypeFieldSerializer` emits `children[]` recursively to full depth (no depth stop).
