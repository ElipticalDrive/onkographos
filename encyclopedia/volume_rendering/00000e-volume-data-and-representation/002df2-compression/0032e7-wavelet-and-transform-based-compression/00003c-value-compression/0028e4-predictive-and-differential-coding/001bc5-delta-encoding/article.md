# Delta Encoding

Also called: DPCM, Differential pulse-code modulation, Heightfield Compression, Temporal prediction

## Overview
Delta encoding (or Differential Pulse-Code Modulation, DPCM) predicts a current sample value based on previously processed neighbors or reference frames, computes the residual (difference) between the actual and predicted values, and encodes this residual [S1][S2, unofficial][S3]. It solves the problem of high redundancy in spatially or temporally coherent data by transforming the signal into a residual with a smaller dynamic range and higher probability of zero values, which is more efficiently compressed by entropy coding [S1][S2, unofficial][S3]. Unlike sibling methods that may use complex linear combinations or transform-domain residuals, standard delta encoding typically relies on simple causal predictors (e.g., previous value or heightfield base), making it computationally trivial but potentially limiting for random access [S1].

## How it works
**Inputs:** A sequence of voxel values $V_t$ (where $t$ denotes time step or spatial scan order) and a prediction model.
**Outputs:** A stream of delta values (residuals) $R_t$, ready for entropy coding.

**Data Structures:**
*   **Input Buffer:** Array of original values $V$.
*   **Prediction Buffer:** Array of previously decoded/predicted values $V_{pred}$ or $V_{prev}$.
*   **Output Buffer:** Array of residuals $R$.

**Numbered Steps:**
1.  **Prediction:** Estimate the current value $V_{pred}$ using the prediction model $f$. For temporal data, $V_{pred} = V_{t-1}$ [S1]. For spatial heightfields, $V_{pred}$ may be the minimum value of the current block or a linear combination of neighbors [S2, unofficial][S3].
2.  **Residual Calculation:** Compute the difference $R_t = V_t - V_{pred}$ [S1][S2, unofficial][S3].
3.  **Storage:** Store $R_t$ in the output stream [S1].
4.  **Decompression (GPU/CPU):** To reconstruct, add the stored delta $R_t$ to the existing/predicted value $V_{pred}$ rather than overwriting it [S1].

**Formulas:**
*   Prediction: $V_{pred} = f(V_{neighbors})$
*   Residual: $R = V - V_{pred}$
*   Reconstruction: $V_{rec} = V_{pred} + R$

**Parameters:**
*   **Prediction Model:** Simple temporal ($V_{t-1}$) or spatial (block minimum, parallelogram) [S1][S3].
*   **Bit Depth:** Original values are often uniformly quantized (e.g., 16 bits) before encoding [S3]. Residuals may use reduced bit-depths if the range is small [S1].

**Edge Cases and Pitfalls:**
*   **Random Access Limitation:** Standard delta encoding prevents jumping to a specific time step or spatial location because all previous values must be decoded to reconstruct the current one [S1][S2, unofficial]. This limits use in real-time visualization requiring efficient queries [S2, unofficial].
*   **Mitigation:** Save absolute values in addition to delta values and load them on demand, increasing storage cost [S1].
*   **Precision:** Ensure that the residual calculation does not overflow the target bit depth. If residuals are quantized, closed-loop decoding (using reconstructed values for prediction) prevents error accumulation [S1][S2, unofficial][S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delta_encode(values, prediction_model='temporal'):
    """
    Encodes a sequence of values using delta encoding.
    
    Args:
        values: List of original voxel values.
        prediction_model: 'temporal' for time-varying data, 
                          'block_min' for heightfields (simplified).
                          
    Returns:
        List of delta values.
    """
    deltas = []
    prev_value = None
    
    for i, v in enumerate(values):
        if prediction_model == 'temporal':
            if i == 0:
                # First value must be stored as absolute or handled specially
                # For simplicity, assume first value is stored as-is or 
                # a reference frame exists. Here we store delta from 0.
                pred = 0 
            else:
                pred = prev_value
        elif prediction_model == 'block_min':
            # Simplified block-based: assume 'values' is a single block
            # In practice, block_min is computed over the block first.
            # This step assumes 'pred' is the block minimum.
            # Note: This requires two passes or pre-computation.
            pred = min(values) # Placeholder for block minimum logic
        
        delta = v - pred
        deltas.append(delta)
        prev_value = v # Use original value for open-loop prediction
        
    return deltas

def delta_decode(deltas, prediction_model='temporal'):
    """
    Decodes delta values back to original values.
    
    Args:
        deltas: List of delta values.
        prediction_model: 'temporal' or 'block_min'.
        
    Returns:
        List of reconstructed values.
    """
    reconstructed = []
    prev_value = None
    
    for i, d in enumerate(deltas):
        if prediction_model == 'temporal':
            if i == 0:
                # Reconstruct first value
                # If first delta was from 0, then v = d
                # If first value was stored absolute, this logic differs.
                # Assuming standard DPCM where first value is absolute or delta from 0:
                pred = 0
            else:
                pred = prev_value
        elif prediction_model == 'block_min':
            # For block-based, the base value is known from the block header
            # This simplified version assumes the base is added to all deltas
            # or that the first delta is absolute.
            # Standard block delta: v = base + delta
            # Here we assume 'deltas' are relative to a known base 'B'.
            # Since 'B' is not passed, we assume the first delta is absolute 
            # or the base is implicitly 0 for this generic function.
            pred = 0 # Placeholder for block base value
            
        v = pred + d
        reconstructed.append(v)
        prev_value = v
        
    return reconstructed
```

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method compresses heightfields by partitioning them into equal-sized blocks, where each block is encoded relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two reads: one for the block header and one for the block's data.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression is a value compression method that stores quantized differences relative to a block base value, expanding them to 16-bit integers during decompression to reconstruct final data. It distinguishes itself from sibling methods by utilizing pre-computed shift patterns to handle variable bit-depths efficiently during SIMD-based decompression, avoiding runtime bit-manipulation overhead.

### Parallelogram Prediction with Diamond Residuals
Lossless compression of height fields using linear prediction. Height values are uniformly quantized to 16 bits. Parallelogram prediction is used for the z-component. Vertices are traversed in diagonal rows. The first three vertices of the first triangle are stored uncompressed. Subsequent vertices are predicted using a diamond configuration to maximize square diamond configurations, ensuring predictor vertices are close in the domain [S3].

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are $O(N)$ where $N$ is the number of voxels, as each value is processed once with constant-time arithmetic [S1].
*   **Memory Complexity:** $O(N)$ for input and output buffers. No significant auxiliary memory is required for the basic algorithm [S1].
*   **Measured Performance:** On GPUs, delta encoding is trivial to implement and can achieve full coalescing of memory accesses if threads are organized to process adjacent memory cells (e.g., by x-coordinate in a brick) [S1].
*   **Strengths:** Extremely low computational overhead; effective for data with high temporal or spatial coherence (e.g., time-varying volumes, heightfields) [S1][S2, unofficial].
*   **Limitations:** Poor random access performance without additional structures (e.g., keyframes or block headers); sensitive to prediction accuracy [S1][S2, unofficial].
*   **When to Use:** Time-varying volume data with high temporal coherence; heightfields with progressive variation; scenarios where computational cost of complex predictors is prohibitive [S1][S2, unofficial].
*   **When Not to Use:** Data with high randomness or low coherence; applications requiring efficient random access to arbitrary time steps or spatial locations without block-based extensions [S1][S2, unofficial].

## Background
Delta encoding, also known as Differential Pulse-Code Modulation (DPCM), is a foundational technique in signal and data compression. It was originally developed for audio and image compression to exploit redundancy. In volume rendering, it has been adapted for time-varying volume data and heightfields. Key applications include the compression of time-varying scientific data [S1] and layered datasets like QuadStack [S2, unofficial]. The technique is often combined with entropy coding (e.g., Huffman, Run-Length) to achieve significant compression ratios [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
