# VTC 3D Texture De-interleaving

Also called: NVIDIA VTC order conversion, block-compressed 3D texture slice extraction

## Overview
VTC 3D Texture De-interleaving converts NVIDIA's Volume Texture Compression (VTC) block-compressed 3D texture data from its native interleaved memory layout into a contiguous, slice-by-slice layout suitable for standard GPU volume upload. It solves the problem where DXT-compressed volumes were incorrectly treated as 2D textures (using only the first slice-sized bytes), causing sampling artifacts such as dark translucent squares with hard edges due to the interleaved block structure [S1, unofficial]. It sets itself apart from generic volume uploads by specifically handling the spatial interleaving of compressed blocks across depth layers defined in the `NV_texture_compression_vtc` specification [S1, unofficial].

## How it works
**Inputs:**
- `width`, `height`: Dimensions of the texture in texels.
- `depth`: Total number of slices (layers) in the volume.
- `block_bytes`: Size of a compressed block in bytes (typically 8 for DXT1/S3TC, 16 for DXT5/S3TC) [S1, unofficial].
- `vtc_buffer`: The source memory buffer containing the VTC-ordered data.

**Outputs:**
- `output_buffer`: A flat 1D array where each slice is stored contiguously, ready for GPU upload.

**Data Structures:**
- The VTC storage order groups slices in sets of four. Within each group, the four slices' blocks at the same spatial position are stored contiguously [S1, unofficial].
- If `width` and `height` are not powers of two, the volume is treated as slice-after-slice (no de-interleaving required) [S1, unofficial].

**Parameters:**
- `block_bytes`: Compressed block size. Typical values: 8 (DXT1/S3TC), 16 (DXT5/S3TC) [S1, unofficial].
- `blocks`: Number of compressed blocks per slice. Calculated as `(width / 4) * (height / 4)`. Note: This assumes `width` and `height` are multiples of 4, which is required for block compression [G].

**Steps:**
1. Check if `width` and `height` are powers of two. If not, copy the `vtc_buffer` directly to `output_buffer` in slice-after-slice order [S1, unofficial].
2. If they are powers of two, initialize `output_buffer` and iterate through each slice `z` from `0` to `depth - 1`.
3. For each slice `z`, determine the group it belongs to:
   - Full groups contain 4 slices.
   - The last group contains `depth % 4` slices. If `depth % 4 == 0`, there is no partial last group.
   - Let `n` be the layer count of the group containing slice `z`. For full groups, `n = 4`. For the last group, `n = depth % 4` [S1, unofficial].
4. Calculate the source offset of block `i` in slice `z` within the `vtc_buffer` using the formula:
   $$ \text{offset} = (4 \times \lfloor z / 4 \rfloor \times \text{blocks} + i \times n + z \% 4) \times \text{block\_bytes} $$
   Where:
   - `blocks` is the number of blocks per slice.
   - `i` is the block index within the slice ($0 \le i < \text{blocks}$).
   - `z` is the current slice index.
   - `n` is the group size (4 or `depth % 4`).
   - `block_bytes` is the compressed block size [S1, unofficial].
5. **Edge Case for Last Group:** If `depth % 4 >= 2`, the spec's `blockIndex` formula stores slices sequentially, which contradicts the spec's prose layout (4x4x2 or 4x4x3 blocks). Implementation must follow the prose layout for the last group rather than the formula [S1, unofficial]. This means the simple formula in Step 4 is valid only for full groups and the first slice of a partial group if it exists, but generally, the prose layout dictates the actual memory arrangement for the tail [S1, unofficial]. *Correction based on source:* The source states the formula is used for full groups, and for the last group, the prose is followed. The formula provided is explicitly linked to the spec's `blockIndex` which is valid for full groups. For the last group, the layout is interleaved as 4x4x2 or 4x4x3 blocks. The implementation must handle this deviation. The pseudocode below reflects the formula for full groups and a placeholder for the prose-based extraction for the last group, as the exact prose layout algorithm is not fully detailed in the provided text beyond "4x4x2 or 4x4x3 blocks" [S1, unofficial].
6. Copy the block from `vtc_buffer` at the calculated offset to the corresponding position in `output_buffer`.
7. Repeat for all blocks in all slices.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vtc_deinterleave(width, height, depth, block_bytes, vtc_buffer):
    # Check if width and height are powers of two
    if not (width & (width - 1) == 0) or not (height & (height - 1) == 0):
        # Not powers of two: treat as slice-after-slice
        # Copy vtc_buffer directly to output_buffer
        # Assuming vtc_buffer is already in slice-after-slice order for non-VTC
        return vtc_buffer.copy()

    # Calculate blocks per slice
    blocks_per_slice = (width // 4) * (height // 4)
    
    # Initialize output buffer
    output_buffer = bytearray(len(vtc_buffer))
    
    # Determine number of full groups and remaining slices
    num_full_groups = depth // 4
    remaining_slices = depth % 4
    
    current_output_offset = 0
    
    # Process full groups
    for g in range(num_full_groups):
        # For each slice in the group (z = g*4 to g*4 + 3)
        for z_local in range(4):
            z = g * 4 + z_local
            # n is 4 for full groups
            n = 4
            
            # Process each block in the slice
            for i in range(blocks_per_slice):
                # Formula for full groups
                # offset = (4 * floor(z / 4) * blocks + i * n + z % 4) * block_bytes
                # Since z = g*4 + z_local, floor(z/4) = g, z%4 = z_local
                source_offset = (4 * g * blocks_per_slice + i * n + z_local) * block_bytes
                
                # Copy block
                output_buffer[current_output_offset : current_output_offset + block_bytes] = \
                    vtc_buffer[source_offset : source_offset + block_bytes]
                
                current_output_offset += block_bytes

    # Process last group if it exists
    if remaining_slices > 0:
        # The spec's formula stores slices sequentially for the last group,
        # but the prose specifies interleaved 4x4x2 or 4x4x3 blocks when depth % 4 >= 2.
        # Implementation must follow the prose for the last group [S1, unofficial].
        # The exact algorithm for the prose layout is not fully detailed in the source
        # beyond "4x4x2 or 4x4x3 blocks". 
        # Placeholder for prose-based extraction logic.
        # In practice, this requires implementing the specific interleaving pattern
        # described in the NV_texture_compression_vtc spec for the tail group.
        # For the purpose of this implementation guide, we note the deviation.
        # If remaining_slices == 1, it might be sequential or part of a 4x4x1 block?
        # The source mentions "4x4x2 or 4x4x3 blocks" for depth % 4 >= 2.
        # This implies a specific interleaving. Without the full spec text, 
        # we cannot provide the exact formula for the last group's prose layout.
        # However, the source confirms the formula is WRONG for the last group if depth % 4 >= 2.
        
        # Example placeholder: If remaining_slices == 2, it forms a 4x4x2 block group.
        # The blocks are interleaved. The exact mapping requires the spec's prose.
        # We will leave this as a comment indicating the need for spec-specific logic.
        pass # Implement prose-based layout for remaining_slices here

    return output_buffer
```

**Edge Cases and Pitfalls:**
- **Non-Power-of-Two Dimensions:** If `width` or `height` is not a power of two, the volume is not in VTC order and should be treated as slice-after-slice [S1, unofficial].
- **Last Group Discrepancy:** When `depth % 4 >= 2`, the spec's `blockIndex` formula contradicts its prose layout. The formula stores slices sequentially, while the prose specifies interleaved 4x4x2 or 4x4x3 blocks. Implementations must follow the prose for the last group [S1, unofficial].
- **2D Texture Misinterpretation:** If a VTC volume is read as a 2D texture using only the first slice-sized bytes, the interleaved layout causes every fourth block to be opaque black, rendering as dark translucent squares with hard edges [S1, unofficial].
- **Metal Uploads:** For Metal uploads, `bytesPerImage` must be set to `row_bytes * texel_rows * 4` (four times the slice size) to correctly span block-compressed slices [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the total number of blocks in the volume. Each block is read once and written once.
- **Memory Complexity:** $O(N)$, requiring a buffer of the same size as the input for the de-interleaved output.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with volume size.
- **Strengths:** Correctly handles NVIDIA's VTC compressed volume data, preventing sampling artifacts.
- **Limitations:** Only applicable to block-compressed formats (DXT) with power-of-two width and height. Requires specific handling for the last group of slices.
- **When to use:** When uploading DXT-compressed 3D textures to the GPU in a VTC order.
- **When not to use:** For uncompressed volumes, or volumes with non-power-of-two dimensions (which are handled as slice-after-slice).

## Background
The method originated from the need to correctly upload block-compressed (DXT) 3D textures in the RPCS3 emulator. Previously, the 3D upload path only supported uncompressed formats, causing DXT volumes to be incorrectly treated as 2D textures, leading to visual artifacts in retail titles like explosion smoke effects [S1, unofficial]. The function `eng_unvtc_3d_slice` was developed as a port of RPCS3's `copy_unmodified_block_vtc` and later rewritten to rely solely on the `NV_texture_compression_vtc` specification [S1, unofficial]. It was merged into the codebase in version v0.18.0 [S1, unofficial].

## Sources
[S1] rsx engine: block-compressed (DXT) 3D textures go up as volumes, https://github.com/sp00nznet/ps3recomp/pull/585, fetched 2026-10-27 (unofficial)
