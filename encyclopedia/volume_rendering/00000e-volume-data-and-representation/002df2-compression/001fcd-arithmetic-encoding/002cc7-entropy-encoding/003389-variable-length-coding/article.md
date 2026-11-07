# Variable-Length Coding
Also called: Variable Bit Length Coding, Delta Encoding with Variable Bit Depth

## Overview
Variable-Length Coding (VLC) is a lossless compression technique used in volume rendering that reduces storage requirements by exploiting the local value range of data bricks. It solves the problem of inefficient fixed-width storage when voxel values within a brick are clustered within a narrow range [S1]. Unlike its parent [Entropy Encoding](5360-entropy-encoding.md), which maps symbols to bit lengths based on frequency, VLC maps symbols to bit lengths based on the numerical range of the data block, storing a base value and narrow-range differences [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A brick (sub-volume) of voxel data.
*   **Output:** A compressed bitstream containing a base value and $n$-bit difference values for each voxel.

**Data Structures:**
The compressed representation of a brick consists of:
1.  **Base Value:** The minimum value found in the brick, stored using the original data type's bit width (typically 16 bits for short integers, as $n < 16$) [S1].
2.  **Difference Values:** For each voxel, the difference between the voxel value and the base value, stored using $n$ bits [S1].

**Steps:**
1.  Determine the minimum ($min$) and maximum ($max$) values within the brick [S1].
2.  Calculate the range $R = max - min$ [S1].
3.  Determine the smallest integer $n$ such that $R < 2^n$ and $n < 16$ [S1].
4.  If the brick is uniform (all voxels have the same value), store only the base value (the voxel value). The difference values are implicitly zero and omitted from the bitstream [S1].
5.  If the brick is not uniform:
    a. Store the base value ($min$) [S1].
    b. For each voxel $v_i$ in the brick, compute the difference $d_i = v_i - min$ [S1].
    c. Store $d_i$ using exactly $n$ bits [S1].

**Formula:**
The condition for applying variable-length coding to reduce storage is:
$$ max - min < 2^n $$
where $n$ is the number of bits used to store each difference value, and $n < 16$ [S1].

The storage size per voxel becomes $n$ bits, compared to the original fixed width (e.g., 16 bits) [S1].

**Parameters:**
*   **Supported Bit Lengths ($n$):** 16, 8, 4, and 0 bits [S1].
*   **Rationale for Limited Bit Lengths:** Supporting all possible bit lengths is not justified due to hardware cost versus compression ratio gain. Analysis of test data showed that supporting only 16, 8, 4, and 0 bits achieves a compression ratio close to optimal [S1].

**Edge Cases and Pitfalls:**
*   **Uniform Bricks:** Handled directly by storing only the base value, effectively using 0 bits for differences [S1].
*   **GPU Implementation:** Because different bit lengths require different addressing modes for memory access, individual CUDA kernels are implemented for each supported bit length (16, 8, 4, 0) [S1].
*   **Temporal Data:** VLC is often applied to delta values (temporal prediction residuals) rather than absolute values. Since delta values are typically smaller than absolute values, VLC achieves higher compression ratios on deltas [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variable_length_encode(brick):
    """
    Compress a brick of voxel data using variable-length coding.
    brick: list of integers (voxel values)
    Returns: tuple (base_value, n_bits, differences)
    """
    if not brick:
        return None, 0, []

    min_val = min(brick)
    max_val = max(brick)
    range_val = max_val - min_val

    # Determine n such that range_val < 2^n and n < 16
    # If range_val is 0, n=0 (uniform brick)
    if range_val == 0:
        n = 0
    else:
        n = 0
        while (1 << n) <= range_val:
            n += 1
            if n >= 16:
                # Fallback or error if range exceeds 16-bit capacity
                # In practice, this case might be handled by a different scheme
                break 
    
    # If n >= 16, VLC might not be beneficial or supported
    # Assuming n < 16 for valid VLC application
    if n >= 16:
        return None, 16, brick # Fallback to 16-bit storage

    differences = [v - min_val for v in brick]
    return min_val, n, differences

def variable_length_decode(base_value, n_bits, differences):
    """
    Decompress a brick using variable-length coding.
    base_value: integer (min value)
    n_bits: integer (0, 4, 8, or 16)
    differences: list of integers (difference values)
    Returns: list of integers (voxel values)
    """
    if n_bits == 0:
        # Uniform brick, all values are base_value
        # Note: length of output must be known from context (brick size)
        return [base_value] * len(differences) # Assuming differences list length indicates brick size
    
    return [base_value + d for d in differences]
```

## Subtopics

### Variable Bit Length Encoding
A variant where a quadtree structure is built for each data slice of the volume data. The quadtree allows for adaptive bit-length encoding across different regions of the slice. This method usually needs to be combined with Run-Length Encoding (RLE) to store the encoding bit length of each voxel efficiently [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Determining min/max is $O(N)$ where $N$ is the number of voxels in the brick. Encoding/decoding is $O(N)$ with simple arithmetic. GPU implementation requires separate kernels for each bit length, adding launch overhead [S1].
*   **Memory Complexity:** Storage is reduced from $W$ bits per voxel to $n$ bits per voxel, where $n < W$ (typically 16). The base value adds negligible overhead per brick [S1].
*   **Performance:** Achieves high compression ratios when applied to delta values (temporal prediction residuals) because these values are small and clustered [S1].
*   **Strengths:** Simple to implement; highly effective for data with local value clustering; compatible with GPU hardware acceleration via specialized kernels [S1].
*   **Limitations:** Requires separate kernels for different bit lengths, increasing code complexity and potential memory fragmentation; less effective if the brick range is large (requiring $n$ close to original bit width) [S1].
*   **When to Use:** When volume data bricks exhibit small local value ranges, especially after temporal prediction [S1].
*   **When Not to Use:** When the data range within bricks is uniformly large, or when hardware does not support efficient variable-bit addressing [S1].

## Background
Variable-Length Coding for volume data was introduced in the context of GPU-supported lossless compression for rendering time-varying volume data. The method was proposed by Klajnsek et al. in 2010 [S1]. It is part of a hybrid CPU/GPU decompression pipeline where data packets are resolved on the GPU before being copied to a CUDA array for raycasting [S1]. The approach leverages the fact that voxel values in a dataset are not uniformly distributed, allowing bricks with small ranges to be compressed significantly [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
