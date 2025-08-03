## Reasoning: Handling of Newlines During IR Encoding

### Why a dedicated `["H2", PayloadBitlength, LineEnding]` segment?

1. **Preserves fidelity** — some files begin with or rely on specific line endings. This rule ensures exact reproduction.
2. **Vendor flexibility with discipline** — while vendors may choose not to emit this segment directly, they are guaranteed that unhandled line endings will still be emitted in a consistent and predictable way by the SDK.
3. **Human Readability** — if readability is a concern, emitting line endings separately improves the clarity and formatting of the IR.
4. **Binary optimization** — the structure of H2 segments makes them ideal for optimization by the SDK or for downstream compression using tools like gzip or Brotli.
5. **Structural consistency** — clearly distinguishes structural line breaks from data content, removing ambiguity for both humans and automated tooling.

### Why not embed line endings in vendor segments or LUTs?

Vendors **may embed line endings** within segments or LUTs if doing so reduces overhead or aligns better with the document structure. For example:

* Including line endings inline may avoid creating additional segments (e.g., a separate newline followed by a fresh segment), which can reduce IR verbosity or improve compression.
* In tightly formatted content (e.g., CSV, Markdown), treating line endings as part of meaningful content may be more natural or efficient.

However, this comes with tradeoffs:

* Embedding line endings can lead to larger or irregular segments, reducing IR readability and limiting segment reuse.
* Multi-vendor or SDK-level tooling may find it harder to interpret such IRs uniformly, especially when transformations or composition across IRs is required.
* Structural clarity is reduced, making transformation, streaming, or partial decoding more complex.

By contrast, using a dedicated segment like `['H2', PayloadBitlength, LineEnding]` promotes:

* Structural clarity and IR normalization,
* Easier transformation pipelines, and
* Centralized optimization or substitution (e.g., reducing to 1 byte in final binary).

**Ultimately**, the choice is left to the vendor, but **standardized line ending handling improves interoperability and toolchain support**.

## Rationale for Header Format Design

The structured format of IR headers — especially the required headers and PascalCase naming — ensures CCCP remains predictable, extensible, and vendor-safe.

### 🔒 Why `H1` and `H2` Are Mandatory

- `H1` represents **excluded (unencoded) raw data** — such as original text, images, or binary regions.
  Vendors **must use** this label during encoding when segments are intentionally left uncompressed or uninterpreted.
  Decoders must recognize it to **preserve passthrough fidelity**.

- `H2` represents **newline regions** in the source document.
  Vendors must insert `H2` segments to explicitly mark these, allowing newline handling to be **normalized or deferred**.
  Multiple `H2` segments may exist depending on the document's structure, and **must not be replaced by vendor-specific headers**.

Together, these headers:
- Define a **baseline contract** between all encoders and decoders
- Ensure **universal interpretability** of minimal IRs
- Allow safe composition with additional vendor-specific headers like `H3`, `H4`, etc.

### 📐 Why PascalCase for Vendor and Name

- Enforces **visual clarity** and predictable casing in IRs
- Prevents confusion from style clashes (`knolbay`, `KnolBay`, `KNOLBAY`)
- Simplifies tooling, validation, and indexing of LUTs

### 🔄 Why Namespacing + Versioning is Required

- Enables **coexistence** of multiple versions of the same LUT
- Allows **graceful degradation** in case of unknown or unsupported transformations
- Supports **decentralized plugin ecosystems**
- Makes IRs **deterministic and auditable** for long-term reproducibility
