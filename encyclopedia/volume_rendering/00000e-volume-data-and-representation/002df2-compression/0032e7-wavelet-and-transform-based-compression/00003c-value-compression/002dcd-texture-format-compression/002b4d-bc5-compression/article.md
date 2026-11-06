# BC5 Compression


## Overview
BC5 is a block compression format for 2-channel (RG) textures, compressing 4x4 blocks (32 bytes) into 16 bytes [S1, unofficial]. It supports two variants: BC5_UNORM (outputs [0, 1] floats) and BC5_SNORM (outputs [-1, 1] floats) [S1, unofficial]. It is the 2-channel counterpart to BC4, which handles single-channel (R) data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4x4 block of 16 unsigned 8-bit integer values per channel (32 bytes total for 2 channels).
*   **Output:** A 16-byte compressed block.

**Data Structures and Layout:**
The compressed block consists of:
1.  1 byte for the minimum value (`min_val`).
2.  1 byte for the maximum value (`max_val`).
3.  16 indices, each 3 bits wide, totaling 6 bytes. These indices reference a color table derived from `min_val` and `max_val`.

The bit layout for the indices is arranged in two rows of 8 indices (little-endian bit packing within bytes, row-major order for pixels):
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
/*
MSB                   LSB
| red_0               | 1 byte minimum color
| red_1               | 1 byte maximum color
| red_h | red_g | red_f | red_e | red_d | red_c | red_b | red_a | 3 bits x 8
| red_p | red_o | red_n | red_m | red_l | red_k | red_j | red_i | 3 bits x 8
*/
```
[S1, unofficial]

**Algorithm Steps:**
1.  **Compute Min/Max:** Find the minimum (`colors[0]`) and maximum (`colors[1]`) of the 16 input values.
2.  **Scale:** Shift both `colors[0]` and `colors[1]` left by 20 bits (`Shift = 20`) to perform fixed-point arithmetic [S1, unofficial].
3.  **Generate Color Table:**
    *   If `colors[1] < colors[0]` (which implies `min == max` due to initialization logic where `colors[0]` and `colors[1]` start equal and are updated via min/max functions, but the code checks `colors[1] < colors[0]` after shift, effectively checking if max < min or if they are equal and the shift logic creates a specific state? Note: The code initializes colors[0]=block[0], colors[1]=block[0], then updates min/max. If all values are equal, min==max. If min < max, then colors[0] < colors[1]. The condition `colors[1] < colors[0]` is technically impossible if min <= max and shift is positive, unless there's an overflow or specific edge case handling not fully explained, but the code implements two branches: one for 6 interpolated values and one for 4.):
        *   Use 6 interpolated values.
        *   `colors[2] = (6 * colors[0] + 1 * colors[1]) / 7`
        *   `colors[3] = (5 * colors[0] + 2 * colors[1]) / 7`
        *   `colors[4] = (4 * colors[0] + 3 * colors[1]) / 7`
        *   `colors[5] = (3 * colors[0] + 4 * colors[1]) / 7`
        *   `colors[6] = (2 * colors[0] + 5 * colors[1]) / 7`
        *   `colors[7] = (1 * colors[0] + 6 * colors[1]) / 7`
    *   Else (`colors[1] >= colors[0]`):
        *   Use 4 interpolated values.
        *   `colors[2] = (4 * colors[0] + 1 * colors[1]) / 5`
        *   `colors[3] = (3 * colors[0] + 2 * colors[1]) / 5`
        *   `colors[4] = (2 * colors[0] + 3 * colors[1]) / 5`
        *   `colors[5] = (1 * colors[0] + 4 * colors[1]) / 5`
        *   `colors[6] = 0`
        *   `colors[7] = 255 << Shift`
    [S1, unofficial]
4.  **Encode Indices:** For each of the 16 pixels, find the index (0-7) of the table value in `colors` that is closest to the pixel's value (scaled by `Shift`). This is done by iterating through all 8 colors and tracking the minimum absolute difference.
5.  **Pack:** Pack the 16 indices into 6 bytes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bc5_compress(block: list[int]) -> bytes:
    """
    Compress a 4x4 block of 16 unsigned 8-bit integers using BC5 logic.
    block: list of 16 integers (0-255).
    Returns: 16 bytes.
    """
    # Step 1: Compute Min/Max
    min_val = block[0]
    max_val = block[0]
    for i in range(1, 16):
        if block[i] < min_val:
            min_val = block[i]
        if block[i] > max_val:
            max_val = block[i]

    # Step 2: Scale
    Shift = 20
    colors = [0] * 8
    colors[0] = min_val << Shift
    colors[1] = max_val << Shift

    # Step 3: Generate Color Table
    if colors[1] < colors[0]:
        # 6 interpolated color values
        colors[2] = (6 * colors[0] + 1 * colors[1]) // 7
        colors[3] = (5 * colors[0] + 2 * colors[1]) // 7
        colors[4] = (4 * colors[0] + 3 * colors[1]) // 7
        colors[5] = (3 * colors[0] + 4 * colors[1]) // 7
        colors[6] = (2 * colors[0] + 5 * colors[1]) // 7
        colors[7] = (1 * colors[0] + 6 * colors[1]) // 7
    else:
        # 4 interpolated color values
        colors[2] = (4 * colors[0] + 1 * colors[1]) // 5
        colors[3] = (3 * colors[0] + 2 * colors[1]) // 5
        colors[4] = (2 * colors[0] + 3 * colors[1]) // 5
        colors[5] = (1 * colors[0] + 4 * colors[1]) // 5
        colors[6] = 0
        colors[7] = 255 << Shift

    # Step 4: Encode Indices
    indices = []
    for pixel_val in block:
        x = pixel_val << Shift
        min_diff = abs(colors[0] - x)
        best_index = 0
        for i in range(1, 8):
            diff = abs(colors[i] - x)
            if diff < min_diff:
                min_diff = diff
                best_index = i
        indices.append(best_index)

    # Step 5: Pack Indices
    # 16 indices of 3 bits each = 48 bits = 6 bytes
    packed = bytearray(6)
    for i in range(16):
        byte_idx = i // 8
        bit_offset = (i % 8) * 3
        packed[byte_idx] |= (indices[i] << bit_offset)

    # Construct final block
    result = bytearray(16)
    result[0] = min_val
    result[1] = max_val
    result[2:8] = packed
    return bytes(result)
```
[S1, unofficial]

**Parameters:**
*   `Shift`: 20 [S1, unofficial]

**Edge Cases and Pitfalls:**
*   The condition `colors[1] < colors[0]` is technically impossible if `min <= max` and `Shift` is positive, unless there's an overflow or specific edge case handling not fully explained. The code implements two branches: one for 6 interpolated values and one for 4. [S1, unofficial]
*   The provided implementation notes that results differ from GPUOpen Compressonator, suggesting the current method may not be optimal or fully compliant with the standard reference implementation. [S1, unofficial]

## Operating characteristics
Not given in the sources.

## Background
BC5 is part of the S3 Texture Compression (S3TC) family, specifically designed for 2-channel textures. It is commonly used for normal maps, where the X and Y channels are stored in the R and G channels respectively. The format was introduced as part of the DirectX Texture Compression standards.

## Sources
[S1] BC5 圧縮テクスチャ, https://qiita.com/taqu/items/651ca51f6c46876fe9c1, fetched 2026-10-28 (unofficial)
