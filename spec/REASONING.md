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

In the current [POC](https://github.com/brucekaushik/cccp-python-poc) implementation, the encoder class:

```
cccp/codec/encoders/AsciiToJsonIr
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

# Minimizing the overhead caused by Segments of IR when encoding to Binary

## Segment Structure

A CCCP IR segment in binary form follows the pattern:

```
["Hn", <PayloadBitlength>, <Payload>]
```

where:

* `Hn` — Segment type, where `n` is an integer mapping to a transformation.
* `PayloadBitlength` — Length of the payload, expressed in a compact form.
* `Payload` — The encoded data itself.

---

## 1. Encoding `Hn`

* **Range and size**: With fewer than 255 expected transformations per IR, and realistically fewer than 100 vendor-specific ones per IR, `Hn` fits comfortably in **1 byte**.
* **Potential sub-byte encoding**: If mixed-bit packing is used, `Hn` could be encoded in less than 1 byte. However, for determinism and simplicity, we will assume it to **1 byte** for now.
* **Byte alignment vs. compactness**: Choosing 1 byte ensures byte-aligned parsing across SDKs, avoiding ambiguity.

---

## 2. Encoding `PayloadBitlength`

Once `Hn` is fixed-size, no separator is needed — `PayloadBitlength` can directly follow `Hn`.

**Representation options:**

* **Byte count** — Used when the payload is not LUT-encoded.
* **Symbol width** — Used when LUT encoding is applied (symbol width available from LUT metadata).

**Symbol width encoding choices:**

1. **Binary representation**

   * Allows up to 254 symbols per byte and `00000000` is the delimiter which follows.
   * Normally requires 2 bytes of overhead for symbol width.
   * Can be reduced to **1 byte** if segments are limited to ≤255 symbols — sufficient in most real-world cases.

2. **ASCII representation**

   * Requires a stop indicator (deterministic byte such as a comma).
   * Can be optimized using ASCII hex, but generally incurs a **minimum 2-byte** overhead.

---

## 3. Encoding the `Payload`

* Fundamentally just an integer stream in binary.
* **Byte alignment** makes parsing simpler but can waste up to 7 bits.
* **Variable bit length encoding** can reduce overhead, but increases decoding complexity.

**End detection strategies:**

* **With `PayloadBitlength`** — No delimiter needed; decoder reads exactly the given length.
* **Without `PayloadBitlength`** — Requires a delimiter of the same symbol width, ensuring it doesn’t appear in the payload (possible in LUT-based encoding).

In most cases, **explicit `PayloadBitlength`** is preferred for efficiency.

---

## 4. Responsibility for Overhead Reduction

Reducing per-segment overhead is **the SDK's responsibility**, not the vendor encoder's. However:

* Vendor encoders **must** be aware of the overhead cost per segment (in bits/bytes).
* An SDK API should expose introspection methods so encoders can make informed decisions.

---

**Design Goals:**

* Keep parsing deterministic.
* Minimize wasted bits.
* Ensure flexibility for both LUT and non-LUT encodings.
* Maintain vendor awareness of encoding efficiency without burdening them with low-level bit-packing logic.

# Handling of Exclude segments when encoding IR to Binary

```
["H1", <PayloadBitlength>, <Payload>]
```

These segments are encoded using the strategies mentioned [here](#minimizing-the-overhead-caused-by-segments-of-ir-when-encoding-to-binary).


# Handling of Newline Segments when encoding IR to Binary

```
["H2", <PayloadBitlength>, <Payload>]
```

Newline segments (`H2`) can be optimized during binary encoding by packing them into **1 byte or less** when mixed-bit packing is used.

* **PayloadBitlength** is unnecessary for binary encoding in this case, as it exists in the IR only for structural consistency with other segments.
* **Payload** (i.e., the actual newline character(s)) should be included in the binary header so that the exact same character(s) can be restored during decoding.
* This segment is **self-delimiting**: once the decoder reads the stored newline character(s), it can immediately detect the start of the next segment—whether it's another newline or a completely different type of segment.

> **Note**: For continuous newlines, a new header may be introduced to allow bit-efficient encoding mentioned [here](#minimizing-the-overhead-caused-by-segments-of-ir-when-encoding-to-binary).
