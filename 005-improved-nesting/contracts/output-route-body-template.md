# Contract: Output Route body template

Adds an author-written JSON body template to body-carrying Output Routes, replacing the fixed
`{general, email, data}` envelope. Existing non-body or legacy routes keep byte-identical output.

## When shown

- Method ∈ {POST, PUT, PATCH} → body-template field is shown and required.
- Method carries no body (e.g. GET/DELETE) → no body-template field required.

## Template language

Placeholders inside an otherwise-literal JSON document:

| Placeholder | Resolves to |
|-------------|-------------|
| `{{id}}`, `{{email_processed_at}}`, … | provided record metadata variables |
| `{{data}}` | the whole nested extracted-data object |
| `{{data.CargoLines}}` (dotted) | a group-level subtree within the extracted data |

Wrappers, constants, and metadata live in the template — not in a fixed data section.

```jsonc
// Author template
{
  "Message": {
    "DateTime": "{{email_processed_at}}",
    "SenderIdentifier": "VW",
    "Orders": { "Order": { "Reference": "{{id}}", "CargoLines": "{{data.CargoLines}}" } }
  }
}
```

## Save-time validation (FR-014)

| Condition | Result |
|-----------|--------|
| Malformed JSON | **hard error** — save blocked |
| Unresolvable placeholder (typo / unknown key) | **hard error** — save blocked |
| Valid template leaving some configured fields unreferenced | **non-blocking warning** — save proceeds |

## Delivery (FR-012)

On delivery the template renders to the final payload with nesting preserved end-to-end: each
cargo line carries only its own goods; each good its own packing. Output deep-equals the
customer target shape (SC-004). Draft-first preserved — render produces the route output for
human review, never an autonomous send.

## Helper panel (FR-013, US5)

The dialog lists offered metadata fields (with keys) and **all** configured groups & fields,
highlighting each entry when referenced in the body and un-highlighting when not. A whole-object
`{{data}}` reference counts every nested field as referenced.
