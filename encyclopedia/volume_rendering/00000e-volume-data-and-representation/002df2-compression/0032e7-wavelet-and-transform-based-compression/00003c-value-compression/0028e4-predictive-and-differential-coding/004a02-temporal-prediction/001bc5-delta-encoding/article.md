# Delta Encoding

Also called: Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding is a predictive compression method that exploits temporal coherence in time-varying volume data by storing the difference (error) between the current voxel value and a prediction, typically the value from the previous time step [S1]. It sets itself apart from sibling methods like [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md) by operating directly in the spatial domain without requiring a transform step, making it computationally simpler but strictly sequential for reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of volume datasets $V_t$ and $V_{t-1}$ representing voxel values at time steps $t$ and $t-1$.
*   **Output:** A compressed dataset containing delta values $\Delta V_t$, where $\Delta V_t = V_t - V_{t-1}$ (prediction).

**Data Structures:**
The output is a volume of the same dimensions as the input, storing the quantized difference values. If the original data is $n$-bit, the delta values may require $n$ bits or fewer if the range of differences is small. The data is typically organized into bricks or blocks for efficient processing [S1].

**Steps:**
1.  **Prediction:** For each voxel at position $(x, y, z)$, predict the current value $V_t(x,y,z)$ using the value from the previous time step: $\hat{V}_t(x,y,z) = V_{t-1}(x,y,z)$ [S1].
2.  **Delta Calculation:** Compute the difference between the actual current value and the predicted value:
    $$ \Delta V_t(x,y,z) = V_t(x,y,z) - \hat{V}_t(x,y,z) $$
    [S1]
3.  **Storage:** Store $\Delta V_t(x,y,z)$. Regions where the voxel value has not changed result in a delta value of zero [S1].
4.  **Compression:** The resulting delta values often exhibit uniform structures (e.g., many zeros) and a non-uniform distribution with a smaller dynamic range than the original data, allowing for efficient variable-length coding or other entropy coding techniques [S1].

**GPU Decompression:**
To reconstruct the volume on the GPU, the kernel adds the stored delta value to the existing value in the volume buffer (which must hold the previous time step's data) rather than overwriting it [S1]. This requires memory coalescing, typically achieved by distributing brick assembly across blocks of threads where each thread processes voxels along a specific coordinate (e.g., x-coordinate) to ensure adjacent threads access adjacent memory cells [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_delta_encoding(volume_current, volume_previous):
    """
    Encodes a volume using delta encoding relative to a previous time step.
    
    Args:
        volume_current: 3D array of voxel values at time t.
        volume_previous: 3D array of voxel values at time t-1.
        
    Returns:
        delta_volume: 3D array of difference values.
    """
    delta_volume = empty_like(volume_current)
    for x, y, z in volume_current.indices():
        prediction = volume_previous[x, y, z]
        actual = volume_current[x, y, z]
        delta_volume[x, y, z] = actual - prediction
    return delta_volume

def decode_delta_encoding(delta_volume, volume_previous):
    """
    Decodes a delta-encoded volume by adding deltas to the previous volume.
    
    Args:
        delta_volume: 3D array of difference values.
        volume_previous: 3D array of voxel values at time t-1.
        
    Returns:
        volume_current: 3D array of reconstructed voxel values at time t.
    """
    volume_current = empty_like(volume_previous)
    for x, y, z in delta_volume.indices():
        prediction = volume_previous[x, y, z]
        delta = delta_volume[x, y, z]
        volume_current[x, y, z] = prediction + delta
    return volume_current
```

**Parameters:**
*   **Prediction Model:** The default is temporal prediction ($V_{t-1}$) [S1].
*   **Bit Depth:** Not given in the sources. Typically matches the input precision or is reduced if the delta range is small [S1].

**Edge Cases and Pitfalls:**
*   **Random Access Limitation:** Delta encoding prevents jumping directly to a specific time step because all previous time steps must be read sequentially to reconstruct the data [S1]. This limits practical use in applications requiring efficient queries or traversals, such as real-time visualization [S2, unofficial].
*   **Mitigation:** The limitation can be resolved by saving absolute values in addition to delta values and loading them on demand, at the cost of increased storage requirements [S1].

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method compresses heightfields by partitioning them into equal-sized blocks, where each block is encoded relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression is a value compression method that stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
This variant is a lossless compression technique for height fields using linear prediction [S3]. Height values are uniformly quantized to 16 bits [S3]. It uses parallelogram prediction for the z-component (height), with vertices traversed in diagonal rows [S3]. The first three vertices of the first triangle are stored uncompressed [S3]. Subsequent vertices are predicted using a diamond configuration to maximize square diamond configurations, ensuring predictor vertices are as close as possible in the domain [S3].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are linear with respect to the number of voxels, $O(N)$, where $N$ is the total number of voxels [S1].
*   **Memory Complexity:** Requires storage for the delta values plus the previous volume state for decoding [S1].
*   **Performance:** Provides good compression results for data with high temporal coherence, as unchanged regions result in zero deltas which compress efficiently [S1][S2, unofficial].
*   **Strengths:** Simple to implement; effective for time-varying data with smooth changes; low computational overhead for prediction and reconstruction [S1][S2, unofficial].
*   **Limitations:** Sequential dependency prevents random access to time steps; performance degrades if temporal coherence is low (high-frequency changes) [S1][S2, unofficial].
*   **When to use:** Suitable for time-varying volume data where temporal coherence is high and sequential access is acceptable [S1].
*   **When not to use:** Unsuitable for applications requiring random access to specific time steps or rapid traversal of the dataset [S1][S2, unofficial].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), was introduced for volume data compression to exploit temporal coherence in time-varying datasets [S1]. It was proposed as a simple method providing good results because height values and volume data usually vary progressively [S2, unofficial]. The method is cited in works such as [AABY06] and [FBS07] as referenced in [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
