# Handling of Newlines During IR Encoding

`['H2', <newline character>]`

The segment header `H2` is **reserved exclusively** for representing line endings in CCCP IR. Vendors must emit line endings explicitly using this segment only.

The `<newline character>` must match the original document's encoding (`\n`, `\r\n`, or `\r`). Vendors must not embed line endings within data segments or LUTs, nor introduce or normalize newlines unless explicitly instructed.

# IR Header Formats

All CCCP IRs must include a `headers` table that defines how each segment should be interpreted. This section defines the required structure and conventions.

## Required Headers

The following headers **must always be present** in every IR:

- `"H1": "EXCLUDE"` – Reserved for excluded segments (non-decodable)
- `"H2": "NEWLINE"` – Reserved for newline handling

These serve as baseline references for minimal decoder compatibility.

## Vendor-Specific Transformations

Additional headers may point to vendor-defined LUTs or decoding functions. These must follow the exact format:

`LUT:<Vendor>:<Name>@<Version>`

## Example

```json
"headers": {
  "H1": "EXCLUDE",
  "H2": "NEWLINE",
  "H3": "LUT:Knolbay:DialogueTelugu@1.0.0",
  "H4": "LUT:CCCP:Zoo@1.0.1"
}
```

## Field Definitions

**Vendor**: The source of the transformation.
- Must use PascalCase (e.g., Knolbay, CCCP, MetaAI)
- ❌ Disallowed: lowercase, UPPERCASE, snake_case, camelCase, etc.

**Name**: The name of the LUT or decoding function.
Must also use PascalCase (e.g., DialogueTelugu, Zoo)
Names may include internal casing or acronyms (e.g., ImagePNG, EncodeTileset)

**Version**: Must follow Semantic Versioning
- Format: @MAJOR.MINOR.PATCH (e.g., @1.0.0, @2.3.1)
