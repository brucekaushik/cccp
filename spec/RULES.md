# Handling of Newlines During IR Encoding

Line endings in CCCP IR may or may not be explicitly handled by vendor encoders. If **not encoded** by the vendor, the SDK must emit them using a **dedicated newline segment** in the following format:

```
["H2", <PayloadBitlength>, <LineEnding>]
```

* `H2` is **reserved exclusively** for line endings in CCCP IR.
* `<LineEnding>` reflects the original source encoding (e.g., `\n`, `\r\n`, `\r`).
* `<PayloadBitlength>` is the bit-length of the encoded representation of the line ending.

Vendors **must not embed** line endings within LUT mappings or data segments unless the SDK explicitly allows it.

The actual enforcement is handled by the SDK. For example, in the POC, the encoder class:

```
cccp/codec/packers/AsciiToJsonIr
```

automatically emits all line endings using the `H2` segment format and **prevents vendors from encoding newlines manually**.

➡️ In the **Final Binary**, newline segments may be **compressed** further — potentially to a **single byte** or less — depending on the SDK and binary encoder logic.

This separation ensures consistent and predictable handling of line endings across different vendors and platforms.

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
