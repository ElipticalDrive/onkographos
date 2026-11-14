# Scalar Quantization

Also called: Per-block scalar quantization, Scalar Quantization Impact

## Overview
Scalar quantization (per-block) is a lossy compression technique that maps high-precision volume data to a lower-precision discrete domain by computing local minimum and maximum values for each data block, alongside a local index for every element within that block [S2, S3]. It solves the problem of random-access decompression in GPU-based direct volume rendering by enabling parallel reconstruction of voxel values on-the-fly without decompressing the entire volume into main memory [S2, S3]. Unlike general-purpose codecs, it is a simple, hardware-friendly method where the reconstruction error is directly controlled by the number of bits allocated to the local index [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A block of $N$ volume data elements (voxels or coefficients) with original precision (e.g., 32-bit float).
*   **Output:** A compressed block consisting of:
    1.  A global minimum value ($min$) for the block.
    2.  A global maximum value ($max$) for the block.
    3.  $N$ local indices, each stored using $ibits$ bits.

**Data Structures:**
*   **Block Header:** Stores $min$ and $max$ in their original precision (e.g., 32-bit float).
*   **Index Array:** A contiguous array of $N$ integers, each occupying $ibits$ bits. The indices are unsigned integers in the range $[0, 2^{ibits} - 1]$.

**Parameters:**
*   $ibits$: The number of bits used to represent each local index. Typical values depend on the desired compression ratio and accuracy, often ranging from 4 to 8 bits [S1, S3].
*   Block Size: The number of elements $N$ in a block. Common block sizes in GPU implementations are small (e.g., 4x4x1, 4x4x2, or 4x4x4 for VTC, though VTC is a specific fixed-rate variant [S3]).

**Algorithm Steps:**
1.  **Block Partitioning:** Divide the volume data into non-overlapping blocks of size $N$.
2.  **Compute Extrema:** For each block, find the minimum value ($min$) and the maximum value ($max$) among all $N$ elements.
3.  **Index Calculation:** For each element $v_{ijk}$ in the block, compute its local index $index$ based on its normalized position between $min$ and $max$. The source provides the *decompression* formula, implying the *compression* formula is the inverse:
    $$ index = \text{round}\left( \frac{v_{ijk} - min}{max - min} \times (2^{ibits} - 1) \right) $$
    *Note: The source formula uses $2^{ibits-1}$ in the denominator for decompression, which implies the index range is $[0, 2^{ibits-1}]$ or that the formula assumes a specific scaling. However, standard scalar quantization typically maps to $[0, 2^{ibits}-1]$. The source formula $v_{ijk} = min + (max - min) * \frac{index}{2^{ibits-1}}$ suggests the maximum index is $2^{ibits-1}$. This is a specific implementation detail. We will use the source's decompression formula as the ground truth for reconstruction.*
4.  **Storage:** Store $min$, $max$, and the $N$ indices for the block.

**Decompression (Reconstruction):**
For each element in the block, reconstruct the value $v_{ijk}$ using the stored $min$, $max$, and $index$:
$$ v_{ijk} = min + (max - min) \times \frac{index}{2^{ibits-1}} $$
where:
*   $v_{ijk}$ is the decompressed value.
*   $min$ is the block minimum.
*   $max$ is the block maximum.
*   $index$ is the local index for the element.
*   $ibits$ is the number of bits used to represent the index.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_block(values, ibits):
    """
    Compress a block of values using per-block scalar quantization.
    
    Args:
        values: List of float values in the block.
        ibits: Number of bits for the index.
        
    Returns:
        min_val: The minimum value in the block.
        max_val: The maximum value in the block.
        indices: List of integer indices.
    """
    min_val = min(values)
    max_val = max(values)
    
    # Handle edge case where min == max to avoid division by zero
    if min_val == max_val:
        indices = [0] * len(values)
        return min_val, max_val, indices
    
    # Calculate indices based on the inverse of the decompression formula
    # Decompression: v = min + (max - min) * index / (2^(ibits-1))
    # Therefore: index = (v - min) * (2^(ibits-1)) / (max - min)
    scale = (2 ** (ibits - 1)) / (max_val - min_val)
    indices = [int(round((v - min_val) * scale)) for v in values]
    
    return min_val, max_val, indices

def decompress_block(min_val, max_val, indices, ibits):
    """
    Decompress a block of values using per-block scalar quantization.
    
    Args:
        min_val: The minimum value of the block.
        max_val: The maximum value of the block.
        indices: List of integer indices.
        ibits: Number of bits used for the index.
        
    Returns:
        values: List of decompressed float values.
    """
    scale = 1.0 / (2 ** (ibits - 1))
    values = [min_val + (max_val - min_val) * (index * scale) for index in indices]
    return values
```

**Edge Cases and Pitfalls:**
*   **Min equals Max:** If all values in a block are identical, $max - min = 0$. The index calculation must handle this to avoid division by zero. In this case, all indices can be set to 0 [G].
*   **Precision Loss:** The reconstruction error roughly doubles for each reduction by one bit in the scalar quantizer [S1]. This is because each bit reduction halves the precision of the quantized interpolation weight [S1].
*   **Index Range:** The source formula uses $2^{ibits-1}$ in the denominator. This implies that the maximum index value is $2^{ibits-1}$, not $2^{ibits}-1$. This is a specific constraint of the method described in [S2, S3]. Implementers must ensure indices do not exceed this limit.

## Subtopics

### [Fixed-rate Block-coding (VTC/S3TC)](5727-fixed-rate-block-coding-vtc-s3tc.md)
A hardware-accelerated variant that extends 2D S3TC to 3D, using fixed block sizes and linear interpolation from representative values. Choose when hardware support is available and fixed-rate compression is required.

### [Uniform Quantization](5855-uniform-quantization.md)
Maps data to discrete levels with equal spacing across the entire dataset. Choose when simplicity is prioritized over local adaptivity, but be aware of potential feature loss in data with wide dynamic ranges.

### [Quantization Functions](13871-quantization-functions.md)
Converts float3 to half-precision and scales to integer range for 7, 9, or 10-bit quantization. Choose when integrating with specific neural field or rendering pipelines that require these specific bit depths.

## Operating characteristics
*   **Time Complexity:** Encoding time is dominated by finding the min/max and computing indices, which is $O(N)$ per block. Decompression is also $O(N)$ per block, involving simple arithmetic operations.
*   **Memory Complexity:** The compressed size is $2 \times P + N \times ibits$ bits per block, where $P$ is the precision of the min/max values (e.g., 32 bits) and $N$ is the block size. This results in significant compression ratios, especially for large $N$ and small $ibits$.
*   **Measured Performance:** In the context of the Mixture Graph method, lower bitrates (fewer bits for quantization) lead to fewer nodes in the graph, reducing memory usage and encoding time, but increasing reconstruction error [S1]. For example, reducing bits from 8 to 4 can significantly reduce the number of nodes, but the error doubles for each bit reduction [S1].
*   **Strengths:** Simple implementation, parallelizable on GPU, supports random-access decompression, and provides a direct trade-off between compression ratio and accuracy via $ibits$ [S2, S3].
*   **Limitations:** Lossy compression, error increases with lower bitrates, and performance is sensitive to the dynamic range within each block (blocks with large min-max ranges may have higher quantization error for the same $ibits$) [S1].
*   **When to Use:** Suitable for GPU-based direct volume rendering where random access is required and hardware acceleration is beneficial [S2, S3].
*   **When Not to Use:** When lossless compression is required or when the data has very high local variance that cannot be well-approximated by linear interpolation within small blocks.

## Background
Scalar quantization (per-block) is a foundational technique in compressed volume rendering, particularly for GPU-based direct volume rendering. It was highlighted in surveys of compressed GPU-based direct volume rendering as a method for random-access decompression [S2, S3]. The method is often associated with implementations that compute local min/max values and indices for each block, allowing for efficient on-the-fly reconstruction [S2, S3]. The impact of scalar quantization on performance, particularly in the context of the Mixture Graph data structure, has been studied, showing that lower bitrates reduce the number of nodes and encoding time but increase reconstruction error [S1].

## Sources
[S1] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
