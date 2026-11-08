# Delta Encoding

Also called: Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding is a predictive compression method that exploits temporal or spatial coherence by storing the difference (residual) between a predicted value and the actual value, rather than the absolute value itself [S1]. It solves the problem of redundant data in time-varying volumes or smoothly varying heightfields by converting uniform regions into zero-valued residuals, which are highly compressible via entropy coding [S1]. Unlike sibling methods that may use complex linear predictors or transform-domain residuals, standard delta encoding relies on the simplest prediction model: the previous value [S1]. Its primary limitation is the lack of random access, as reconstructing any specific value requires decompressing all preceding data in the processing order [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of voxel values $v_t$ (where $t$ denotes time step or spatial position in a fixed scan order).
*   **Output:** A sequence of delta values $d_t$, followed by entropy-coded residuals.

**Data Structures:**
*   **Delta Buffer:** An array of integers with the same dimensions as the input volume. If the input is $N$ bits per voxel, the delta values typically require $N$ bits (or fewer if quantized/clamped, though standard DPCM preserves bit-depth before entropy coding) [S1].
*   **Block Header (for block-based variants):** Stores the base value (e.g., minimum height) for the block [S1].

**Numbered Steps:**
1.  **Prediction:** Estimate the current voxel value $\hat{v}_t$ using the previous voxel value in the processing order. For temporal data, $\hat{v}_t = v_{t-1}$ [S1]. For spatial heightfields, $\hat{v}_t$ is the previously processed neighbor (e.g., in a diagonal traversal) [S3].
2.  **Residual Calculation:** Compute the difference $d_t = v_t - \hat{v}_t$ [S1].
3.  **Storage/Encoding:** Store or entropy-code $d_t$ [S1].
4.  **Decompression (GPU/CPU):** To reconstruct $v_t$, add the decoded delta $d_t$ to the previously reconstructed value $\hat{v}_t$ (which is effectively $v_{t-1}$ in a closed-loop system) [S1].

**Formulas:**
*   Prediction: $\hat{v}_t = v_{t-1}$ [S1]
*   Residual: $d_t = v_t - \hat{v}_t$ [S1]
*   Reconstruction: $v_t = \hat{v}_t + d_t$ [S1]

**Parameters:**
*   **Bit Depth:** Typically matches the input data type (e.g., 16-bit for heightfields) [S3].
*   **Block Size:** For block-based implementations, equal-sized blocks (e.g., $m \times n$) are used [S1].

**Edge Cases and Pitfalls:**
*   **Random Access:** Standard delta encoding prevents jumping to a specific time step or spatial location because all preceding data must be decompressed [S1][S2, unofficial]. This can be mitigated by storing absolute values for keyframes or block headers, increasing storage cost [S1, S1].
*   **Precision:** In lossy scenarios, using reconstructed (dequantized) values for prediction is critical to prevent encoder-decoder drift [Parent].
*   **Uniform Regions:** Unchanged regions result in delta values of zero, creating uniform bricks that compress efficiently with variable-length coding [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delta_encode(volume):
    """
    Encodes a 1D sequence of voxel values using delta encoding.
    Assumes volume is a list of integers.
    Returns a list of delta values.
    """
    if not volume:
        return []
    
    deltas = []
    # First value has no predecessor; store as absolute or handle as special case
    # Standard DPCM often stores the first value absolutely or assumes v[-1] = 0
    # Here we assume v[-1] = 0 for simplicity, or store v[0] as d[0]
    prev_v = 0 
    
    for v in volume:
        d = v - prev_v
        deltas.append(d)
        prev_v = v  # Update for next step (closed-loop for lossy, open for lossless if exact)
        
    return deltas

def delta_decode(deltas):
    """
    Decodes a sequence of delta values back to original voxel values.
    """
    if not deltas:
        return []
        
    reconstructed = []
    prev_v = 0
    
    for d in deltas:
        v = prev_v + d
        reconstructed.append(v)
        prev_v = v
        
    return reconstructed
```

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method compresses heightfields by partitioning them into equal-sized blocks, where each block is encoded relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression is a value compression method that stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
This variant uses linear prediction for lossless compression of height fields, where height values are uniformly quantized to 16 bits [S3]. It employs parallelogram prediction for the z-component (height) with vertices traversed in diagonal rows [S3]. The first three vertices of the first triangle are stored uncompressed, and subsequent vertices are predicted using a diamond configuration to maximize square diamond configurations, ensuring predictor vertices are close in the domain [S3].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are $O(N)$, where $N$ is the number of voxels, as each voxel requires one subtraction and one addition [S1].
*   **Memory Complexity:** $O(N)$ for storing the delta values [S1].
*   **Measured Performance:** On GPUs, delta encoding is trivial to implement and allows for coalesced memory accesses if threads are organized to process adjacent memory cells (e.g., distributing brick assembly onto blocks of 64 CUDA threads) [S1].
*   **Strengths:** Simple implementation; highly effective for data with high temporal coherence or smooth spatial gradients (e.g., heightfields) [S1][S2, unofficial].
*   **Limitations:** No random access without auxiliary data (keyframes/block headers); performance degrades if the prediction error distribution is wide or non-uniform [S1][S2, unofficial].
*   **When to Use:** Time-varying volumes with high temporal coherence; heightfields with progressive variation [S1][S2, unofficial].
*   **When Not to Use:** Applications requiring efficient random access or traversal without auxiliary structures; data with high-frequency noise or random changes [S1][S2, unofficial].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), was introduced for time-varying volume data to exploit temporal coherence [S1]. It was proposed as a simple method providing good results because height values usually vary progressively [S2, unofficial]. The method was further adapted for heightfield compression and terrain rendering, where it was combined with block-based strategies to address random access limitations [S1, S3].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
