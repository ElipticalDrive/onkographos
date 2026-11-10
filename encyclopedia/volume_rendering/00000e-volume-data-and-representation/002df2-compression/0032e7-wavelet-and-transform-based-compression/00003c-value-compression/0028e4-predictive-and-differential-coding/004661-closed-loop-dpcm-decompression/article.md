# Closed-Loop DPCM Decompression

Also called: TraceBlockCodec.decompress

## Overview
Closed-loop DPCM decompression reconstructs volumetric data from compressed trace blocks using a feedback mechanism where the decoder's prediction is based on previously reconstructed values rather than original source data [S1, unofficial]. This method ensures that quantization errors do not accumulate across samples within a trace, distinguishing it from open-loop variants [S1, unofficial]. It serves as the primary decoder for modern `.sdc` v3+ files, replacing legacy open-loop and raw compression schemes [S1, unofficial].

## How it works
The decompression process operates on a `CompressedTraceBlock` object. It first determines the compression format to select the appropriate decoding path. If the payload is not identified as closed-loop, it delegates to the legacy decoder [S1, unofficial]. For closed-loop payloads, the algorithm extracts quantization parameters, unpacks the bitstream, and iteratively reconstructs sample values using a feedback loop [S1, unofficial].

**Inputs:**
*   `cb`: A `CompressedTraceBlock` object containing the compressed payload, metadata (trace ID, min/max values), and sample count.

**Outputs:**
*   A `TraceBlock` object containing the decompressed trace ID and an array of normalized floating-point samples.

**Data Structures:**
*   `CompressedTraceBlock`:
    *   `payload`: Byte array or buffer containing the compressed data.
    *   `samplesPerTrace()`: Integer method returning the number of samples in the trace.
    *   `min()`, `max()`: Float values representing the original data range for denormalization.
    *   `traceId`: Identifier for the trace.
*   `ClosedLoopQuantizer`: Helper object providing quantization logic.
    *   `isClosedLoop(payload)`: Boolean check.
    *   `storedBits(payload)`: Integer bit depth used during compression.
    *   `unpack(payload, n)`: Returns an array of `n` quantized integer values.
    *   `reconstruct(prev, q, inv)`: Computes the current reconstructed value based on the previous reconstructed value and the current quantized residual.
*   `Preprocessing`: Helper object for normalization.
    *   `denormalizeFromMinusOneToOne(norm, min, max)`: Scales normalized values $[-1, 1]$ to $[min, max]$.

**Steps:**
1.  **Format Check:** Verify if the payload is closed-loop using `ClosedLoopQuantizer.isClosedLoop(cb.payload())`. If false, invoke `decompressLegacy(cb)` and return the result [S1, unofficial].
2.  **Parameter Extraction:**
    *   Retrieve the number of samples: $n = cb.samplesPerTrace()$.
    *   Retrieve the bit depth: $bits = ClosedLoopQuantizer.storedBits(cb.payload())$.
    *   Compute the inverse quantization scale factor: $inv = \frac{1.0}{ClosedLoopQuantizer.maxQ(bits)}$.
3.  **Unpacking:** Extract the quantized integer values $q[0 \dots n-1]$ from the payload using `ClosedLoopQuantizer.unpack(cb.payload(), n)` [S1, unofficial].
4.  **Reconstruction Loop:**
    *   Initialize the previous reconstructed value: $prev = 0.0$.
    *   For each sample index $i$ from $0$ to $n-1$:
        *   Compute the current reconstructed normalized value: $prev = ClosedLoopQuantizer.reconstruct(prev, q[i], inv)$.
        *   Store the result: $norm[i] = prev$.
    *   *Note:* The `reconstruct` function internally implements the closed-loop prediction: $\hat{v}_i = prev_{i-1} + q[i] \times inv$ (assuming additive residual coding with unity gain in the feedback path, as implied by the standard DPCM structure and the use of `inv` as a scaler) [S1, unofficial]. The exact internal formula of `ClosedLoopQuantizer.reconstruct` is not exposed, but it relies on the previous *reconstructed* value $prev$ to maintain synchronization between encoder and decoder.
5.  **Denormalization:** Convert the normalized array $norm$ back to the original data scale using `Preprocessing.denormalizeFromMinusOneToOne(norm, cb.min(), cb.max())` [S1, unofficial].
6.  **Return:** Construct and return a new `TraceBlock` with the original `traceId` and the denormalized samples.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_closed_loop(cb):
    # Step 1: Check format
    if not ClosedLoopQuantizer.isClosedLoop(cb.payload):
        return decompress_legacy(cb)

    # Step 2: Extract parameters
    n = cb.samplesPerTrace()
    bits = ClosedLoopQuantizer.storedBits(cb.payload)
    inv = 1.0 / ClosedLoopQuantizer.maxQ(bits)
    
    # Step 3: Unpack quantized values
    q = ClosedLoopQuantizer.unpack(cb.payload, n)
    
    # Step 4: Reconstruct samples
    prev = 0.0
    norm = [0.0] * n
    
    for i in range(n):
        # Closed-loop reconstruction: uses previous reconstructed value
        prev = ClosedLoopQuantizer.reconstruct(prev, q[i], inv)
        norm[i] = prev
        
    # Step 5: Denormalize
    samples = Preprocessing.denormalizeFromMinusOneToOne(norm, cb.min(), cb.max())
    
    # Step 6: Return result
    return TraceBlock(traceId=cb.traceId, samples=samples)
```

**Edge Cases and Pitfalls:**
*   **Legacy Compatibility:** The decoder must handle payloads from older versions (`.sdc` v2) which lack the closed-loop header. Failing to detect this leads to incorrect decoding or crashes [S1, unofficial].
*   **Bit Depth Ambiguity:** Legacy files written with fewer than 16 bits cannot be decoded correctly because the bit depth was not stored in the legacy format [S1, unofficial]. Closed-loop files store the bit depth explicitly, avoiding this issue [S1, unofficial].
*   **Initialization:** The initial `prev` value is set to 0.0. This assumes the first sample's prediction is based on a zero baseline or that the first residual is absolute. If the encoder uses a different initial condition, synchronization will fail [S1, unofficial].

## Subtopics

### Legacy Decompression
Decompresses payloads written before the closed-loop revision, supporting raw zlib streams, open-loop delta encoding, and 16-bit quantization [S1, unofficial]. It is retained for backward compatibility with `.sdc` v2 files [S1, unofficial]. The process involves inflating the zlib stream, converting bytes to shorts, decoding deltas using `LinearQuantizer`, delta-decoding the normalized values, and denormalizing [S1, unofficial]. A critical pitfall is that legacy files compressed with fewer than 16 bits cannot be decoded correctly because the bit depth was never stored in the legacy format [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $n$ in the trace, $O(n)$, due to the single pass unpacking and reconstruction loop [S1, unofficial].
*   **Memory Complexity:** $O(n)$ for storing the unpacked quantized values and the normalized output array [S1, unofficial].
*   **Strengths:** Eliminates error accumulation across samples within a trace by using a closed-loop feedback mechanism, ensuring exact reconstruction (within quantization limits) regardless of trace length [S1, unofficial]. Explicitly stores bit depth, preventing the ambiguity present in legacy formats [S1, unofficial].
*   **Limitations:** Requires the decoder to maintain state (the previous reconstructed value) for each sample, preventing simple parallelization of sample reconstruction within a single trace without lookahead [S1, unofficial]. Dependent on the correctness of the `ClosedLoopQuantizer` implementation [S1, unofficial].
*   **When to use:** When decompressing modern `.sdc` v3+ files where data integrity and precision are critical [S1, unofficial].
*   **When not to use:** When processing legacy `.sdc` v2 files, which require the legacy decoder path [S1, unofficial].

## Background
Closed-loop DPCM decompression is part of the SeismicDataCompressor project, an open-source tool for compressing seismic data [S1, unofficial]. It was introduced as a revision to the original open-loop DPCM approach to address error accumulation issues in long traces [S1, unofficial]. The method is implemented in the `TraceBlockCodec` class, specifically via the `decompress` method which dispatches to either the closed-loop or legacy decoder based on payload metadata [S1, unofficial]. The project is hosted on GitHub and is considered unofficial as it is not peer-reviewed [S1, unofficial].

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
