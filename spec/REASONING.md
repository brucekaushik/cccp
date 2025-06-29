## Why Arrays for IR Segments?

Using a list for `segments` instead of an object allows:

* Order preservation
* Duplicate header usage
* Easy replacement or transformation of individual entries
* Seamless pluggability for vendor pipelines

## Reasoning: Handling of Newlines During IR Encoding

### Why a dedicated ['H2', <newline>] segment?

1. **Preserves fidelity** — some files begin with or rely on specific line endings. This rule ensures exact reproduction.
2. **Vendor discipline** — vendors are forced to treat line endings as structural markers, not incidental characters.

### Why not embed line endings in segments?

Although technically possible, this is discouraged because:

1. It can result in extremely long lines, reducing human readability.
2. Full binary decoders (especially multi-vendor ones) may misinterpret vendor intent, leading to incorrect or lossy decoding.

### Why not use `[]` or `['__NL__']`?

1. Header parsing becomes unpredictable — empty or custom segments break the Hn structure.
2. Byte length increases — Hn headers can be tightly mapped to binary tokens (e.g., 2 bits), enabling compression. Mixed segments ruin this.

By keeping newline segments structured as `['H2', ...]`, we preserve both clarity and future binary optimization.

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
