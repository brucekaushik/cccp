# Rationale for a separate segment for un-encoded data during IR encoding

## Why a dedicated `["H1", PayloadBitlength, Payload]` segment?

1. **Enables Vendorization** - The existence of H1 is what makes vendorization possible. Only un-encoded data (in H1) is eligible for further processing by another vendor. A subsequent vendor may choose to encode that data, replacing the H1 segment with a vendor segment, or splitting it into multiple vendor and exclude segments.

2. **Guarantees fidelity of source data** — Certain portions of the input may not be suitable or necessary for encoding. By preserving these portions in H1 segments, the encoder ensures that the original data can be reconstructed exactly as-is.

3. **Separation of concerns** — Keeping excluded data in a dedicated segment ensures clarity and semantic separation. It becomes easier for tooling to distinguish between encoded and untouched content.

4. **Predictable decoding** — SDKs and tools can reliably interpret H1 segments as raw content, eliminating the need to analyze or guess encoding state.

5. **Support for partial decoding** — Because H1 segments are untouched, they can be decoded or inspected independently, allowing for partial rendering, streaming, or debugging without requiring the full decoding context.

6. **Compression-friendly layout** — Raw or repeated data across files (e.g., boilerplate text) stored in H1 segments is easier for traditional compression algorithms (such as those used in gzip or Brotli) to handle effectively, thanks to consistent formatting.

### Why not treat un-encoded data as part of vendor segment payloads?

While technically feasible, treating un-encoded data as just another part of a vendor segment turns the CCCP IR into an opaque stream—indistinguishable from traditional compression formats. This would undermine many of CCCP's core goals:

* **Loss of structure awareness** — When un-encoded content is embedded within vendor payloads, tools and SDKs lose visibility into which parts were transformed and which were passed through.

* **Inconsistent behavior** — Interoperability suffers when vendors or SDKs handle opaque payloads differently, leading to fragile or incompatible encoding/decoding pipelines.

By contrast, the `['H1', PayloadBitlength, Payload]` structure:

* Enforces clarity of intent,
* Enables deterministic parsing,
* Simplifies structural transformations, and
* Aligns with CCCP’s principle of "composable and inspectable streams."

## SDK Responsibilities

The SDK layer is responsible for ensuring the proper use and handling of H1 segments:

* **Auto-emission** — When a vendor omits encoding certain content, the SDK automatically wraps that data in H1 segments.
* **Enforcement** — Vendors should not embed raw data within their own segment formats unless explicitly allowed.
* **Compression** — The SDK may apply additional compression to H1 content at binary emission time if needed.

In the current POC implementation, the encoder class:

```
cccp/codec/packers/AsciiToJsonIr
```

automatically emits H1 segments for any text not actively encoded by the vendor.

This standardized mechanism ensures that excluded content is handled transparently, predictably, and in a way that supports interoperability and future re-encoding.

# Reasoning: Handling of Newlines During IR Encoding

## Why a dedicated `["H2", PayloadBitlength, LineEnding]` segment?

1. **Preserves fidelity** — some files begin with or rely on specific line endings. This rule ensures exact reproduction.
2. **Vendor flexibility with discipline** — while vendors may choose not to emit this segment directly, they are guaranteed that unhandled line endings will still be emitted in a consistent and predictable way by the SDK.
3. **Human Readability** — if readability is a concern, emitting line endings separately improves the clarity and formatting of the IR.
4. **Binary optimization** — the structure of H2 segments makes them ideal for optimization by the SDK or for downstream compression using tools like gzip or Brotli.
5. **Structural consistency** — clearly distinguishes structural line breaks from data content, removing ambiguity for both humans and automated tooling.

## Why not embed line endings in vendor segments or LUTs?

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

# Rationale for Header Format Design

The structured format of IR headers — especially reserved headers, vendor segments, and PascalCase naming — ensures CCCP remains predictable, extensible, and vendor-safe.

## 🔒 Why Reserved Headers Exist (e.g., `H1`, `H2`)

* `H1` represents **excluded (unencoded) raw data** — such as original text, images, or binary regions.
  Vendors may use this label during encoding when segments are intentionally left uncompressed or uninterpreted.
  Decoders must recognize it to **preserve passthrough fidelity**.

* `H2` represents **newline regions** in the source document.
  Vendors may emit `H2` segments directly, or rely on the SDK to inject them automatically for unhandled line endings.
  These allow newline handling to be **normalized or deferred**, and multiple `H2` segments may exist depending on structure.

Although currently only `H1` and `H2` are reserved, **more reserved headers may be introduced** as the protocol evolves.

SDKs **must offer configuration options** to:

* Control whether reserved headers are explicitly included in the IR (e.g., for hashing or reproducibility).
* Recognize that reserved headers are **implicitly known and excluded** from the final binary to reduce overhead.

> ✅ SDKs must also expose methods like `add_header()` so applications do not need to manually assign or manage header numbers.

Vendor-defined headers must **only begin after the reserved header range**, which protects protocol consistency and prevents accidental collisions.

In the final binary:

* Reserved headers like `H1` and `H2` are **not included**, as their meaning is assumed.
* Publicly registered headers (e.g., `Knolbay:DialogueTelugu@1.0.0`) may be **mapped to compact numeric IDs**.
* Unregistered headers must remain in their full namespaced form to preserve meaning.

Together, these rules:

* Define a **baseline contract** between all encoders and decoders
* Ensure **universal interpretability** of minimal IRs
* Allow safe composition with vendor-defined transformations

## 📐 Why PascalCase for Vendor and Name

* Enforces **visual clarity** and predictable casing in IRs
* Prevents confusion from style clashes (`knolbay`, `KnolBay`, `KNOLBAY`)
* Simplifies tooling, validation, and indexing of LUTs

## 🔄 Why Namespacing + Versioning is Required

* Enables **coexistence** of multiple versions of the same LUT
* Allows **graceful degradation** in case of unknown or unsupported transformations
* Supports **decentralized plugin ecosystems**
* Makes IRs **deterministic and auditable** for long-term reproducibility
