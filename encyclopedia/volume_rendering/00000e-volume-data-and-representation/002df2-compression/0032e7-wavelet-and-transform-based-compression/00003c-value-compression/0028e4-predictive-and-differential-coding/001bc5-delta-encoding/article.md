# Delta Encoding

Also called: Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding predicts the current value of a data element based on a previous value (typically the preceding element in a traversal order or the same element in a previous time step) and stores only the difference (residual) between the prediction and the actual value [S1]. It solves the problem of compressing data with high local or temporal coherence, where values change slowly or remain static [S1]. Unlike its siblings in the transform domain, delta encoding operates directly on the raw values in spatial or temporal order without a linear transform, making it particularly effective for time-varying volumes or heightfields with progressive variation [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of values $V = \{v_0, v_1, \dots, v_{N-1}\}$ representing voxels in a volume or heightfield, traversed in a specific order (e.g., time steps, or spatial scanline).
*   **Output:** A sequence of delta values $D = \{d_0, d_1, \dots, d_{N-1}\}$, where $d_i$ is the difference between $v_i$ and its prediction $\hat{v}_i$.

**Data Structures:**
The output is typically a stream of integers. To enable efficient entropy coding, the delta values are often stored in a format that highlights uniform regions (e.g., bricks of zeros). No complex hierarchical data structures are required for the core delta encoding itself, unlike wavelet-based methods.

**Steps:**
1.  **Prediction:** For each element $v_i$ in the input sequence, predict its value $\hat{v}_i$ based on previously processed data.
    *   For **temporal data** (time-varying volumes), the prediction is usually the value from the previous time step: $\hat{v}_i = v_{i-1}$ [S1].
    *   For **spatial data** (heightfields), the prediction is often the previous element in the traversal order or a linear combination of neighbors (e.g., parallelogram prediction) [S3].
2.  **Delta Calculation:** Compute the residual (delta) $d_i$:
    $$ d_i = v_i - \hat{v}_i $$
    If the prediction is perfect, $d_i = 0$ [S1].
3.  **Storage:** Store $d_i$. If the data type is fixed-width (e.g., 16-bit), $d_i$ is stored in that format. For heightfields, values may be uniformly quantized to 16 bits before encoding [S3].
4.  **Entropy Coding:** Apply variable-length coding (e.g., Huffman, Run-Length) to the delta stream. Regions with no change produce sequences of zeros, which compress efficiently [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delta_encode(values):
    """
    Encodes a sequence of values using delta encoding.
    values: List of numeric values (e.g., int16, float32).
    Returns: List of delta values.
    """
    deltas = []
    # For the first element, the prediction is typically 0 or the value itself is stored
    # as an absolute reference. Here we assume the first value is stored as-is or 
    # predicted as 0. [G] Standard practice often stores v[0] absolutely.
    if len(values) == 0:
        return []
    
    # Store first value absolutely to allow random access start [G]
    deltas.append(values[0])
    
    for i in range(1, len(values)):
        # Prediction: previous value (temporal or spatial neighbor)
        prediction = values[i-1]
        
        # Calculate delta
        delta = values[i] - prediction
        
        deltas.append(delta)
        
    return deltas

def delta_decode(deltas):
    """
    Decodes a sequence of delta values back to original values.
    deltas: List of delta values.
    Returns: List of reconstructed values.
    """
    if len(deltas) == 0:
        return []
        
    values = [deltas[0]]
    
    for i in range(1, len(deltas)):
        # Reconstruct by adding delta to the previously reconstructed value
        reconstructed = values[-1] + deltas[i]
        values.append(reconstructed)
        
    return values
```

**Parameters:**
*   **Prediction Model:** The choice of predictor (e.g., previous value, linear extrapolation) depends on the data coherence [S1, S3].
*   **Bit Depth:** The bit depth of the delta values. If the original data is 16-bit, deltas may also be 16-bit, but often a smaller range allows for reduced bit-depth storage after quantization [S3].

**Edge Cases and Pitfalls:**
*   **Random Access:** Delta encoding prevents jumping directly to a specific time step or spatial location because all preceding values must be decoded to reconstruct the current one [S1][S2, unofficial]. This limits practical use in applications requiring efficient queries or real-time visualization [S2, unofficial].
*   **Error Propagation:** In lossy scenarios or if a single delta is corrupted, the error propagates to all subsequent values in the sequence [G].
*   **Storage Overhead:** Storing absolute values alongside deltas for random access increases storage requirements [S1].

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method partitions a heightfield into equal-sized blocks and encodes each block independently relative to its minimum height value using reduced bit-depth for differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
A lossless compression method for height fields using linear prediction where height values are uniformly quantized to 16 bits [S3]. It uses parallelogram prediction for the z-component, traversing vertices in diagonal rows to maximize square diamond configurations, ensuring predictor vertices are close in the domain [S3]. The first three vertices of the first triangle are stored uncompressed [S3].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are linear $O(N)$ with respect to the number of elements, as each value depends only on the previous one [S1].
*   **Memory Complexity:** $O(N)$ for storing the delta stream. No additional hierarchical structures are needed for the core method [S1].
*   **Performance:** Highly effective for data with high temporal or spatial coherence (e.g., time-varying volumes with slow changes, heightfields with progressive variation) [S1][S2, unofficial].
*   **Strengths:** Simple implementation; excellent compression ratios for smooth or static data; efficient GPU decompression (addition operation) [S1].
*   **Limitations:** Poor compression for random or high-frequency data; lack of random access without auxiliary structures; error propagation [S1][S2, unofficial].
*   **When to Use:** Time-varying volume rendering where frames are similar; heightfields with smooth terrain; scenarios where sequential access is sufficient [S1][S2, unofficial].
*   **When Not to Use:** Data with high randomness; applications requiring random access to arbitrary time steps or spatial locations without overhead [S1][S2, unofficial].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), has been used in various forms for decades in signal processing and data compression [S1]. In the context of volume rendering, it was notably applied to compress time-varying volume data by exploiting temporal coherence [S1]. It is also used in heightfield compression for terrain data, where height values vary progressively [S2, unofficial]. The method is often combined with entropy coding to achieve lossless compression [S1, S3].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
