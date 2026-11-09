# Delta Encoding

Also called: Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding (also known as Differential Pulse-Code Modulation or DPCM) is a predictive compression method that exploits coherence in data by storing the difference between a predicted value and the actual value [S1]. In volume rendering, it is primarily used to compress time-varying data by predicting that a voxel's value remains unchanged between time steps, or heightfields where values vary progressively [S1][S2, unofficial]. It sets itself apart from sibling methods like [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md) by using a constant predictor (identity or previous time step) rather than a complex linear combination of spatial neighbors, resulting in lower computational overhead but potentially higher residual entropy if the data changes rapidly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of volume data $V_t$ (where $t$ is the time step) or a heightfield $H$.
*   **Output:** A stream of residual values (deltas) $D$, which are subsequently entropy-encoded.

**Data Structures:**
The residual $d$ for a voxel at position $p$ in time step $t$ is stored as an integer. If the original data is 16-bit, the delta is typically stored in 16-bit signed integers to accommodate negative differences [S1].

**Steps:**
1.  **Prediction:** For each voxel $v_{t,p}$ in the current time step $t$, predict the value $\hat{v}_{t,p}$ based on the previous time step $t-1$.
    $$ \hat{v}_{t,p} = v_{t-1,p} $$
    For heightfields, the prediction is often the previous value in a scan order or a spatial neighbor [S2, unofficial][S3].
2.  **Residual Calculation:** Compute the difference between the actual value and the prediction.
    $$ d_{t,p} = v_{t,p} - \hat{v}_{t,p} $$
    If the voxel has not changed, $d_{t,p} = 0$ [S1].
3.  **Storage:** Store the residual $d_{t,p}$. Regions with no change produce uniform bricks of zeros, which are highly compressible via variable-length coding (e.g., Huffman) [S1].
4.  **Decompression (GPU/CPU):** To reconstruct $v_{t,p}$, the decoder adds the decoded residual to the previously reconstructed value.
    $$ v_{t,p} = \hat{v}_{t,p} + d_{t,p} $$
    On the GPU, this is implemented by adding the calculated delta to the existing value in the volume buffer rather than overwriting it, ensuring memory coalescing if threads are organized by x-coordinate [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_delta_encoding(volume_prev, volume_curr):
    """
    Encodes the difference between two consecutive time steps.
    volume_prev: 3D array of previous time step values.
    volume_curr: 3D array of current time step values.
    Returns: 3D array of delta residuals.
    """
    residuals = empty_array_like(volume_curr)
    for p in all_voxels(volume_curr):
        predicted = volume_prev[p]
        actual = volume_curr[p]
        residuals[p] = actual - predicted
    return residuals

def decode_delta_encoding(volume_prev, residuals):
    """
    Reconstructs the current time step from the previous step and residuals.
    volume_prev: 3D array of previously reconstructed values.
    residuals: 3D array of decoded delta residuals.
    Returns: 3D array of reconstructed current values.
    """
    volume_curr = empty_array_like(residuals)
    for p in all_voxels(residuals):
        volume_curr[p] = volume_prev[p] + residuals[p]
    return volume_curr
```

**Parameters:**
*   **Bit Depth:** The residual storage bit depth should match the dynamic range of the expected differences. For 16-bit height data, 16-bit signed integers are standard [S3].
*   **Block Size:** For heightfields, partitioning into blocks (e.g., $m \times n$) allows independent compression of blocks [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Random Access Limitation:** Delta encoding creates a temporal dependency chain. To access time step $t$, all previous steps $0 \dots t-1$ must be decoded [S1]. This prevents jumping directly to a specific time step.
*   **Mitigation:** Absolute values can be stored alongside deltas for keyframes, allowing random access to those specific steps at the cost of increased storage [S1].
*   **Precision:** If the original data is quantized, the residuals are also quantized. Ensure the residual type can represent the maximum possible difference without overflow [S1].

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This variant partitions a heightfield into equal-sized blocks and encodes each block relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
A lossless compression method for height fields using linear prediction where height values are uniformly quantized to 16 bits [S3]. It uses parallelogram prediction for the z-component, traversing vertices in diagonal rows to maximize square diamond configurations, ensuring predictor vertices are close in the domain [S3]. The first three vertices of the first triangle are stored uncompressed [S3].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are $O(N)$, where $N$ is the number of voxels, as each voxel requires one subtraction and one addition [S1].
*   **Memory Complexity:** $O(N)$ for storing the residuals. If absolute values are stored for keyframes, memory usage increases proportionally to the keyframe frequency [S1].
*   **Performance:** Highly efficient for data with high temporal coherence (e.g., slowly evolving simulations) or smooth heightfields [S1][S2, unofficial]. Performance on GPU is optimized by coalesced memory accesses when processing voxels along x-coordinates [S1].
*   **Strengths:** Simple implementation; excellent compression ratios for static or slowly changing regions; low computational overhead [S1].
*   **Limitations:** Poor compression for random or rapidly changing data; lack of random access in time-varying data without keyframes [S1].
*   **When to Use:** Time-varying volumes with high frame-to-frame similarity; heightfields with progressive variation [S1][S2, unofficial].
*   **When Not to Use:** Data with high entropy or random noise; applications requiring frequent random access to intermediate time steps without keyframe overhead [S1].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), was introduced for volume data compression to exploit temporal coherence [S1]. It was notably applied in GPU-supported lossless compression schemes for rendering time-varying volume data [S1]. The method has also been adapted for heightfield compression in layered datasets like QuadStack, where it serves as a simple yet effective baseline for interval height compression [S2, unofficial]. The parallelogram prediction variant was developed for on-the-fly decompression and rendering of multiresolution terrain [S3].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
