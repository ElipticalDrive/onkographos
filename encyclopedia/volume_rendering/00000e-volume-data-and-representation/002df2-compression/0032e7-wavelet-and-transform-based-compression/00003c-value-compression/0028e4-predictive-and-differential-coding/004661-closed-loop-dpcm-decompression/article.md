# Closed-Loop DPCM Decompression

Also called: TraceBlockCodec.decompress

## Overview
Closed-loop DPCM decompression reconstructs seismic trace data from compressed payloads by reversing a predictive coding scheme that maintains state consistency between encoding and decoding steps [S1, unofficial]. It solves the problem of accurately recovering quantized sample values from a bitstream while ensuring that the decoder's internal prediction state matches the encoder's, thereby preventing error accumulation [S1, unofficial]. It is distinguished from open-loop or legacy DPCM methods by its explicit handling of quantization parameters and bit-depth metadata within the payload, allowing for precise reconstruction without relying on fixed assumptions about the encoding process [S1, unofficial].

## How it works
**Inputs:** A `CompressedTraceBlock` object containing a payload (byte array), metadata including `samplesPerTrace`, `min`, and `max` values, and a trace identifier.
**Outputs:** A `TraceBlock` object containing the reconstructed trace ID and an array of normalized floating-point samples.

**Data Structures:**
*   `CompressedTraceBlock`:
    *   `payload`: `byte[]` containing the compressed data.
    *   `samplesPerTrace`: `int` $n$, the number of samples in the trace.
    *   `min`: `float`, the minimum value of the original trace.
    *   `max`: `float`, the maximum value of the original trace.
    *   `traceId`: `int`, identifier for the trace.
*   `TraceBlock`:
    *   `traceId`: `int`.
    *   `samples`: `float[]` of length $n$.

**Steps:**
1.  **Check Mode:** Determine if the payload uses the closed-loop format. This is done by calling `ClosedLoopQuantizer.isClosedLoop(cb.payload())` [S1, unofficial].
2.  **Fallback:** If the payload is not closed-loop, invoke `decompressLegacy(cb)` and return the result immediately [S1, unofficial].
3.  **Extract Parameters:**
    *   Let $n = cb.samplesPerTrace()$.
    *   Let $bits = ClosedLoopQuantizer.storedBits(cb.payload())$. This retrieves the number of bits used for quantization [S1, unofficial].
    *   Let $inv = 1.0f / ClosedLoopQuantizer.maxQ(bits)$. This is the inverse of the maximum quantization value for the given bit depth [S1, unofficial].
4.  **Unpack Quantized Values:** Unpack the quantized integer values $q$ from the payload using `ClosedLoopQuantizer.unpack(cb.payload(), n)`. This produces an array $q$ of length $n$ containing the encoded deltas or residuals [S1, unofficial].
5.  **Reconstruct Samples:**
    *   Initialize `prev` = $0.0f$. This variable holds the previously reconstructed sample value [S1, unofficial].
    *   For each index $i$ from $0$ to $n-1$:
        *   Compute the reconstructed normalized value: `prev` = `ClosedLoopQuantizer.reconstruct(prev, q[i], inv)`.
        *   The `reconstruct` function typically adds the quantized delta $q[i]$ (scaled by $inv$) to the previous sample `prev` [S1, unofficial].
        *   Store the result: $norm[i] = \text{prev}$.
6.  **Denormalize:** Convert the normalized samples $norm$ from the range $[-1, 1]$ back to the original scale using the trace's min and max values:
    $$ \text{original}[i] = \frac{norm[i] + 1}{2} \times (cb.max() - cb.min()) + cb.min() $$
    This is performed by `Preprocessing.denormalizeFromMinusOneToOne(norm, cb.min(), cb.max())` [S1, unofficial].
7.  **Return:** Create and return a new `TraceBlock` with the original `traceId` and the denormalized samples.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_closed_loop(cb):
    # Step 1: Check if payload is closed-loop
    if not ClosedLoopQuantizer.isClosedLoop(cb.payload):
        return decompress_legacy(cb)

    # Step 2: Extract parameters
    n = cb.samplesPerTrace
    bits = ClosedLoopQuantizer.storedBits(cb.payload)
    inv = 1.0 / ClosedLoopQuantizer.maxQ(bits)
    
    # Step 3: Unpack quantized values
    q = ClosedLoopQuantizer.unpack(cb.payload, n)
    
    # Step 4: Reconstruct samples
    prev = 0.0
    norm = [0.0] * n
    
    for i in range(n):
        # Closed-loop reconstruction: add scaled delta to previous sample
        prev = ClosedLoopQuantizer.reconstruct(prev, q[i], inv)
        norm[i] = prev
        
    # Step 5: Denormalize
    samples = Preprocessing.denormalizeFromMinusOneToOne(norm, cb.min, cb.max)
    
    # Step 6: Return TraceBlock
    return TraceBlock(traceId=cb.traceId, samples=samples)
```

**Parameters:**
*   $bits$: Number of bits used for quantization. Typical values depend on the compression profile (e.g., 8, 12, 16 bits). Not given in the sources.
*   $inv$: Inverse quantization step size. Calculated as $1 / 2^{bits}$ or similar, depending on `maxQ` implementation.

**Edge Cases and Pitfalls:**
*   **Legacy Payloads:** If the payload was created by a legacy encoder, `isClosedLoop` returns false, and the method falls back to `decompressLegacy`. Attempting to decode legacy data with closed-loop logic will fail or produce garbage [S1, unofficial].
*   **Bit Depth Mismatch:** The legacy decoder does not store bit depth, leading to potential decoding errors if the file was written with fewer than 16 bits [S1, unofficial]. The closed-loop decoder explicitly stores and retrieves bit depth, avoiding this issue [S1, unofficial].
*   **Precision:** The use of `float` (32-bit) for reconstruction may introduce minor precision errors compared to the original double-precision data, but this is standard for seismic data storage [S1, unofficial].

## Subtopics

### Legacy Decompression
Decompresses payloads written before the closed-loop revision, supporting raw zlib streams, open-loop delta encoding, and 16-bit quantization [S1, unofficial]. It is kept for backward compatibility with `.sdc` v2 files [S1, unofficial]. The process involves inflating the zlib stream, converting bytes to shorts, decoding deltas using `LinearQuantizer.decode`, delta-decoding the result, and denormalizing [S1, unofficial]. A key pitfall is that files written with fewer than 16 bits by the legacy version cannot be decoded correctly because the bit depth was never stored in the file format [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $n$, $O(n)$, due to the single pass unpacking and reconstruction loop [S1, unofficial].
*   **Memory Complexity:** Linear, $O(n)$, to store the unpacked quantized values and the reconstructed normalized samples [S1, unofficial].
*   **Performance:** Fast decompression due to simple arithmetic operations and no complex inverse transforms [S1, unofficial].
*   **Strengths:** Accurate reconstruction due to closed-loop state management; handles variable bit depths explicitly [S1, unofficial].
*   **Limitations:** Requires the payload to be in the closed-loop format; legacy formats require a separate code path [S1, unofficial].
*   **When to Use:** When decompressing seismic data stored in the modern `.sdc` format with closed-loop encoding [S1, unofficial].
*   **When Not to Use:** When dealing with legacy `.sdc` v2 files, which require the legacy decompressor [S1, unofficial].

## Background
This method is part of the SeismicDataCompressor project, an unofficial implementation for compressing and decompressing seismic data [S1, unofficial]. It was developed to provide efficient storage and transmission of seismic trace blocks, addressing the need for lossless or near-lossless compression in seismic data processing workflows [S1, unofficial]. The closed-loop approach was introduced to fix issues with error accumulation in open-loop predictive coding and to support variable bit-depth quantization [S1, unofficial].

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
