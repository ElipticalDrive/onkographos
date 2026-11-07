# Delta Encoding

Also called: DPCM, Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding (or Differential Pulse-Code Modulation, DPCM) predicts the current voxel value based on previously processed data—typically the value from the previous time step in temporal data or a spatial neighbor—and stores only the difference (residual) between the prediction and the actual value [S1]. It sets itself apart from sibling methods by focusing on the explicit exploitation of temporal coherence or local spatial continuity to reduce the dynamic range of the data, often resulting in uniform regions of zero residuals that compress efficiently with variable-length coding [S1]. Unlike transform-based methods that operate on the entire dataset simultaneously, delta encoding is inherently sequential and causal [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of volume data frames $V_t$ (for temporal) or a spatial volume $V$ (for spatial). Each voxel has an original value $v$.
*   **Output:** A sequence of delta values $d$ (residuals).

**Data Structures:**
The output is typically stored as a stream of integers. If the original data is $N$-bit, the deltas may be stored in the same bit-depth or a reduced bit-depth if the residuals are known to be small. For block-based variants, a block header stores the base value (e.g., minimum height) and the block dimensions [S1].

**Steps:**
1.  **Prediction:** Estimate the current voxel value $\hat{v}_t$ using the value from the previous time step: $\hat{v}_t = v_{t-1}$ [S1]. For spatial data, $\hat{v}$ might be the previously decoded neighbor in raster order.
2.  **Residual Calculation:** Compute the difference $d_t = v_t - \hat{v}_t$ [S1].
3.  **Encoding:** Encode $d_t$ using entropy coding (e.g., Huffman, Arithmetic). If $v_t = v_{t-1}$, then $d_t = 0$, creating uniform bricks of zeros that compress highly efficiently [S1].
4.  **Reconstruction:** The decoder reconstructs the value by adding the decoded delta to the previously reconstructed value: $v_t = \hat{v}_t + d_t$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_delta(original_values, previous_reconstructed_values):
    """
    Encodes a volume frame using delta encoding relative to the previous frame.
    Assumes previous_reconstructed_values is available from the prior step.
    """
    deltas = []
    for v_current, v_prev in zip(original_values, previous_reconstructed_values):
        # Step 1 & 2: Predict and calculate residual
        # Prediction: v_hat = v_prev
        # Residual: d = v_current - v_hat
        d = v_current - v_prev
        deltas.append(d)
    return deltas

def decode_delta(deltas, previous_reconstructed_values):
    """
    Decodes a volume frame using delta encoding.
    Must use the *quantized/reconstructed* previous values to prevent drift.
    """
    current_reconstructed = []
    for d, v_prev in zip(deltas, previous_reconstructed_values):
        # Step 4: Reconstruct
        v_current = v_prev + d
        current_reconstructed.append(v_current)
    return current_reconstructed
```

**Parameters:**
*   **Prediction Model:** For time-varying data, the "no change" model ($\hat{v}_t = v_{t-1}$) is standard [S1].
*   **Bit Depth:** Not given in the sources. Typically matches the input or is reduced based on the range of $d$.

**Edge Cases and Pitfalls:**
*   **Drift:** In lossy scenarios (if residuals are quantized), the encoder must update its prediction state using the *quantized* (reconstructed) value, not the original, to prevent encoder-decoder drift [S1].
*   **Random Access:** Standard delta encoding prevents jumping to a specific time step because all previous steps must be decoded to reconstruct the current one [S1]. This can be mitigated by storing absolute values (keyframes) in addition to deltas, at the cost of increased storage [S1].
*   **Access Latency:** Decompressing the whole dataset is required for any data access, limiting use in applications requiring efficient queries or real-time visualization [S2, unofficial].

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
Compresses heightfields by partitioning them into blocks, encoding values relative to a block base (e.g., minimum height) to enable random access with two reads.

### [Differences-based Compression](5723-differences-based-compression.md)
Stores quantized differences relative to a block base, expanding to 16-bit integers during decompression, using pre-computed shift patterns for SIMD efficiency.

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are $O(N)$ where $N$ is the number of voxels, as each voxel is processed independently given the previous state.
*   **Memory Complexity:** $O(N)$ for storing the deltas. Random access requires $O(N)$ memory to hold the full history of reconstructed frames unless keyframes are used [S1].
*   **Performance:** Highly effective for data with high temporal coherence (e.g., slowly evolving simulations) or smooth spatial gradients, where residuals cluster around zero [S1].
*   **Strengths:** Simple implementation; excellent compression ratios for coherent data; efficient GPU decompression via coalesced memory accesses (adding deltas to existing textures) [S1].
*   **Limitations:** Poor compression for random or noisy data; sequential dependency prevents random access to intermediate time steps without storing absolute values [S1]; full decompression required for any access [S2, unofficial].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), is a classic technique in signal processing adapted for volume rendering [S1]. It was applied to time-varying volume data to exploit temporal coherence, where voxel values are predicted to remain unchanged between time steps [S1]. The method is cited in works such as [AABY06, FBS07] referenced in [S1]. It is also used in heightfield compression for layered datasets, where progressive height variations make delta encoding effective [S2, unofficial].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
