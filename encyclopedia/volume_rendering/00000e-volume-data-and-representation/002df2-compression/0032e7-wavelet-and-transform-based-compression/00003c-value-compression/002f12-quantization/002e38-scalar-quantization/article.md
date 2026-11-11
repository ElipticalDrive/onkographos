# Scalar Quantization

Also called: Per-block scalar quantization, Scalar Quantization Impact

## Overview
Scalar quantization (per-block) is a random-access decompression method used in compressed volume rendering to reconstruct voxel values on-the-fly without storing fully decompressed data in memory [S2][S3]. It sets itself apart from general scalar quantization by operating on localized blocks, computing local minimum and maximum bounds to define a linear interpolation range, which allows for efficient GPU implementation and direct rendering from compressed data [S1][S2]. Unlike vector quantization, it does not rely on codebooks or weighted combinations of multiple basis vectors, making it simpler to implement but potentially less accurate for complex data distributions [S2][S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A block of compressed voxel data, where each voxel is represented by a local index. The block is associated with two scalar values: the minimum value ($min$) and the maximum value ($max$) of the original data within that block.
*   **Output:** The decompressed scalar value $v_{ijk}$ for a specific voxel at coordinates $(i, j, k)$ within the block.

**Data Structures:**
*   **Block Metadata:** Two floating-point or fixed-point values representing the global minimum ($min$) and maximum ($max$) of the block.
*   **Voxel Data:** A sequence of integers representing the local index for each voxel in the block. The number of bits used to store each index is denoted by $ibits$.

**Steps:**
1.  **Block Partitioning:** Divide the volume data into fixed-size blocks. The block size is not specified in the sources [G].
2.  **Compute Bounds:** For each block, determine the minimum value ($min$) and the maximum value ($max$) of the original unquantized data contained within that block.
3.  **Quantize (Encoding):** For each voxel in the block, compute a local index based on its original value relative to $min$ and $max$. The index is an integer in the range $[0, 2^{ibits} - 1]$. The sources do not specify the exact rounding or clamping rules for this step [G].
4.  **Store:** Store the $min$ and $max$ values once per block, and store the local indices for all voxels in the block using $ibits$ per index.
5.  **Reconstruct (Decompression):** For a target voxel, retrieve its local index and the block's $min$ and $max$ values. Compute the decompressed value using the linear interpolation formula.

**Formula:**
The decompressed value $v_{ijk}$ is calculated as:
$$ v_{ijk} = min + (max - min) \times \frac{index}{2^{ibits-1}} $$
Where:
*   $v_{ijk}$ is the decompressed value of the voxel at $(i, j, k)$.
*   $min$ is the minimum value of the block.
*   $max$ is the maximum value of the block.
*   $index$ is the local integer index stored for the voxel.
*   $ibits$ is the number of bits used to represent the index.

**Parameters:**
*   **$ibits$ (bits per index):** The precision of the quantization. Typical values are not explicitly given, but the sources discuss bitrates ranging from 4 to 9 bits in the context of scalar quantization impact [S1]. Lower bitrates lead to higher reconstruction error.
*   **Block Size:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision Loss:** The reconstruction error roughly doubles for each reduction by one bit in the scalar quantizer [S1]. This is because each bit reduction halves the precision of the quantized interpolation weight [S1].
*   **Block Range:** If the data within a block has a very large range relative to the desired precision, the quantization error will be high. The method assumes the data within a block is somewhat locally uniform or linearly varying.
*   **Index Range:** The index must be clamped to $[0, 2^{ibits} - 1]$ during encoding to ensure valid reconstruction. If the original value is outside $[min, max]$, it should be clamped to the nearest bound [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_voxel(min_val, max_val, index, ibits):
    """
    Decompresses a single voxel value using per-block scalar quantization.
    
    Args:
        min_val (float): The minimum value of the block.
        max_val (float): The maximum value of the block.
        index (int): The local index of the voxel.
        ibits (int): The number of bits used to represent the index.
        
    Returns:
        float: The decompressed voxel value.
    """
    # Calculate the denominator based on the number of bits
    # The formula uses 2^(ibits-1) as the denominator
    denominator = 1 << (ibits - 1)
    
    # Compute the decompressed value
    # Note: The formula in the sources is: min + (max - min) * index / 2^(ibits-1)
    # This implies the index ranges from 0 to 2^(ibits-1) for a full range,
    # or potentially 0 to 2^ibits - 1 with a different scaling.
    # Based on the source formula, we strictly follow:
    value = min_val + (max_val - min_val) * (index / denominator)
    
    return value

def compress_voxel(original_val, min_val, max_val, ibits):
    """
    Compresses a single voxel value to a local index.
    
    Args:
        original_val (float): The original voxel value.
        min_val (float): The minimum value of the block.
        max_val (float): The maximum value of the block.
        ibits (int): The number of bits used to represent the index.
        
    Returns:
        int: The quantized local index.
    """
    # Calculate the denominator
    denominator = 1 << (ibits - 1)
    
    # Calculate the raw index
    # The formula is inverted: index = (v - min) / (max - min) * denominator
    if max_val == min_val:
        return 0
    
    raw_index = (original_val - min_val) / (max_val - min_val) * denominator
    
    # Clamp the index to the valid range [0, 2^ibits - 1]
    # Note: The source formula uses 2^(ibits-1) as the denominator,
    # which suggests the max index might be 2^(ibits-1) or 2^ibits - 1.
    # Standard integer quantization usually maps to [0, 2^bits - 1].
    # However, the source formula explicitly uses 2^(ibits-1).
    # We will clamp to the range implied by the denominator, but typically
    # indices are stored in ibits bits, so the max value is 2^ibits - 1.
    # The source formula might imply a specific scaling where the max index
    # corresponds to max_val. Let's assume standard integer representation
    # and that the formula is an approximation or specific to the source's
    # implementation details which are not fully elaborated.
    # To be safe and consistent with the source formula, we calculate the index
    # and then clamp it to the maximum value representable by ibits bits.
    max_index = (1 << ibits) - 1
    
    # Clamp
    index = max(0, min(int(round(raw_index)), max_index))
    
    return index
```

## Subtopics

### [Fixed-rate Block-coding (VTC/S3TC)](5727-fixed-rate-block-coding-vtc-s3tc.md)
Fixed-rate block-coding techniques, specifically Value Texture Compression (VTC) and S3TC, are hardware-supported lossy compression methods that approximate voxel values by selecting linear interpolation modes from a lookup table and applying formulas over precomputed representative values. They differ from general scalar quantization by providing on-the-fly hardware decoding that allows rendering directly from compressed data without storing decompressed volumes in memory.

## Operating characteristics
*   **Time Complexity:** Decompression is $O(1)$ per voxel, as it involves a simple arithmetic operation [S2][S3]. Encoding time decreases as the number of nodes (in the context of the Mixture Graph) decreases, which happens at lower bitrates [S1].
*   **Memory Complexity:** The output size is dominated by the voxel data at the finest level, so the impact of the quantization bitrate on the total output size is relatively small [S1]. The size of the metadata (min/max per block) is negligible for large volumes.
*   **Measured Performance:** In the Mixture Graph context, reducing the bitrate of the scalar quantizer leads to fewer nodes and faster encoding/reconstruction times, but at the cost of higher reconstruction error [S1]. For example, reducing the bitrate from 9 to 4 bits can significantly reduce the number of nodes (e.g., from ~1,029,188 to ~460,210 for the Hippocampus dataset) and reduce encoding time (e.g., from 3.51 min to 2.21 min) [S1].
*   **Strengths:** Simple to implement, efficient random-access decompression, suitable for GPU parallelization [S2][S3].
*   **Limitations:** Reconstruction error increases exponentially with bit reduction (error doubles for each bit lost) [S1]. Not suitable for data with high local variance within a block.
*   **When to use:** When random access is required and the data has local coherence. When hardware support for texture compression is available (via variants like VTC/S3TC) [S1][S2].
*   **When not to use:** When high fidelity is required with very low bitrates, or when the data has large variations within small blocks.

## Background
Scalar quantization (per-block) is a technique used in compressed GPU-based direct volume rendering. It is referenced in the context of random-access decompression methods that can be easily implemented on GPUs [S2][S3]. The method is attributed to earlier works such as YNV08 and IGM10 [S2][S3]. It is also discussed in the context of the Mixture Graph data structure for compressing, rendering, and querying segmentation histograms [S1].

## Sources
[S1] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
