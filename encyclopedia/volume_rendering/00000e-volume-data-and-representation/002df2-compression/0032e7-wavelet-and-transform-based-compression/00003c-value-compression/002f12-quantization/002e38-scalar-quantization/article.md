# Scalar Quantization
Also called: Per-block scalar quantization

## Overview
Scalar Quantization (Per-Block) is a random-access decompression method used in compressed GPU-based direct volume rendering to reconstruct voxel values on-the-fly [S1][S2]. It solves the problem of storing high-resolution volume data by mapping a block of voxels to a local range defined by block-specific minimum and maximum values, allowing parallel decompression without loading the entire volume into memory [S1][S2]. It differs from [Uniform Quantization](5855-uniform-quantization.md) by adapting the quantization range to local data statistics per block rather than using a global range, and from [Fixed-rate Block-coding (VTC/S3TC)](5727-fixed-rate-block-coding-vtc-s3tc.md) by relying on simple linear interpolation rather than hardware-specific lookup tables and interpolation modes [S1][S2].

## How it works
**Inputs:** A compressed block of voxel data containing the block minimum ($min$), block maximum ($max$), and a set of local indices ($index$) for each voxel within the block.
**Outputs:** The decompressed scalar value ($v_{ijk}$) for each voxel in the block.

**Data Structures:**
The compressed block layout typically consists of:
1.  **Header:** Two floating-point values (or fixed-point equivalents) representing $min$ and $max$.
2.  **Index Data:** A bit-packed array where each voxel is represented by $ibits$ bits. The total number of bits per voxel is determined by the desired compression ratio.

**Steps:**
1.  **Compute Block Extremes:** Identify the minimum ($min$) and maximum ($max$) values among all voxels in the current block. [S1][S2]
2.  **Compute Local Index:** For each voxel, determine its local index ($index$) based on its original value relative to the block's range. This index is stored as an integer using $ibits$ bits. [S1][S2]
3.  **Reconstruct Value:** During rendering or decompression, compute the final value using linear interpolation between $min$ and $max$ based on the stored index. [S1][S2]

**Formula:**
The decompressed value $v_{ijk}$ is calculated as:
$$v_{ijk} = min + (max - min) * \frac{index}{2^{ibits-1}}$$
Where:
*   $v_{ijk}$ is the decompressed value of the voxel at coordinates $(i, j, k)$.
*   $min$ is the minimum value in the block.
*   $max$ is the maximum value in the block.
*   $index$ is the local integer index stored for the voxel.
*   $ibits$ is the number of bits used to represent the index values. [S1][S2]

**Parameters:**
*   $ibits$: The bit-depth for the index. Common values depend on the block size and target compression ratio. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero Range:** If $min = max$, the denominator $2^{ibits-1}$ remains valid, but the term $(max - min)$ becomes zero, resulting in all voxels having the value $min$. This is mathematically correct but may indicate a degenerate block.
*   **Precision Loss:** The linear interpolation assumes a uniform distribution of values within the block. If the data distribution is highly non-uniform, quantization error increases. [S1][S2]
*   **Boundary Conditions:** The formula maps the index range $[0, 2^{ibits}-1]$ to $[min, max]$. Note that the maximum index $2^{ibits}-1$ maps to $min + (max - min) * \frac{2^{ibits}-1}{2^{ibits-1}}$, which is slightly less than $max$ if $ibits > 1$. The sources define the formula specifically with $2^{ibits-1}$ in the denominator, which effectively scales the index to a range of $[0, 2]$ before adding to min, or implies a specific normalization. However, standard linear interpolation usually uses $2^{ibits}-1$ as the denominator to map the full range. The sources explicitly state $2^{ibits-1}$ [S1][S2]. Implementers should verify if this is a typo in the source or a specific design choice (e.g., leaving headroom). Given the strict instruction to follow sources, the formula uses $2^{ibits-1}$. [G] *Note: Standard linear quantization typically uses $2^{ibits}-1$ as the denominator to map the full integer range to the float range. The source formula uses $2^{ibits-1}$. This may result in the maximum index not reaching the exact max value. Implementers should check if the source implies a specific offset or if this is a known approximation.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_block(block_min, block_max, indices, ibits):
    """
    Decompresses a block of voxels using per-block scalar quantization.
    
    Args:
        block_min (float): The minimum value in the block.
        block_max (float): The maximum value in the block.
        indices (list of int): List of local indices for each voxel.
        ibits (int): Number of bits used to represent each index.
        
    Returns:
        list of float: List of decompressed voxel values.
    """
    # Calculate the scaling factor as per the source formula
    # Note: Source formula uses 2^(ibits-1) in the denominator
    denominator = 2 ** (ibits - 1)
    
    decompressed_values = []
    for index in indices:
        # Linear interpolation between min and max
        # v_ijk = min + (max - min) * (index / 2^(ibits-1))
        value = block_min + (block_max - block_min) * (index / denominator)
        decompressed_values.append(value)
        
    return decompressed_values
```

## Subtopics

### [Fixed-rate Block-coding (VTC/S3TC)](5727-fixed-rate-block-coding-vtc-s3tc.md)
Fixed-rate block-coding techniques, specifically Value Texture Compression (VTC) and S3TC, are hardware-supported lossy compression methods that approximate voxel values by selecting linear interpolation modes from a lookup table and applying formulas over precomputed representative values. They differ from general scalar quantization by providing on-the-fly hardware decoding that allows rendering directly from compressed data without storing decompressed volumes in memory.

## Operating characteristics
*   **Time Complexity:** Decompression is $O(1)$ per voxel, as it involves a simple arithmetic operation. This allows for parallel execution on GPUs. [S1][S2]
*   **Memory Complexity:** The compression ratio depends on the block size and $ibits$. For a block of $N$ voxels, the overhead is 2 values (min, max) plus $N \times ibits$ bits for indices. [S1][S2]
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Simple implementation, random access capability, suitable for GPU parallelization. [S1][S2]
*   **Limitations:** Quantization error can be high if the data distribution within a block is not uniform. The choice of $ibits$ affects compression ratio and accuracy. [S1][S2]
*   **When to use:** When random access is required and the data exhibits local coherence. [S1][S2]
*   **When not to use:** When high fidelity is required for data with large local variations that cannot be captured by linear interpolation. [S1][S2]

## Background
The method is described as a random-access decompression method easily implemented in GPUs as a per-block scalar quantization [YNV08, IGM10] [S1][S2]. It is part of the broader category of thresholding and quantization models for compressed GPU-based direct volume rendering [S1][S2]. The specific formula and implementation details are cited from these sources. [S1][S2]

## Sources
[S1] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
