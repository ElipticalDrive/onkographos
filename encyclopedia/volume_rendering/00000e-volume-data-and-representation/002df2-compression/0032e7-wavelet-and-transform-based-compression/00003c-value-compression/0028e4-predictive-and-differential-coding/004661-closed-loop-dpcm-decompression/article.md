# Closed-Loop DPCM Decompression

Also called: TraceBlockCodec.decompress

## Overview
Closed-Loop DPCM Decompression reconstructs volumetric trace data from compressed payloads by applying inverse differential pulse-code modulation using reconstructed (dequantized) values for prediction, thereby preventing error accumulation in lossy scenarios [S1, unofficial]. It distinguishes itself from open-loop variants by ensuring decoder consistency through the use of previously decoded samples rather than original values for the prediction model [S1, unofficial].

## How it works
**Inputs:** A `CompressedTraceBlock` object containing a payload, metadata (trace ID, sample count, min/max values), and a flag indicating closed-loop encoding.
**Outputs:** A `TraceBlock` object containing the reconstructed trace ID and normalized sample values.

**Data Structures:**
*   `CompressedTraceBlock`:
    *   `payload`: Binary data containing quantized residuals.
    *   `traceId`: Identifier for the trace.
    *   `samplesPerTrace`: Integer $n$, the number of samples in the trace.
    *   `min`: Float, minimum original value for denormalization.
    *   `max`: Float, maximum original value for denormalization.
*   `TraceBlock`:
    *   `traceId`: Identifier.
    *   `samples`: Array of floats, the reconstructed values.

**Steps:**
1.  **Check Mode:** Determine if the payload uses closed-loop encoding. If not, delegate to legacy decompression [S1, unofficial].
2.  **Extract Parameters:** Retrieve $n$ (samples per trace) and `bits` (stored bit depth) from the payload metadata [S1, unofficial].
3.  **Unpack Quantized Values:** Decode the binary payload into an array of quantized integers $q[0 \dots n-1]$ [S1, unofficial].
4.  **Compute Inverse Scale:** Calculate the inverse quantization scale factor $inv = 1.0 / Q_{max}$, where $Q_{max}$ is the maximum quantization value for the given bit depth (e.g., $2^{bits} - 1$) [S1, unofficial].
5.  **Initialize State:** Set the previous reconstructed value $prev = 0.0$ [S1, unofficial].
6.  **Reconstruct Samples:** For each sample index $i$ from $0$ to $n-1$:
    *   Retrieve the quantized residual $q[i]$.
    *   Compute the reconstructed normalized value: $norm[i] = prev + q[i] \times inv$ [S1, unofficial].
    *   Update the closed-loop state: $prev = norm[i]$ [S1, unofficial].
7.  **Denormalize:** Map the normalized values $norm[i] \in [-1, 1]$ back to the original data range $[min, max]$ using linear scaling [S1, unofficial].
8.  **Return:** Construct and return the `TraceBlock` with the denormalized samples.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_closed_loop(cb):
    # Step 1: Check if payload is closed-loop
    if not is_closed_loop(cb.payload):
        return decompress_legacy(cb)

    # Step 2: Extract parameters
    n = cb.samples_per_trace
    bits = get_stored_bits(cb.payload)
    
    # Step 3: Unpack quantized values
    q = unpack_payload(cb.payload, n)
    
    # Step 4: Compute inverse scale
    max_q = (1 << bits) - 1
    inv = 1.0 / max_q
    
    # Step 5: Initialize state
    prev = 0.0
    norm = [0.0] * n
    
    # Step 6: Reconstruct samples
    for i in range(n):
        # Reconstruct using closed-loop prediction
        # norm[i] = prev + q[i] * inv
        # prev becomes the current reconstructed value for the next step
        prev = prev + q[i] * inv
        norm[i] = prev
        
    # Step 7: Denormalize
    min_val = cb.min
    max_val = cb.max
    samples = [denormalize_from_minus_one_to_one(norm[i], min_val, max_val) for i in range(n)]
    
    # Step 8: Return
    return TraceBlock(trace_id=cb.trace_id, samples=samples)
```

**Edge Cases and Pitfalls:**
*   **Legacy Fallback:** If the payload is not marked as closed-loop, the method must fall back to legacy decompression to handle older file formats [S1, unofficial].
*   **Bit Depth:** The bit depth must be correctly extracted from the payload; incorrect bit depth leads to wrong $Q_{max}$ and scaling errors [S1, unofficial].

## Subtopics

### Legacy Decompression
Decompresses payloads written before the closed-loop revision, supporting raw zlib streams, open-loop delta encoding, and 16-bit quantization for backward compatibility with `.sdc` v2 files [S1, unofficial]. It inflates the payload, converts bytes to shorts, decodes deltas using a linear quantizer, performs delta decoding, and denormalizes the result [S1, unofficial]. A critical pitfall is that files written with fewer than 16 bits by the legacy version cannot be decoded correctly because the bit depth was never stored in the file format [S1, unofficial].

## Operating characteristics
**Time Complexity:** Linear $O(n)$ with respect to the number of samples per trace, as each sample requires one unpacking, one addition, and one multiplication [S1, unofficial].
**Memory Complexity:** $O(n)$ for storing the quantized residuals and the reconstructed normalized values [S1, unofficial].
**Strengths:** Prevents error accumulation in lossy compression scenarios by using reconstructed values for prediction, ensuring decoder consistency [S1, unofficial].
**Limitations:** Requires explicit bit depth metadata in the payload; legacy files without this metadata are unsupported [S1, unofficial].
**When to Use:** When decoding compressed seismic or volumetric data that was encoded using the closed-loop DPCM method [S1, unofficial].
**When Not to Use:** When decoding legacy `.sdc` v2 files or payloads not marked as closed-loop [S1, unofficial].

## Background
Closed-Loop DPCM Decompression is part of the SeismicDataCompressor project, implemented in the `TraceBlockCodec` class [S1, unofficial]. It was introduced to improve the accuracy of lossy compression by addressing error accumulation issues present in open-loop predictors [S1, unofficial]. The method is used in the decompression pipeline for seismic trace data stored in `.sdc` format [S1, unofficial].

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
