# Scalar Quantization

Also called: Per-block scalar quantization

## Overview
Scalar Quantization is a random-access decompression method used in compressed GPU-based direct volume rendering to reconstruct voxel values on-the-fly [S1][S2]. It solves the problem of storing high-resolution volume data efficiently while maintaining the ability to access individual voxels without decompressing the entire dataset [S1][S2]. It sets itself apart from sibling methods by computing a local minimum and maximum per block and using a linear interpolation formula based on a local index, rather than using uniform global ranges or complex attribute-based transformations [S1][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A block of compressed volume data containing the block minimum ($min$), block maximum ($max$), and a local index ($index$) for each voxel within the block. The parameter $ibits$ defines the bit-width of the index.
*   **Output:** The decompressed scalar value $v_{ijk}$ for each voxel $(i,j,k)$ in the block.

**Data Structures:**
The data is organized in blocks. Each block stores:
1.  $min$: The minimum scalar value in the block.
2.  $max$: The maximum scalar value in the block.
3.  $index$: A sequence of integers, each occupying $ibits$ bits, representing the relative position of the voxel value within the $[min, max]$ range.

**Steps:**
1.  Retrieve the block minimum ($min$) and maximum ($max$) values for the current block.
2.  Retrieve the local index ($index$) for the target voxel.
3.  Compute the decompressed value using the linear interpolation formula:
    $$v_{ijk} = min + (max - min) \times \frac{index}{2^{ibits-1}}$$
    where:
    *   $v_{ijk}$ is the decompressed value.
    *   $min$ is the block minimum.
    *   $max$ is the block maximum.
    *   $index$ is the local index value.
    *   $ibits$ is the number of bits used to represent the index.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_voxel(min_val, max_val, index, ibits):
    """
    Decompress a single voxel value using per-block scalar quantization.
    
    Args:
        min_val (float): The minimum value in the block.
        max_val (float): The maximum value in the block.
        index (int): The local index for the voxel.
        ibits (int): The number of bits used to represent the index.
        
    Returns:
        float: The decompressed voxel value.
    """
    # Calculate the scaling factor based on the bit depth
    # The denominator is 2^(ibits-1) as per the source formula
    denominator = 1 << (ibits - 1)
    
    # Compute the normalized position of the index
    normalized_index = index / denominator
    
    # Linear interpolation between min and max
    decompressed_value = min_val + (max_val - min_val) * normalized_index
    
    return decompressed_value
```

**Edge Cases and Pitfalls:**
*   **Precision:** The division by $2^{ibits-1}$ may introduce floating-point precision errors, especially if $ibits$ is large or if $min$ and $max$ have significantly different magnitudes.
*   **Boundary Conditions:** If $min == max$, the result is always $min$, regardless of the index. This is a valid degenerate case but may indicate a block with uniform data.
*   **Index Range:** The index must be within the range $[0, 2^{ibits-1}]$. Values outside this range will produce results outside the $[min, max]$ interval, which is incorrect.

## Subtopics

## Operating characteristics
*   **Time Complexity:** Decompression is $O(1)$ per voxel, as it involves a constant number of arithmetic operations. This makes it highly suitable for parallel execution on GPUs [S1][S2].
*   **Memory Complexity:** The memory overhead per block is constant (storing $min$, $max$, and the indices). The compression ratio depends on the bit-width $ibits$ and the data distribution within the block.
*   **Performance:** Measured performance is generally high due to the simplicity of the decompression formula and the ability to process voxels in parallel [S1][S2].
*   **Strengths:** Simple implementation, low latency for random access, and efficient GPU utilization [S1][S2].
*   **Limitations:** Compression ratio is limited by the block size and the bit-width $ibits$. It may not achieve high compression ratios for data with large dynamic ranges within a block [S1][S2].
*   **When to use:** Suitable for real-time volume rendering where random access is required and moderate compression is acceptable [S1][S2].
*   **When not to use:** Not suitable for scenarios requiring very high compression ratios or where the data distribution within blocks is highly non-uniform [S1][S2].

## Background
Scalar Quantization, specifically in the context of per-block scalar quantization for volume rendering, was introduced as a random-access decompression method to facilitate efficient GPU-based direct volume rendering [S1][S2]. It builds on earlier work in thresholding and quantization models [S1][S2]. The method is cited in surveys of compressed GPU-based direct volume rendering, highlighting its role in enabling parallel decompression [S1][S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
