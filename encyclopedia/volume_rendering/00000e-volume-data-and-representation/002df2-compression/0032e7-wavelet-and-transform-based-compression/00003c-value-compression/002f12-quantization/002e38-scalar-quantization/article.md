# Scalar Quantization
Also called: Per-block scalar quantization

## Overview
Scalar Quantization (Per-Block) is a random-access decompression method used in compressed volume rendering that maps voxel values to a discrete index within a local range defined by the minimum and maximum values of a data block [S1, S2]. It differs from global quantization methods by computing block-specific extrema, allowing for better fidelity on data with high local variance, and differs from hardware-specific block coding (like S3TC) by using a simple linear interpolation formula rather than complex lookup tables or mode selection [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A block of compressed data containing the block minimum ($min$), block maximum ($max$), and a sequence of local indices ($index$) for each voxel in the block.
*   **Output:** The decompressed scalar value ($v_{ijk}$) for each voxel in the block.

**Data Structures:**
The compressed block typically stores:
1.  $min$: The minimum scalar value in the block (original bit depth).
2.  $max$: The maximum scalar value in the block (original bit depth).
3.  $index$: A bit-packed array of integers, where each integer uses $ibits$ bits. The number of bits $ibits$ is determined by the target compression ratio (e.g., if the original data is 32-bit float and the target is 8-bit index, $ibits=8$).

**Steps:**
1.  **Compute Extrema:** Identify the minimum ($min$) and maximum ($max$) values within the current block of voxel data.
2.  **Compute Local Index:** For each voxel, determine its local index ($index$) based on its value relative to the block's range. This is typically done during compression by mapping the original value to the nearest integer in the range $[0, 2^{ibits} - 1]$.
3.  **Reconstruct Value:** Calculate the decompressed value using linear interpolation between $min$ and $max$ based on the local index.

**Formula:**
The decompressed value $v_{ijk}$ is calculated as [S1, S2]:

$$
v_{ijk} = min + (max - min) \times \frac{index}{2^{ibits} - 1}
$$

Where:
*   $v_{ijk}$ is the decompressed scalar value for the voxel at coordinates $(i, j, k)$.
*   $min$ is the minimum value in the block.
*   $max$ is the maximum value in the block.
*   $index$ is the local index stored in the compressed data.
*   $ibits$ is the number of bits used to represent the index values.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_scalar_quantization_block(min_val, max_val, indices, ibits):
    """
    Decompresses a block of voxel data using per-block scalar quantization.
    
    Args:
        min_val (float): The minimum value in the block.
        max_val (float): The maximum value in the block.
        indices (list[int]): List of local indices for each voxel.
        ibits (int): Number of bits used to represent each index.
        
    Returns:
        list[float]: List of decompressed voxel values.
    """
    # Calculate the maximum possible index value (2^ibits - 1)
    max_index = (1 << ibits) - 1
    
    # Handle edge case where min equals max to avoid division by zero
    if min_val == max_val:
        return [min_val] * len(indices)
    
    decompressed_values = []
    for index in indices:
        # Clamp index to valid range [0, max_index] to handle potential encoding errors
        index = max(0, min(index, max_index))
        
        # Linear interpolation formula
        value = min_val + (max_val - min_val) * (index / max_index)
        decompressed_values.append(value)
        
    return decompressed_values
```

**Parameters:**
*   $ibits$: The number of bits per index. Common values depend on the desired compression ratio. For example, if the original data is 32-bit and the block size is 64 voxels, storing 8-bit indices ($ibits=8$) yields a significant compression ratio. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Constant Blocks:** If $min = max$, the denominator in the interpolation formula becomes zero. The implementation must handle this by returning $min$ for all voxels in the block [G].
*   **Precision:** The reconstruction error is bounded by the quantization step size $\phi = (max - min) / (2^{ibits} - 1)$. Floating-point rounding errors may occur during the multiplication and addition steps, but are generally negligible for standard volume rendering applications [G].
*   **Out-of-Range Indices:** If the compression process produces an index outside the valid range $[0, 2^{ibits} - 1]$, it must be clamped to prevent invalid memory access or extreme values [G].

## Subtopics

### [Fixed-rate Block-coding (VTC/S3TC)](5727-fixed-rate-block-coding-vtc-s3tc.md)
Fixed-rate block-coding techniques, specifically Value Texture Compression (VTC) and S3TC, are hardware-supported lossy compression methods that approximate voxel values by selecting linear interpolation modes from a lookup table and applying formulas over precomputed representative values. They differ from general scalar quantization by providing on-the-fly hardware decoding that allows rendering directly from compressed data without storing decompressed volumes in memory.

## Operating characteristics
*   **Time Complexity:** Decompression is $O(1)$ per voxel, as it involves a constant number of arithmetic operations (subtraction, multiplication, division) per voxel. This makes it highly suitable for parallel execution on GPUs [S1, S2].
*   **Memory Complexity:** The compressed size is proportional to the number of voxels in the block times the number of bits per index ($ibits$), plus the overhead of storing $min$ and $max$ per block.
*   **Performance:** The method is efficient for random-access decompression, allowing individual voxels to be decompressed in parallel without decoding the entire block [S1, S2].
*   **Strengths:** Simple implementation, low computational overhead during decompression, and better fidelity than global quantization for data with local variations.
*   **Limitations:** Compression ratio is limited by the block size and the number of bits per index. It may not perform well on data with large global variations but small local variations, or vice versa.
*   **When to use:** Suitable for GPU-based direct volume rendering where random access is required and hardware support for simple arithmetic is available.
*   **When not to use:** Not suitable for scenarios requiring very high compression ratios where more advanced techniques (like wavelet-based compression) are needed, or for data with complex structures that cannot be well-approximated by linear interpolation within small blocks.

## Background
The method was described in the context of compressed GPU-based direct volume rendering, where random-access decompression is essential for efficient rendering [S1, S2]. It is based on the principles of thresholding and quantization models, specifically per-block scalar quantization [YNV08, IGM10]. The approach allows each voxel to be decompressed in parallel by accessing corresponding codewords and their weighting coefficients, as proposed by Gobbetti et al. [GIM12].

## Sources
[S1] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
