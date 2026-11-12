# Variable-Length Coding


## Overview
Variable-Length Coding (VLC) is a lossless compression technique for volume data that reduces storage by encoding voxel values relative to a base value within a data block (brick) [S1]. It solves the problem of inefficient fixed-width storage when data values in a local region have a small dynamic range [S1]. Unlike its parent [Entropy Encoding](5360-entropy-encoding.md), which assigns codes based on symbol frequency, VLC assigns a fixed bit depth $n$ to an entire brick based on its local value range [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A brick (a sub-volume) containing voxel values.
*   **Output:** A compressed representation containing a base value, the bit depth $n$, and a bit-packed stream of differences.

**Data Structures:**
The compressed brick stores three components [S1]:
1.  **Base Value:** The minimum voxel value in the brick, stored with full source precision.
2.  **Bit Depth ($n$):** An integer $< 16$ indicating the number of bits used per voxel.
3.  **Differences:** A bit-stream where each voxel stores its offset from the base value using exactly $n$ bits.

**Steps:**
1.  Scan the brick to find the minimum value ($min$) and the maximum value ($max$) [S1].
2.  Check if the range satisfies $max - min < 2^n$ for a supported integer $n < 16$ [S1].
3.  Store $min$ as the base value [S1].
4.  Compute the difference $d_i = v_i - min$ for each voxel $v_i$ [S1].
5.  Pack each $d_i$ into exactly $n$ bits [S1].
6.  For uniform bricks where $max = min$, store only the base value and omit the difference stream, as all deltas are zero bits [S1].

**Formulas:**
*   Reduction condition: $max - min < 2^n$ where $n < 16$ [S1].
*   Offset calculation: $d_i = v_i - min$ [S1].
*   Storage per voxel: $n$ bits for the difference from the base value [S1].

**Parameters:**
*   **Supported Bit Lengths:** GPU kernels support exactly 16, 8, 4, and 0 bits; supporting all possible lengths is not justified due to cost versus compression ratio gain [S1].
*   **Bit Length $n$:** Must be an integer strictly less than 16 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variable_length_encode(brick):
    """
    Compress a brick using variable-length coding.
    brick: list of voxel values (integers or floats)
    Returns: (base_value, bit_depth, packed_bytes)
    """
    if not brick:
        return None, 0, b''
    
    min_val = min(brick)
    max_val = max(brick)
    range_val = max_val - min_val
    
    # Handle uniform bricks: store only base, omit differences
    if range_val == 0:
        return min_val, 0, b''
    
    # Supported bit lengths for GPU kernels
    supported = [16, 8, 4, 0]
    n = None
    for candidate in supported:
        if candidate == 0:
            continue
        if range_val < (1 << candidate):
            n = candidate
            break
            
    if n is None:
        raise ValueError("Range too large for supported bit depths (n < 16)")
    
    # Compute differences
    diffs = [v - min_val for v in brick]
    
    # Pack differences into bytes
    packed = bytearray()
    bit_buffer = 0
    bits_remaining = 0
    for val in diffs:
        bit_buffer = (bit_buffer << n) | val
        bits_remaining += n
        if bits_remaining >= 8:
            packed.append(bit_buffer >> (bits_remaining - 8))
            bits_remaining -= 8
    if bits_remaining > 0:
        packed.append(bit_buffer << (8 - bits_remaining))
        
    return min_val, n, bytes(packed)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The base value must be stored with full source precision to avoid quantization errors during reconstruction [G].
*   **Boundary Conditions:** If $max - min \ge 2^{16}$, the brick cannot be compressed with $n < 16$ and must fall back to uncompressed storage [S1].
*   **GPU Kernel Implementation:** Individual CUDA kernels are required for each supported bit length (16, 8, 4, 0) because memory addressing modes differ [S1].

## Subtopics

### [Variable Bit Length Encoding](7195-variable-bit-length-encoding.md)
A variant that uses a hierarchical structure (e.g., quadtree) to determine bit lengths locally per sub-block, offering finer granularity than global brick-based VLC.

## Operating characteristics
*   **Time Complexity:** Determining min/max and packing differences is $O(N)$, where $N$ is the number of voxels per brick [G].
*   **Memory Complexity:** Storage reduces from $16 \times N$ bits to $n \times N + \text{overhead}$ bits, where $n < 16$ [G].
*   **Measured Performance:** The method achieves higher compression ratios on temporal delta values than on absolute values, because temporal deltas are typically smaller [S1].
*   **Strengths:** Simple to implement, effective for data with local uniformity or small dynamic range, and compatible with GPU acceleration via specialized kernels [S1].
*   **Limitations:** Restricted to 16, 8, 4, and 0 bits, ineffective for bricks with large dynamic ranges, and requires separate kernels for different bit depths [S1].
*   **When to Use:** When volume data exhibits local smoothness or uniformity, and when GPU decompression speed is critical [G].
*   **When Not to Use:** When data has high local variance or when the overhead of managing multiple bit depths outweighs the compression gain [G].

## Background
Variable-Length Coding for volume data was introduced in a 2010 Eurographics paper for GPU-supported lossless compression of time-varying volume data to address increased storage requirements [S1]. The method is applied alongside temporal prediction and brick assembly in a hybrid CPU/GPU decompression pipeline [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
