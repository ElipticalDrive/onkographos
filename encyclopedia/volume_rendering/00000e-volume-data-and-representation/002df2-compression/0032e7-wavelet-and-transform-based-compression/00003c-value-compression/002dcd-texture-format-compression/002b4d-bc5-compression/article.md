# BC5 Compression

## Overview
BC5 is a block compression format for 2-channel (RG) textures, compressing 4x4 blocks (32 bytes) into 16 bytes [S1, unofficial]. It supports two variants: BC5_UNORM (outputs [0, 1] floats) and BC5_SNORM (outputs [-1, 1] floats) [S1, unofficial]. It builds on [Texture Format Compression](5623-texture-format-compression.md) by applying independent 1-channel BC4-like compression to each of the two channels within a single 4x4 block.

## How it works
**Inputs and Outputs:**
- Input: A 4x4 block of 16 unsigned 8-bit integer values per channel (32 bytes total for 2 channels).
- Output: A 16-byte compressed block.

**Data Structures:**
The compressed block layout is fixed at 16 bytes:
1. Byte 0: Minimum value for the channel (8-bit).
2. Byte 1: Maximum value for the channel (8-bit).
3. Bytes 2-7: 16 indices, each 3 bits wide, packed into 6 bytes. The indices are arranged in row-major order (16 pixels). The bit packing follows the layout:
   ```
   /*
   MSB                   LSB
   | red_0               | 1 byte minimum color
   | red_1               | 1 byte maximum color
   | red_h | red_g | red_f | red_e | red_d | red_c | red_b | red_a | 3 bits x 8
   | red_p | red_o | red_n | red_m | red_l | red_k | red_j | red_i | 3 bits x 8
   */
   ```
   Each channel is compressed independently using this layout.

**Algorithm Steps:**
For each channel (R and G) independently:

1. **Find Min/Max:**
   Let $V$ be the array of 16 input values for the channel.
   $$ min = \min(V) $$
   $$ max = \max(V) $$

2. **Scale Min/Max:**
   Define a scaling constant $Shift = 20$ [S1, unofficial].
   $$ c_0 = min \ll Shift $$
   $$ c_1 = max \ll Shift $$
   Here, $c_0$ and $c_1$ are 32-bit signed integers (assuming 8-bit input, $min, max \in [0, 255]$, so $c_0, c_1 \in [0, 255 \cdot 2^{20}]$).

3. **Generate Color Table:**
   Determine the interpolation mode based on the relationship between $c_0$ and $c_1$. Note: The source code checks `if (c_1 < c_0)` [S1, unofficial]. Since $min \le max$, $c_0 \le c_1$ normally. The condition `c_1 < c_0` is technically impossible unless there is overflow or specific edge case handling not fully explained in the source, but the code implements two branches [S1, unofficial].
   
   - **Case A: 6-value mode** (Taken when `c_1 < c_0` in the reference implementation, likely intended for specific edge cases or min==max scenarios where standard 4-value might be suboptimal, though standard BC5 typically uses 4-value for equal min/max) [S1, unofficial]:
     $$ c_2 = \lfloor \frac{6 c_0 + 1 c_1}{7} \rfloor $$
     $$ c_3 = \lfloor \frac{5 c_0 + 2 c_1}{7} \rfloor $$
     $$ c_4 = \lfloor \frac{4 c_0 + 3 c_1}{7} \rfloor $$
     $$ c_5 = \lfloor \frac{3 c_0 + 4 c_1}{7} \rfloor $$
     $$ c_6 = \lfloor \frac{2 c_0 + 5 c_1}{7} \rfloor $$
     $$ c_7 = \lfloor \frac{1 c_0 + 6 c_1}{7} \rfloor $$
     The valid indices are 0-7. All 8 values are distinct and strictly between $c_0$ and $c_1$ (excluding endpoints if $c_0 < c_1$).

   - **Case B: 4-value mode** (Taken when `c_1 >= c_0`):
     $$ c_2 = \lfloor \frac{4 c_0 + 1 c_1}{5} \rfloor $$
     $$ c_3 = \lfloor \frac{3 c_0 + 2 c_1}{5} \rfloor $$
     $$ c_4 = \lfloor \frac{2 c_0 + 3 c_1}{5} \rfloor $$
     $$ c_5 = \lfloor \frac{1 c_0 + 4 c_1}{5} \rfloor $$
     $$ c_6 = 0 $$
     $$ c_7 = 255 \ll Shift $$
     In this mode, indices 0 and 1 represent the endpoints. Indices 2-5 represent interpolated values. Indices 6 and 7 are reserved/invalid in standard BC5 decoding but are defined in the source code [S1, unofficial]. *Note: Standard BC5 specification usually defines only 4 valid endpoints/interpolations for the 4-value mode, where index 6 and 7 are not used or have specific meanings. The source code defines them as 0 and max, but standard decoders may treat indices 6 and 7 as invalid or map them to specific values. The source notes that results differ from GPUOpen Compressonator, suggesting potential non-compliance or optimization differences [S1, unofficial].*

4. **Encode Indices:**
   For each of the 16 pixels in the block, find the index $i \in \{0, \dots, 7\}$ such that $|c_i - V[p]|$ is minimized.
   $$ index_p = \arg \min_{i \in \{0, \dots, 7\}} |c_i - V[p]| $$
   Pack these 16 indices into the 6-byte index field.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_bc5_channel(block: list[int]) -> tuple[int, int, int]:
    """
    Compress a single channel of 16 8-bit values.
    Returns (min_val, max_val, index_bits_64)
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
    # Note: Since min <= max, c0 <= c1. This branch is likely for edge cases
    # or specific implementation quirks. Standard BC5 uses 4-value for min==max.
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
    # 16 indices * 3 bits = 48 bits
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
    
    # Layout:
    # Byte 0: R min
    # Byte 1: R max
    # Bytes 2-7: R indices (6 bytes)
    # Byte 8: G min
    # Byte 9: G max
    # Bytes 10-15: G indices (6 bytes)
    
    result = bytearray(16)
    
    # R channel
    result[0] = r_min
    result[1] = r_max
    for i in range(6):
        result[2 + i] = (r_indices >> (8 * i)) & 0xFF
        
    # G channel
    result[8] = g_min
    result[9] = g_max
    for i in range(6):
        result[10 + i] = (g_indices >> (8 * i)) & 0xFF
        
    return bytes(result)
```

**Parameters:**
- `Shift`: 20 [S1, unofficial].

**Edge Cases and Pitfalls:**
- The condition `c_1 < c_0` in the source code is logically impossible for valid min/max values unless overflow occurs or specific edge cases are handled differently. The source notes that results differ from GPUOpen Compressonator, suggesting the current method may not be optimal or fully compliant with the standard reference implementation [S1, unofficial].
- In 4-value mode, indices 6 and 7 are defined as 0 and max respectively in the source code, but standard BC5 decoders may not use these indices or may map them differently. Implementers should verify compatibility with target hardware decoders.
- Precision: The use of 32-bit integers for scaled values ensures sufficient precision for 8-bit input data.

## Operating characteristics
- **Compression Ratio:** 2:1 (32 bytes to 16 bytes) [S1, unofficial].
- **Time Complexity:** $O(1)$ per block (fixed 16 pixels).
- **Memory Complexity:** $O(1)$ per block.
- **Strengths:** Simple to implement, fast compression/decompression, good for height maps or normal maps where two channels are correlated or independent.
- **Limitations:** Block artifacts visible at low bitrates, not suitable for high-quality photographic textures. The source notes potential non-compliance with standard implementations [S1, unofficial].
- **When to use:** When memory bandwidth is critical and visual quality is secondary, or for specific data types like displacement maps.
- **When not to use:** When high visual fidelity is required, or when compatibility with standard GPU texture units is mandatory without custom decompression shaders.

## Background
BC5 is part of the S3 Texture Compression (S3TC) family, originally developed by S3 Graphics. It is widely supported by modern GPUs. The source is a blog post describing a custom implementation [S1, unofficial].

## Sources
[S1] BC5 圧縮テクスチャ, https://qiita.com/taqu/items/651ca51f6c46876fe9c1, fetched 2026-10-28 (unofficial)
