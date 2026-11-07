# BC5 Compression

## Overview
BC5 is a block compression format for 2-channel (RG) textures, compressing 4x4 blocks (32 bytes) into 16 bytes [S1, unofficial]. It supports two variants: BC5_UNORM (outputs [0, 1] floats) and BC5_SNORM (outputs [-1, 1] floats) [S1, unofficial]. It is the 2-channel extension of BC4, which handles single-channel (R) data [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 4x4 block of 16 pixel values for each of the two channels (R and G). The values are typically 8-bit unsigned integers (0-255).
- Output: A 16-byte compressed block.

**Data Structures:**
The compressed block layout is fixed at 16 bytes:
1. 1 byte: Minimum value for the channel.
2. 1 byte: Maximum value for the channel.
3. 14 bytes: 16 indices of 3 bits each, arranged in two rows of 8 indices. The indices reference a color table derived from the min/max values.

The bit layout for the indices is as follows (MSB to LSB):
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```
| red_h | red_g | red_f | red_e | red_d | red_c | red_b | red_a | 3 bits x 8
| red_p | red_o | red_n | red_m | red_l | red_k | red_j | red_i | 3 bits x 8
```
[S1, unofficial]

**Algorithm Steps:**
The compression is performed independently for each channel (R and G). The following steps describe the process for one channel:

1.  **Find Min and Max:**
    Iterate through the 16 input values to find the minimum (`min_val`) and maximum (`max_val`) values.
    [S1, unofficial]

2.  **Scale Min and Max:**
    Scale `min_val` and `max_val` by shifting them left by 20 bits to use fixed-point arithmetic.
    `colors[0] = min_val << 20`
    `colors[1] = max_val << 20`
    [S1, unofficial]

3.  **Determine Color Table Mode:**
    Check if `colors[1] < colors[0]`. This condition is technically impossible if `min_val <= max_val` and no overflow occurs, but the source code implements this check to determine the mode. In practice, if `min_val == max_val`, the standard BC5 specification usually defaults to a specific behavior (often 4-value mode with specific endpoints or a special case). The provided source code uses this condition to branch:
    - If `colors[1] < colors[0]` (interpreted as a special case, possibly for equal min/max or error handling in the source): Use 6-value interpolation.
    - Else (`colors[1] >= colors[0]`): Use 4-value interpolation.
    [S1, unofficial]

4.  **Compute Color Table:**
    Based on the mode, compute the 8 color values (`colors[0]` to `colors[7]`).

    **6-Value Mode:**
    ```
    colors[2] = (6 * colors[0] + 1 * colors[1]) / 7
    colors[3] = (5 * colors[0] + 2 * colors[1]) / 7
    colors[4] = (4 * colors[0] + 3 * colors[1]) / 7
    colors[5] = (3 * colors[0] + 4 * colors[1]) / 7
    colors[6] = (2 * colors[0] + 5 * colors[1]) / 7
    colors[7] = (1 * colors[0] + 6 * colors[1]) / 7
    ```
    [S1, unofficial]

    **4-Value Mode:**
    ```
    colors[2] = (4 * colors[0] + 1 * colors[1]) / 5
    colors[3] = (3 * colors[0] + 2 * colors[1]) / 5
    colors[4] = (2 * colors[0] + 3 * colors[1]) / 5
    colors[5] = (1 * colors[0] + 4 * colors[1]) / 5
    colors[6] = 0
    colors[7] = 255 << 20
    ```
    [S1, unofficial]

5.  **Encode Indices:**
    For each of the 16 pixels in the block, find the index `i` (0-7) in the `colors` array that minimizes the absolute difference between `colors[i]` and the original pixel value (scaled to the same fixed-point range if necessary, or comparing directly if the original value is scaled).
    [S1, unofficial]

6.  **Pack Indices:**
    Pack the 16 3-bit indices into the 14-byte index region of the 16-byte block.
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_bc5_channel(block: list[int]) -> tuple[int, int, list[int]]:
    """
    Compress a single channel of a 4x4 BC5 block.
    block: list of 16 integers (0-255)
    Returns: (min_val, max_val, indices)
    """
    # Step 1: Find min and max
    min_val = min(block)
    max_val = max(block)
    
    # Step 2: Scale
    Shift = 20
    colors = [0] * 8
    colors[0] = min_val << Shift
    colors[1] = max_val << Shift
    
    # Step 3 & 4: Determine mode and compute table
    # Note: The condition colors[1] < colors[0] is logically false for valid min/max.
    # The source code uses this branch for a specific case, possibly equal values.
    # Standard BC5 spec usually uses 4-value mode for min==max.
    # We follow the source code's logic strictly.
    if colors[1] < colors[0]:
        # 6-value mode
        colors[2] = (6 * colors[0] + 1 * colors[1]) // 7
        colors[3] = (5 * colors[0] + 2 * colors[1]) // 7
        colors[4] = (4 * colors[0] + 3 * colors[1]) // 7
        colors[5] = (3 * colors[0] + 4 * colors[1]) // 7
        colors[6] = (2 * colors[0] + 5 * colors[1]) // 7
        colors[7] = (1 * colors[0] + 6 * colors[1]) // 7
    else:
        # 4-value mode
        colors[2] = (4 * colors[0] + 1 * colors[1]) // 5
        colors[3] = (3 * colors[0] + 2 * colors[1]) // 5
        colors[4] = (2 * colors[0] + 3 * colors[1]) // 5
        colors[5] = (1 * colors[0] + 4 * colors[1]) // 5
        colors[6] = 0
        colors[7] = 255 << Shift
        
    # Step 5: Encode indices
    indices = []
    for pixel_val in block:
        # Scale pixel value to fixed point for comparison
        scaled_pixel = pixel_val << Shift
        min_diff = abs(colors[0] - scaled_pixel)
        best_index = 0
        for i in range(1, 8):
            diff = abs(colors[i] - scaled_pixel)
            if diff < min_diff:
                min_diff = diff
                best_index = i
        indices.append(best_index)
        
    return min_val, max_val, indices

def compress_bc5_block(r_block: list[int], g_block: list[int]) -> bytes:
    """
    Compress a 4x4 BC5 block.
    r_block: list of 16 integers for R channel
    g_block: list of 16 integers for G channel
    Returns: 16 bytes
    """
    # Compress R channel
    r_min, r_max, r_indices = compress_bc5_channel(r_block)
    
    # Compress G channel
    g_min, g_max, g_indices = compress_bc5_channel(g_block)
    
    # Pack the block
    # 1 byte min, 1 byte max, then 16 3-bit indices
    # The indices are packed in row-major order, 8 per row
    # Row 0: indices 0-7, Row 1: indices 8-15
    
    # Helper to pack 16 3-bit indices into 6 bytes (48 bits)
    def pack_indices(indices: list[int]) -> bytes:
        # 16 indices * 3 bits = 48 bits = 6 bytes
        packed = 0
        for i, idx in enumerate(indices):
            packed |= (idx << (3 * i))
        # Convert to 6 bytes, little-endian or big-endian? 
        # The source shows MSB to LSB layout, but standard byte packing is usually LSB first in memory for little-endian systems.
        # However, the bit layout description suggests a specific bit order within bytes.
        # Let's assume standard little-endian byte packing for the 6-byte integer.
        return packed.to_bytes(6, byteorder='little')

    r_indices_bytes = pack_indices(r_indices)
    g_indices_bytes = pack_indices(g_indices)
    
    # Construct the 16-byte block
    # [min_r, max_r, 6 bytes R indices, min_g, max_g, 6 bytes G indices]
    # Wait, the source says: "1 byte minimum value, 1 byte maximum value, followed by 16 indices of 3 bits each"
    # It does not explicitly state the interleaving of R and G channels in the 16-byte block.
    # Standard BC5 layout is:
    # Byte 0: R min
    # Byte 1: R max
    # Bytes 2-7: R indices (16 3-bit values)
    # Byte 8: G min
    # Byte 9: G max
    # Bytes 10-15: G indices (16 3-bit values)
    
    block = bytearray(16)
    block[0] = r_min
    block[1] = r_max
    block[2:8] = r_indices_bytes
    block[8] = g_min
    block[9] = g_max
    block[10:16] = g_indices_bytes
    
    return bytes(block)
```
[G] The standard BC5 layout interleaves the R and G channel data as described in the pseudocode above, which is consistent with common implementations, although the source [S1, unofficial] only explicitly details the per-channel structure and the overall 16-byte size. The bit packing of the 3-bit indices into bytes is assumed to be standard little-endian bit packing.

**Parameters:**
- `Shift`: 20 (constant integer value used for fixed-point scaling) [S1, unofficial]

**Edge Cases and Pitfalls:**
- If `min_val == max_val`, the condition `colors[1] < colors[0]` is false, so 4-value mode is used. In 4-value mode, `colors[6]` and `colors[7]` are set to 0 and `255 << 20` respectively, which may not be optimal for constant blocks. The source notes that results differ from GPUOpen Compressonator, suggesting this method may not be optimal or fully compliant with the standard reference implementation [S1, unofficial].
- Precision issues may arise from the fixed-point arithmetic and integer division.
- The condition `colors[1] < colors[0]` is logically impossible for valid `min_val <= max_val` without overflow, so its intended use case is unclear from the source code alone [S1, unofficial].

## Operating characteristics
- **Compression Ratio:** 2:1 (32 bytes to 16 bytes) [S1, unofficial].
- **Time Complexity:** O(1) per 4x4 block, as the block size is fixed.
- **Memory Complexity:** O(1) per 4x4 block.
- **Strengths:** Simple to implement, good for 2-channel data like normal maps (where R and G are tangent space coordinates) or displacement maps.
- **Limitations:** The provided implementation may not be optimal or fully compliant with standard reference implementations, as noted by the discrepancy with GPUOpen Compressonator [S1, unofficial].
- **When to use:** When storing 2-channel textures with high compression requirements and the data fits the BC5 format's assumptions.
- **When not to use:** When high fidelity is required and the simple interpolation modes are insufficient, or when compatibility with standard GPU texture compression is critical and the implementation deviates from the standard.

## Background
BC5 is part of the S3 Texture Compression (S3TC) family, originally developed by S3 Graphics. It is widely supported by modern GPUs. The source [S1, unofficial] is an unofficial implementation guide from 2021.

## Sources
[S1] BC5 圧縮テクスチャ, https://qiita.com/taqu/items/651ca51f6c46876fe9c1, fetched 2026-10-28 (unofficial)
