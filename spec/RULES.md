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

All CCCP IRs may include a `headers` table that defines how each segment should be interpreted. This section defines the required structure and conventions.

## Reserved Headers

The following headers are currently **reserved** and their semantics are assumed by default:

* `"H1": "Exclude"` – Reserved for excluded segments (non-decodable)
* `"H2": "NewLine"` – Reserved for newline handling

Encoders and decoders may omit these from the `headers` table unless SDK configuration explicitly requires inclusion (e.g., for IR integrity checks using hashes like MD5). More reserved headers may be introduced in the future.

In the final binary form, **reserved headers are never included**, as their meaning is standardized and assumed.

## Vendor-Specific Transformations

Vendor headers must begin after the reserved headers (`H3` and above). These must follow the exact format:

`<Vendor>:<Name>@<Version>`

For overhead optimization, the full binary may replace long-form vendor headers with numeric IDs for public or private registered vendors. Unregistered vendors must still use the full string format.

SDKs are required to offer an `add_header()` interface or equivalent, allowing applications to insert headers without needing to manually determine the correct segment number.

## Example

```json
"headers": {
  "H1": "Exclude",
  "H2": "NewLine",
  "H3": "Knolbay:DialogueTelugu@1.0.0",
  "H4": "Cccp:Zoo@1.0.1"
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
