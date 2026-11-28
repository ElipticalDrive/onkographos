# BC5 Compression
Also called: BC5
## Overview
BC5 is a block compression format for 2-channel (RG) textures that compresses 4×4 blocks (32 bytes) into 16 bytes [S1, unofficial]. It supports two variants: BC5_UNORM, which outputs floating-point values in the range [0, 1], and BC5_SNORM, which outputs values in the range [−1, 1] [S1, unofficial]. It builds on [Texture Format Compression](5623-texture-format-compression.md) by applying independent 1-channel compression to each channel within a single 4×4 block [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 4×4 block of 16 unsigned 8-bit integer values per channel (32 bytes total for two channels) [S1, unofficial].
- Output: A 16-byte compressed block [S1, unofficial].

**Data Structures:**
The 16-byte block stores each channel independently: 8 bytes for the first channel followed by 8 bytes for the second channel [S1, unofficial]. Each channel's 8-byte layout is:
1. Byte 0: Minimum value for the channel (8-bit) [S1, unofficial].
2. Byte 1: Maximum value for the channel (8-bit) [S1, unofficial].
3. Bytes 2–7: 16 indices, each 3 bits wide, packed into 6 bytes [S1, unofficial].
The indices are arranged in row-major order across two rows of 8 pixels [S1, unofficial]. The bit packing follows the layout:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```
/*
MSB                   LSB
| red_0               | 1 byte minimum color
| red_1               | 1 byte maximum color
| red_h | red_g | red_f | red_e | red_d | red_c | red_b | red_a | 3 bits x 8
| red_p | red_o | red_n | red_m | red_l | red_k | red_j | red_i | 3 bits x 8
*/
```
Each channel is compressed independently using this layout [S1, unofficial].

**Algorithm Steps:**
For each channel independently:
1. **Find Min/Max:** Let $V$ be the array of 16 input values for the channel [S1, unofficial]. Compute $min = \min(V)$ and $max = \max(V)$ [S1, unofficial].
2. **Scale Min/Max:** Define a scaling constant $Shift = 20$ [S1, unofficial]. Compute $c_0 = min \ll Shift$ and $c_1 = max \ll Shift$ [S1, unofficial]. The values $c_0$ and $c_1$ are stored as 32-bit signed integers [S1, unofficial].
3. **Generate Color Table:** Determine the interpolation mode based on the relationship between $c_0$ and $c_1$ [S1, unofficial]. The reference implementation checks `if (c_1 < c_0)` [S1, unofficial]. Since $min \le max$, $c_0 \le c_1$ normally, making this condition false for standard inputs [S1, unofficial].
   - **6-value mode** (taken when `c_1 < c_0`):
     $$c_2 = \left\lfloor \frac{6 c_0 + 1 c_1}{7} \right\rfloor$$
     $$c_3 = \left\lfloor \frac{5 c_0 + 2 c_1}{7} \right\rfloor$$
     $$c_4 = \left\lfloor \frac{4 c_0 + 3 c_1}{7} \right\rfloor$$
     $$c_5 = \left\lfloor \frac{3 c_0 + 4 c_1}{7} \right\rfloor$$
     $$c_6 = \left\lfloor \frac{2 c_0 + 5 c_1}{7} \right\rfloor$$
     $$c_7 = \left\lfloor \frac{1 c_0 + 6 c_1}{7} \right\rfloor$$
     All 8 values are distinct and strictly between $c_0$ and $c_1$ when $c_0 < c_1$ [S1, unofficial].
   - **4-value mode** (taken when `c_1 >= c_0`):
     $$c_2 = \left\lfloor \frac{4 c_0 + 1 c_1}{5} \right\rfloor$$
     $$c_3 = \left\lfloor \frac{3 c_0 + 2 c_1}{5} \right\rfloor$$
     $$c_4 = \left\lfloor \frac{2 c_0 + 3 c_1}{5} \right\rfloor$$
     $$c_5 = \left\lfloor \frac{1 c_0 + 4 c_1}{5} \right\rfloor$$
     $$c_6 = 0$$
     $$c_7 = 255 \ll Shift$$
     In this mode, indices 0 and 1 represent the endpoints, and indices 2–5 represent interpolated values [S1, unofficial]. Indices 6 and 7 are defined as 0 and the scaled maximum respectively in the source implementation [S1, unofficial].
4. **Encode Indices:** For each of the 16 pixels in the block, find the index $i \in \{0, \dots, 7\}$ such that $|c_i - V[p]|$ is minimized [S1, unofficial]. Pack these 16 indices into the 6-byte index field [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_bc5_channel(block: list[int]) -> tuple[int, int, int]:
    """
    Compress a single channel of 16 8-bit values.
    Returns (min_val, max_val, packed_indices_48bit)
    """
    # 1. Find min and max
    min_val = min(block)
    max_val = max(block)
    
    # 2. Scale
    Shift = 20
    c0 = min_val << Shift
    c1 = max_val << Shift
    
    # 3. Generate color table
    colors = [0] * 8
    colors[0] = c0
    colors[1] = c1
    
    # Check condition from source: if (c1 < c0)
    if c1 < c0:
        # 6-value mode
        colors[2] = (6 * c0 + 1 * c1) // 7
        colors[3] = (5 * c0 + 2 * c1) // 7
        colors[4] = (4 * c0 + 3 * c1) // 7
        colors[5] = (3 * c0 + 4 * c1) // 7
        colors[6] = (2 * c0 + 5 * c1) // 7
        colors[7] = (1 * c0 + 6 * c1) // 7
    else:
        # 4-value mode
        colors[2] = (4 * c0 + 1 * c1) // 5
        colors[3] = (3 * c0 + 2 * c1) // 5
        colors[4] = (2 * c0 + 3 * c1) // 5
        colors[5] = (1 * c0 + 4 * c1) // 5
        colors[6] = 0
        colors[7] = 255 << Shift
        
    # 4. Find nearest indices
    indices = []
    for val in block:
        best_idx = 0
        best_diff = abs(colors[0] - val)
        for i in range(1, 8):
            diff = abs(colors[i] - val)
            if diff < best_diff:
                best_diff = diff
                best_idx = i
        indices.append(best_idx)
        
    # 5. Pack indices into 6 bytes (48 bits)
    packed = 0
    for i, idx in enumerate(indices):
        packed |= (idx << (3 * i))
        
    return min_val, max_val, packed

def compress_bc5_block(r_block: list[int], g_block: list[int]) -> bytes:
    """
    Compress a 4x4 BC5 block.
    r_block and g_block are lists of 16 8-bit integers.
    Returns 16 bytes.
    """
    r_min, r_max, r_indices = compress_bc5_channel(r_block)
    g_min, g_max, g_indices = compress_bc5_channel(g_block)
    
    result = bytearray(16)
    
    # R channel (bytes 0-7)
    result[0] = r_min
    result[1] = r_max
    for i in range(6):
        result[2 + i] = (r_indices >> (8 * i)) & 0xFF
        
    # G channel (bytes 8-15)
    result[8] = g_min
    result[9] = g_max
    for i in range(6):
        result[10 + i] = (g_indices >> (8 * i)) & 0xFF
        
    return bytes(result)
```

**Parameters:**
- `Shift`: 20 [S1, unofficial].

**Edge Cases and Pitfalls:**
- The condition `c_1 < c_0` in the source code is logically impossible for valid min/max values unless overflow occurs or specific edge cases are handled differently [S1, unofficial].
- In 4-value mode, indices 6 and 7 are defined as 0 and the scaled maximum respectively in the source code, but standard BC5 decoders may not use these indices or may map them differently [S1, unofficial].
- Implementers should verify compatibility with target hardware decoders [S1, unofficial].
- Precision: The use of 32-bit integers for scaled values ensures sufficient precision for 8-bit input data [S1, unofficial].

## Operating characteristics
- **Compression Ratio:** 2:1 (32 bytes to 16 bytes) [S1, unofficial].
- **Time Complexity:** $O(1)$ per block due to the fixed 16-pixel block size [G].
- **Memory Complexity:** $O(1)$ per block [G].
- **Strengths:** Simple to implement, fast compression and decompression, suitable for height maps or normal maps where two channels are correlated or independent [G].
- **Limitations:** Block artifacts are visible at low bitrates, and it is not suitable for high-quality photographic textures [G]. The source implementation notes that results differ from GPUOpen Compressonator, suggesting potential non-compliance or optimization differences [S1, unofficial].
- **When to use:** When memory bandwidth is critical and visual quality is secondary, or for specific data types like displacement maps [G].
- **When not to use:** When high visual fidelity is required, or when compatibility with standard GPU texture units is mandatory without custom decompression shaders [G].
- **Measured Performance:** Not given in the sources.

## Background
BC5 is part of the S3 Texture Compression (S3TC) family, originally developed by S3 Graphics, and is widely supported by modern GPUs [G]. The provided implementation originates from a 2021 blog post describing a custom compression routine [S1, unofficial].

## Sources
[S1] BC5 圧縮テクスチャ, https://qiita.com/taqu/items/651ca51f6c46876fe9c1, fetched 2026-10-28 (unofficial)
