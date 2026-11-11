# Delta Encoding
Also called: Differential pulse-code modulation, Heightfield Compression

## Overview
Delta encoding is a predictive compression method for time-varying volume data that exploits temporal coherence by assuming voxel values remain constant between time steps, storing only the difference (residual) between the actual value and the predicted previous value [S1]. It distinguishes itself from sibling methods like [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md) by operating directly in the spatial-temporal domain without prior transform steps, relying on the fact that unchanged regions produce zero-valued residuals that compress efficiently into uniform bricks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of volume datasets $V_t$ at time steps $t=0, 1, \dots, T$. Each voxel has an original value $V_{actual}$.
*   **Output:** A compressed stream of delta values (residuals) $R_t$. For $t=0$, the raw values are typically stored as absolute values.

**Data Structures:**
*   **Bricks:** The volume is partitioned into 3D blocks (bricks).
*   **Delta Brick:** Stores the residual values $R$. If a brick contains only zeros (unchanged region), it is encoded as a uniform structure to maximize compression efficiency [S1].

**Steps:**
1.  **Prediction:** For each voxel at time step $t > 0$, predict the current value $V_{pred}$ to be equal to the value at the previous time step $t-1$.
    $$V_{pred}(x,y,z,t) = V_{reconstructed}(x,y,z,t-1)$$
    Note: In a closed-loop decoder, $V_{reconstructed}$ is used to prevent error accumulation [S1].
2.  **Residual Calculation:** Compute the delta (residual) $R$ as the difference between the actual value and the prediction.
    $$R(x,y,z,t) = V_{actual}(x,y,z,t) - V_{pred}(x,y,z,t)$$
3.  **Storage:** Store the residual values $R$. If the residual is zero, it contributes to a uniform brick structure [S1].
4.  **Entropy Coding:** Apply variable-length coding (e.g., Huffman) to the residuals. Since residuals are often small integers with a Laplacian-like distribution (many zeros, few large values), they compress significantly better than raw data [S1].

**Pseudocode (Decompression on GPU):**
The decoder reconstructs the volume by adding the delta to the existing value in the volume buffer.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_delta_encoding(delta_bricks, previous_volume_buffer):
    """
    Reconstructs the current volume from delta bricks and the previous volume state.
    
    Args:
        delta_bricks: List of compressed delta bricks for the current time step.
        previous_volume_buffer: The reconstructed volume data from time step t-1.
        
    Returns:
        current_volume_buffer: The reconstructed volume data for time step t.
    """
    current_volume_buffer = allocate_buffer_like(previous_volume_buffer)
    
    for brick in delta_bricks:
        # Decompress the delta brick to get raw residuals
        residuals = decompress_variable_length_coding(brick)
        
        # Map residuals to voxel coordinates within the brick
        for x, y, z in brick.voxel_indices:
            # Closed-loop reconstruction: add delta to the previously reconstructed value
            # This assumes previous_volume_buffer holds the state from t-1
            prev_val = previous_volume_buffer[x, y, z]
            delta_val = residuals.get((x, y, z), 0) # 0 if not explicitly stored (uniform brick)
            
            current_volume_buffer[x, y, z] = prev_val + delta_val
            
    return current_volume_buffer
```

**Parameters:**
*   **Brick Size:** Not given in the sources.
*   **Time Step Interval:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Random Access Limitation:** Delta encoding prevents jumping directly to a specific time step $t$ because all previous time steps $0 \dots t-1$ must be read and decompressed to reconstruct the state [S1]. This can be mitigated by storing absolute values for keyframes and loading them on demand, at the cost of increased storage [S1].
*   **Precision:** In lossless contexts, the residual is exact. In lossy contexts, quantization may be applied to the residual [S1].

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method compresses heightfields by partitioning them into equal-sized blocks, where each block is encoded relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression is a value compression method that stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
This variant uses linear prediction for lossless compression of height fields, where height values are uniformly quantized to 16 bits [S3]. It employs parallelogram prediction for the z-component (height) while xy coordinates are known on a regular grid [S3]. Vertices are traversed in diagonal rows to maximize square diamond configurations, ensuring predictor vertices are close in the domain for better accuracy [S3]. The first three vertices of the first triangle are stored uncompressed to bootstrap the process [S3].

## Operating characteristics
*   **Time Complexity:** Decompression is linear with respect to the number of voxels, as it involves a simple addition per voxel [S1].
*   **Memory Complexity:** Requires storage for the delta values plus the previous volume state for closed-loop decoding [S1].
*   **Performance:** Achieves high compression ratios for time-varying data with high temporal coherence (e.g., fluid simulations with slow changes) [S1].
*   **Strengths:** Simple to implement; highly effective when changes between time steps are sparse or uniform [S1].
*   **Limitations:** Poor random access to time steps; requires sequential decoding from the start or keyframes [S1]. Not suitable for data with high temporal noise or random changes [S1].
*   **When to Use:** Time-varying volume rendering where temporal coherence is high and random access to intermediate time steps is not required [S1].
*   **When Not to Use:** Applications requiring efficient queries or traversals across time steps without sequential decoding [S1].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), was proposed for volume compression by Aalberg et al. in 2010 [S1]. It builds on the general principle of predictive coding, exploiting the fact that voxel values in time-varying data often do not change significantly between consecutive time steps [S1]. The method is particularly effective when combined with brick-based storage and variable-length coding, as unchanged regions form uniform bricks that compress efficiently [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
