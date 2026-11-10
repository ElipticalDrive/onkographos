# Closed-Loop DPCM Decompression

Also called: TraceBlockCodec.decompress

## Overview
Closed-loop DPCM decompression reconstructs seismic trace data from compressed payloads by applying inverse quantization and delta decoding within a closed-loop feedback structure. It solves the problem of accurately recovering sample values from quantized differential codes while maintaining backward compatibility with legacy formats. It is distinguished from open-loop delta encoding by its use of a feedback loop where the reconstructed value, rather than the original error, is used as the predictor for the next sample, ensuring that quantization errors do not accumulate unboundedly [S1, unofficial].

## How it works
The method processes a `CompressedTraceBlock` by first determining the compression format. If the payload indicates a closed-loop structure, it performs inverse quantization and delta decoding using a feedback loop. Otherwise, it falls back to legacy decompression.

**Inputs:** A `CompressedTraceBlock` object containing:
- `payload`: The compressed bitstream.
- `samplesPerTrace` ($n$): The number of samples in the trace.
- `min`: The minimum original sample value.
- `max`: The maximum original sample value.

**Outputs:** A `TraceBlock` containing the reconstructed trace samples and metadata.

**Data Structures:**
- `q`: Array of $n$ quantized integer values.
- `norm`: Array of $n$ normalized floating-point values.
- `inv`: Scaling factor $1.0 / Q_{max}$.

**Steps:**
1.  **Format Check:** Determine if the payload is closed-loop.
    -   Call `ClosedLoopQuantizer.isClosedLoop(cb.payload())`.
    -   If false, invoke `decompressLegacy(cb)` and return the result [S1, unofficial].
2.  **Initialization:**
    -   Get $n = cb.samplesPerTrace()$.
    -   Get `bits` = `ClosedLoopQuantizer.storedBits(cb.payload())`.
    -   Compute `inv` = $1.0 / \text{ClosedLoopQuantizer.maxQ}(\text{bits})$.
    -   Initialize `prev` = $0.0$ [S1, unofficial].
3.  **Unpacking:**
    -   Unpack quantized values $q[i]$ from `cb.payload()` for $i = 0 \dots n-1$ using `ClosedLoopQuantizer.unpack`.
4.  **Reconstruction Loop:**
    -   For each $i$ from $0$ to $n-1$:
        -   Reconstruct the normalized value: `prev` = `ClosedLoopQuantizer.reconstruct(prev, q[i], inv)`.
        -   Set `norm[i]` = `prev`.
    -   *Note:* The `reconstruct` function typically performs $v_{rec} = v_{prev} + q[i] \times \text{inv}$, updating `prev` to $v_{rec}$ for the next iteration [S1, unofficial].
5.  **Denormalization:**
    -   Map `norm` from $[-1, 1]$ to original scale:
        $$ v_{final}[i] = \frac{\text{norm}[i] + 1.0}{2.0} \times (\text{max} - \text{min}) + \text{min} $$
    -   This is implemented via `Preprocessing.denormalizeFromMinusOneToOne(norm, cb.min(), cb.max())` [S1, unofficial].
6.  **Return:** Create and return a new `TraceBlock` with the reconstructed samples and original `traceId` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_closed_loop(cb):
    # Step 1: Check format
    if not ClosedLoopQuantizer.isClosedLoop(cb.payload):
        return decompress_legacy(cb)

    # Step 2: Initialization
    n = cb.samplesPerTrace
    bits = ClosedLoopQuantizer.storedBits(cb.payload)
    inv = 1.0 / ClosedLoopQuantizer.maxQ(bits)
    prev = 0.0
    
    # Step 3: Unpacking
    q = ClosedLoopQuantizer.unpack(cb.payload, n)
    
    # Step 4: Reconstruction Loop
    norm = []
    for i in range(n):
        # Closed-loop DPCM: prev is the previously reconstructed value
        prev = ClosedLoopQuantizer.reconstruct(prev, q[i], inv)
        norm.append(prev)
        
    # Step 5: Denormalization
    # Map from [-1, 1] to [min, max]
    min_val = cb.min
    max_val = cb.max
    samples = []
    for val in norm:
        # Linear interpolation from [-1, 1] to [min, max]
        sample = ((val + 1.0) / 2.0) * (max_val - min_val) + min_val
        samples.append(sample)
        
    # Step 6: Return
    return TraceBlock(traceId=cb.traceId, samples=samples)
```

**Parameters:**
-   `bits`: Number of bits used for quantization. Typical values depend on the compression profile (e.g., 8, 12, 16 bits) [S1, unofficial].
-   `inv`: Inverse of the maximum quantization level, derived from `bits` [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Legacy Fallback:** If the payload is not marked as closed-loop, the method must fall back to legacy decoding to support older `.sdc` v2 files [S1, unofficial].
-   **Precision:** The use of `float` (32-bit) for `prev` and `norm` is specified in the source implementation (`1.0f`) [S1, unofficial].

## Subtopics

### Legacy Decompression
Decompresses payloads written before the closed-loop revision, supporting raw zlib streams, open-loop delta, and 16-bit quantization. It is kept for backward compatibility with `.sdc` v2 files [S1, unofficial].
**Steps:**
1.  Inflate `cb.payload()` using `inflate`.
2.  Convert raw bytes to shorts using `bytesToShorts`.
3.  Decode deltas using `LinearQuantizer.decode`.
4.  Delta-decode using `Preprocessing.deltaDecode`.
5.  Denormalize using `Preprocessing.denormalizeFromMinusOneToOne`.
6.  Return `TraceBlock` [S1, unofficial].
**Pitfall:** Files written with fewer than 16 bits by the legacy version cannot be decoded correctly because the bit depth was never stored [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of samples $n$, $O(n)$, due to the single pass unpacking and reconstruction loop [S1, unofficial].
-   **Memory Complexity:** $O(n)$ for storing the output samples and intermediate normalized values [S1, unofficial].
-   **Strengths:** Closed-loop DPCM prevents error accumulation compared to open-loop methods, providing better reconstruction quality for quantized data [S1, unofficial].
-   **Limitations:** Requires checking the payload format and potentially invoking a different code path (legacy) if the format is not recognized [S1, unofficial].
-   **When to use:** When decompressing seismic data compressed with the closed-loop DPCM encoder.
-   **When not to use:** When the data was compressed using legacy formats (use `decompressLegacy` instead) or other compression schemes [S1, unofficial].

## Background
The method is part of the SeismicDataCompressor project, which handles compression and decompression of seismic data. The closed-loop DPCM approach was introduced to improve upon legacy open-loop delta encoding, which suffered from error accumulation and lack of bit-depth metadata [S1, unofficial].

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
