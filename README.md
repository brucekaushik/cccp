# CCCP: Context-Aware Composable Compression Protocol

*Composable Compression & Canonical Packing for Interpretable Encoding Pipelines*

**Version:** `0.1`

## 🧭 Overview

**CCCP** is a meta-compression protocol and intermediate representation (IR) framework designed to enable structured, reversible, and pluggable compression workflows. It provides a vendor-extensible way to interpret and transform data before optional final compression, making it ideal for domain-specific encoding, layered processing, or intelligent substitution.

> CCCP is not just a codec — it's a programmable, stage-based protocol for partial or full transformation of data into compact, structured forms.

## 💡 Core Concepts

### ✅ Structured Interpretation

CCCP splits content into meaningful, vendor-defined segments. Some segments may be substituted via LUTs (lookup tables), others left intact — all tagged and ordered.

### 🔁 Reversibility & Optional Final Compression

CCCP allows progressive encoding: raw → IR → compact textual → full binary. At every stage, decoding remains possible. Final compression is optional (e.g., gzip).

### 🧩 Vendor Extensibility

Third-party vendors can define transformations like LUT substitutions, filters, and custom encoders. CCCP segments reference these via scoped, versioned identifiers.

### 🧠 Context-Aware by Design

While CCCP itself does not interpret content natively, it enables **context-aware compression** by allowing vendor-defined logic to determine how tokens are transformed. For example, a LUT might know that "cat" and "lion" belong to the same semantic category and substitute accordingly.

## 📦 Intermediate Representation Format

A CCCP document in intermediate form consists of:

```json
{
  "version": "0.2",
  "headers": {
    "H1": "EXCLUDE",
    "H2": "LUT:KNOLBAY:zoo@1.0"
  },
  "segments": [
    ["H2", "010010"],
    ["H1", "keyboard"],
    ["H2", "11"]
  ]
}
```

* **`headers`**: Maps symbolic header keys (H1, H2...) to vendor-defined transformations.
* **`segments`**: An **ordered list** of `[HeaderRef, Payload]` pairs.

  * Order is preserved.
  * Duplicate keys are allowed.
  * Allows partial encoding: some values transformed, others left raw.

### Example

**Input text:**

```
cat dog keyboard lion
```

**LUT Used For Substitution**

Assume that this is the LUT of the vendor KNOLBAY for the below Encoded IR:

```json
{
  "lut": "zoo",
  "vendor": "KNOLBAY",
  "version": "1.0",
  "bit_width": 2,
  "lut": {
    " ": "00",
    "cat": "01",
    "dog": "10",
    "lion": "11"
  }
}
```

**Encoded IR:**

```json
{
  "version": "0.2",
  "headers": {
    "H1": "EXCLUDE",
    "H2": "LUT:KNOLBAY:zoo@1.0"
  },
  "segments": [
    ["H2", "010010"],   // 'cat', 'dog'
    ["H1", "keyboard"], // not encoded
    ["H2", "11"]         // 'lion'
  ]
}
```

### Final Textual Form (Optional)

```
CCCPv0.2H2:010010H1:keyboardH2:11
```

### Final Binary Form (Optional)

* Binary packing of header tags and bit sequences
* Suitable for further compression via tools like `gzip`

## 🌍 Use Cases

* Text compression (using domain-specific LUTs)
* Language switching (e.g., `LUT:CCCP:dialogue.telugu@1.0`)
* Game assets (e.g., `LUT:KNOLBAY:encode-tileset@1.0`)
* Embedded devices (sensor logs with shared LUTs)
* Network protocols (compact domain-specific payloads)
* Mixed-data archiving with LUT-switching and fallback zones
* Image compression using vendor-defined functions and tile-based optimization (e.g., `LUT:KNOLBAY:encode-sprite-map@2.1`)
* Database compression for structured columns and BLOBs using contextual LUTs (e.g., compressing log types, product names, or language content with `LUT:CCCP:latin@1.0`)
* Archival storage — layer CCCP IR before binary packing for long-term interpretable storage of logs, telemetry, and structured content
* AI preprocessing pipelines — tokenize and compress multilingual corpora using domain-specific LUTs (e.g., LUT:CCCP:dialogue.telugu@1.0)
* Code compression and syntax normalization (e.g., LUTs for frequent code patterns like `public static`, `console.log`, etc.)
* Configuration file templating (replace common config keys/values in JSON, YAML, etc.)
* Markdown or documentation optimization (tokenize frequent headings or boilerplate formatting)
* Scientific/log data normalization (e.g., compacting standardized units or event types)
* Chat history compression or replay (encode speaker tags, emojis, or frequent phrases)
* Telemetry streams or mobile analytics (minify repeated log events like `button_click`, `page_view`)
* Compiler or AST preprocessing pipelines (use CCCP IR for token stream transforms)



## 🔧 Why Arrays for Segments?

Using a list for `segments` instead of an object allows:

* Order preservation
* Duplicate header usage
* Easy replacement or transformation of individual entries
* Seamless pluggability for vendor pipelines

## 🔄 Decoder Behavior

1. Load and parse the `headers` table.
2. For each `[HeaderRef, Payload]` in `segments`:

   * Match `HeaderRef` (e.g., `H2`) to a transformation spec (e.g., `LUT:KNOLBAY:zoo@1.0`)
   * Decode using the vendor-defined LUT or function.
   * If the header is unknown or the payload is uninterpretable:

     * Fallback to passthrough (raw value retained as-is).

## 🔐 Versioning and Vendorization

Each LUT or function reference must be **namespaced and versioned** for clarity, safety, and interoperability.

- **Vendor**: Identifies the source (e.g., `CCCP`, `Knolbay`)
- **Name**: The LUT or function name (e.g., `zoo`, `dialogue.telugu`, `encode-tileset`)
- **Version**: Semantic versioning (`@1.0`, `@2.3.1`)

This structure enables:
- Safe upgrades and migration paths
- Explicit dependency resolution
- Distributed plugin ecosystems
- Clear failure modes during decoding

## 🔌 Future Direction

* Formal IR schema validation
* CLI encoder/decoder
* Vendor registry & discovery
* Streaming CCCP support
* Web-based playground & inspector

## 🛠️ License

CCCP is experimental and currently under active development. Licensed under MIT.

---

For vendor authorship guides, spec evolution, and examples, see the `spec/` and `examples/` folders. (comming soon)

> Have thoughts, ideas, or want to contribute a LUT or encoder? Open an issue or pull request!

## 🛠 Authors & Contributions

Made by Nanduri Srinivas Koushik — contributions and vendors welcome.

Start compressing with intent, not guesswork.
