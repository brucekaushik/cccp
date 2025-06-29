## Segment Decoding Logic During IR Decoding

This section describes how a decoder should process IR segments:

1. Load and parse the `headers` table.
2. For each `[HeaderRef, Payload]` in `segments`:
   - Match `HeaderRef` (e.g., `H2`) to a transformation spec.
   - Decode using the vendor-defined LUT or function.
   - If header is unknown or payload is uninterpretable:
     - Fallback to passthrough (retain raw bits).
