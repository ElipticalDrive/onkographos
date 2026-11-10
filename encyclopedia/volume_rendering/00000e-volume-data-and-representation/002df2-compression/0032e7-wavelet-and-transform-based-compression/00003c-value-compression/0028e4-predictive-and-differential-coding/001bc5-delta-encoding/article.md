# Delta Encoding
Also called: Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding is a predictive value compression method that exploits temporal or spatial coherence by storing the difference (residual) between an actual sample value and a predicted value, rather than the absolute value itself [S1]. It is primarily used for time-varying volume data where voxel values exhibit high temporal coherence, or for heightfields where values vary progressively [S1][S2, unofficial]. Unlike sibling methods that may use complex linear predictors or transform-domain residuals, standard delta encoding typically relies on the simplest prediction model: the previous sample's value [S1]. This approach transforms the data distribution into one peaked around zero, enabling efficient entropy coding, but at the cost of sequential access requirements for reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of volume data frames (time-varying) or a heightfield grid. Let $V_t(x,y,z)$ be the voxel value at position $(x,y,z)$ in time step $t$.
*   **Output:** A compressed stream of delta values (residuals) $D_t(x,y,z)$ and potentially base values or headers for block-based variants.

**Data Structures:**
*   **Delta Buffer:** Stores the residual values $D_t$. The bit-depth of this buffer can be reduced based on the dynamic range of the residuals, which is typically much smaller than the original data range [S1].
*   **Block Header (for block variants):** Stores the base value (e.g., minimum height) for a block of data [S1][S2, unofficial].

**Prediction Model:**
The simplest and most common prediction model for time-varying data assumes that the current voxel value will not change from the previous time step [S1].
$$ \hat{V}_t(x,y,z) = V_{t-1}(x,y,z) $$
For heightfields, a similar progressive assumption is made, often using the previous value in a traversal order or a block base value [S2, unofficial].

**Residual Generation:**
The residual (delta) is computed as the difference between the actual value and the prediction:
$$ D_t(x,y,z) = V_t(x,y,z) - \hat{V}_t(x,y,z) $$
Substituting the temporal prediction model:
$$ D_t(x,y,z) = V_t(x,y,z) - V_{t-1}(x,y,z) $$
If the voxel value remains unchanged between time steps, $D_t(x,y,z) = 0$ [S1].

**Entropy Coding:**
The delta values are entropy-coded (e.g., Huffman, Arithmetic, or variable-length coding) [S1]. Because unchanged regions produce uniform bricks of zeros, and varying regions produce residuals with a non-uniform distribution concentrated around zero, the data is highly compressible [S1]. The bit-depth of the stored deltas can be optimized per brick or block based on the local dynamic range [S1].

**Reconstruction:**
The decoder reconstructs the value by adding the decoded residual to the previously reconstructed value. For time-varying data, this is an accumulation process:
$$ V_{rec,t}(x,y,z) = V_{rec,t-1}(x,y,z) + D_{rec,t}(x,y,z) $$
This implies that to reconstruct frame $t$, all previous frames $0$ to $t-1$ must be available or reconstructed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_delta_encoding(volume_frame_current, volume_frame_previous):
    """
    Encodes a current volume frame using delta encoding relative to the previous frame.
    
    Args:
        volume_frame_current: 3D array of voxel values at time t.
        volume_frame_previous: 3D array of voxel values at time t-1.
        
    Returns:
        deltas: 3D array of residual values.
    """
    # Initialize deltas array with the same dimensions as the volume
    deltas = empty_array_like(volume_frame_current)
    
    # Iterate over all voxels
    for x, y, z in all_voxels(volume_frame_current):
        # Prediction: assume value does not change
        predicted_value = volume_frame_previous[x, y, z]
        
        # Calculate residual
        actual_value = volume_frame_current[x, y, z]
        deltas[x, y, z] = actual_value - predicted_value
        
    return deltas

def decode_delta_encoding(deltas, volume_frame_previous):
    """
    Decodes a delta-encoded volume frame by adding residuals to the previous frame.
    
    Args:
        deltas: 3D array of residual values.
        volume_frame_previous: 3D array of reconstructed values at time t-1.
        
    Returns:
        volume_frame_current: 3D array of reconstructed voxel values at time t.
    """
    # Initialize current frame
    volume_frame_current = empty_array_like(deltas)
    
    # Iterate over all voxels
    for x, y, z in all_voxels(deltas):
        # Reconstruction: add residual to previous value
        volume_frame_current[x, y, z] = volume_frame_previous[x, y, z] + deltas[x, y, z]
        
    return volume_frame_current
```

**Parameters:**
*   **Prediction Type:** Temporal (previous frame) or Spatial (previous sample in traversal) [S1][S2, unofficial].
*   **Bit-depth:** Determined by the maximum absolute residual value in a brick/block. If the range is $< 2^n$, $n$ bits can be used [S1].

**Edge Cases and Pitfalls:**
*   **Error Propagation:** Since reconstruction is cumulative, any error in decoding a delta value will propagate to all subsequent time steps or spatial positions in the traversal order [S1].
*   **Random Access:** Standard delta encoding prevents jumping directly to a specific time step because all previous steps must be read and reconstructed [S1]. This can be mitigated by storing absolute values (keyframes) in addition to deltas, but this increases storage [S1].
*   **Boundary Conditions:** For the first time step ($t=0$), no previous frame exists. The first frame must be stored as absolute values or a separate base frame [G].

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method partitions a heightfield into equal-sized blocks and encodes each block relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
This variant uses linear prediction for height fields, specifically parallelogram prediction for the z-component, with vertices traversed in diagonal rows [S3]. The first three vertices of the first triangle are stored uncompressed. Subsequent vertices are predicted using a diamond configuration to maximize square diamond configurations, ensuring predictor vertices are as close as possible in the domain [S3].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are $O(N)$ where $N$ is the number of voxels, as each voxel is processed once. However, decoding requires sequential access, which may limit parallelization benefits on GPUs unless block-based variants are used [S1].
*   **Memory Complexity:** $O(N)$ for storing the deltas. If keyframes are stored for random access, memory usage increases proportionally to the frequency of keyframes [S1].
*   **Measured Performance:** On GPUs, delta encoding can be implemented trivially by adding deltas to existing values in the volume, allowing for coalesced memory accesses if threads are organized appropriately (e.g., one thread per x-coordinate processing all z-values) [S1].
*   **Strengths:** Simple to implement; highly effective for data with high temporal coherence or progressive spatial variation; produces uniform bricks of zeros for unchanged regions, which compress well with entropy coding [S1][S2, unofficial].
*   **Limitations:** Prevents random access to time steps without keyframes; error propagation in sequential decoding; not suitable for data with high-frequency noise or random changes between time steps [S1].
*   **When to Use:** Time-varying volume data with smooth changes; heightfields with progressive variations; scenarios where sequential access is acceptable or block-based variants are employed [S1][S2, unofficial].
*   **When Not to Use:** Data requiring random access to arbitrary time steps without keyframes; data with high temporal incoherence; scenarios where error propagation is unacceptable [S1].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), has been used in various forms for audio and image compression for decades. In the context of volume rendering, it was adapted for time-varying volume data to exploit temporal coherence [S1]. The method was highlighted in works such as "A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data" (2010) for its efficiency in GPU-based decompression [S1]. It has also been applied to heightfield compression in terrain rendering, where its simplicity and effectiveness for progressive data make it a common choice [S2, unofficial][S3].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
