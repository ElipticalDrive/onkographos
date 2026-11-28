# Entry Point Buffer with Back-to-Front Projection

Also called: Back-to-Front Projection for Entry Points, Block Projection Entry Point Method

## Overview
This method determines the first intersection of each viewing ray with the visible volume to skip transparent data during raycasting [S1, unofficial]. It establishes an entry point buffer storing depth values and block indices, populated by projecting non-transparent blocks back-to-front [S1, unofficial]. It sets itself apart from standard bounding box raymarching by using block-level visibility culling and hidden surface removal to initialize rays precisely at the volume surface rather than the bounding box [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volume dataset organized into blocks, transfer function, viewing transformation (parallel projection assumed for simplification) [S1, unofficial].
- Output: Entry point buffer containing depth values and block indices for each pixel [S1, unofficial].

**Data Structures:**
1. **Block Metadata:** Stores minimum and maximum sample values for each block [S1, unofficial].
2. **Summed Area Table (SAT):** Precomputed table of the opacity transfer function to quickly determine block visibility [S1, unofficial].
3. **Entry Point Buffer:** Array sized to the image plane [S1, unofficial]. Each entry stores:
   - `relative_depth`: Z-value of the generic template [S1, unofficial].
   - `block_index`: Unique identifier for the block [S1, unofficial].

**Steps:**
1. **Block Visibility Culling:** Identify blocks containing only transparent data [S1, unofficial]. A block is considered transparent if its min/max sample values, when evaluated against the opacity transfer function and summed area table, yield zero opacity [S1, unofficial].
2. **Template Generation:** For parallel projection, generate one geometric template representing the block's projection under the current viewing transformation [S1, unofficial].
3. **Back-to-Front Projection:**
   - Traverse the list of non-transparent blocks in reverse order (back-to-front) [S1, unofficial].
   - For each block, translate the template by vector $\mathbf{t} = (t_x, t_y, t_z)$, where $t_x, t_y$ are the block's position on the image plane and $t_z$ is the depth offset [S1, unofficial].
   - Write the Z-value of the generic template into the entry point buffer for covered pixels, along with the block's unique index [S1, unofficial].
   - Use a Z-buffer algorithm to ensure correct visibility (only the closest entry point is stored) [S1, unofficial].
4. **Ray Initialization:** In the ray setup phase, use the depth values and block indices from the entry point buffer to calculate the absolute ray start position [S1, unofficial]. The absolute depth is found by adding the stored relative depth to the block's $t_z$ value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_entry_points(blocks, transfer_function, sat, image_width, image_height):
    # 1. Cull transparent blocks using min/max values and SAT
    visible_blocks = []
    for block in blocks:
        if not is_block_transparent(block.min_val, block.max_val, transfer_function, sat):
            visible_blocks.append(block)

    # 2. Generate template for parallel projection
    # Rasterize the block's projected quad under viewing transformation [G]
    template = generate_block_template(blocks[0])

    # 3. Initialize entry point buffer
    entry_point_buffer = [None] * (image_width * image_height)

    # 4. Back-to-front projection
    for block in reversed(visible_blocks):
        tx, ty = block.position_on_image_plane
        tz = block.depth_offset

        for pixel in template.pixels:
            screen_x = pixel.x + tx
            screen_y = pixel.y + ty

            if 0 <= screen_x < image_width and 0 <= screen_y < image_height:
                idx = screen_y * image_width + screen_x
                new_depth = pixel.z

                # Z-buffer logic: keep the closest (smallest) depth value
                if entry_point_buffer[idx] is None or new_depth < entry_point_buffer[idx]['relative_depth']:
                    entry_point_buffer[idx] = {
                        'relative_depth': new_depth,
                        'block_index': block.id
                    }

    return entry_point_buffer

def is_block_transparent(min_val, max_val, transfer_function, sat):
    opacity = compute_block_opacity(min_val, max_val, transfer_function, sat)
    return opacity == 0

def compute_block_opacity(min_val, max_val, transfer_function, sat):
    # Standard SAT lookup: sum opacity values for the range [min_val, max_val]
    return sat[max_val] - sat[min_val - 1] if min_val > 0 else sat[max_val]

def generate_block_template(block):
    # Rasterize block under viewing transformation to get relative Z-values per pixel [G]
    pixels = []
    for y in range(block.height):
        for x in range(block.width):
            pixels.append({'x': x, 'y': y, 'z': block.base_depth + (x + y) * block.depth_step})
    return pixels
```

**Parameters:**
- **Summed Area Table:** Used to quickly determine block visibility based on min/max sample values and opacity transfer function [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Performance Overhead:** Using addition and depth test at every pixel of the template for each block is expensive [S1, unofficial].
- **Mitigation:** The back-to-front projection and storing indices/relative depths reduces the number of operations in the ray setup phase, as additions are only performed for rays that actually hit a block [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of visible blocks and their projected area [S1, unofficial]. The projection step involves iterating over template pixels for each block [S1, unofficial].
- **Memory Complexity:** Requires storage for the entry point buffer (depth + index per pixel) and block metadata [S1, unofficial].
- **Strengths:** Effectively skips transparent data, allowing interactive transfer function modification due to fast block-level visibility checks [S1, unofficial].
- **Limitations:** The projection step can be expensive due to per-pixel additions and depth tests [S1, unofficial].
- **When to Use:** Suitable for large medical datasets with significant transparent regions where interactive exploration is required [S1, unofficial].

## Background
Developed for efficient volume visualization of large medical datasets [S1, unofficial]. The method leverages block-based processing and GPU-friendly rasterization techniques to optimize raycasting initialization [S1, unofficial].

## Sources
[S1] Efficient Volume Visualization of Large Medical Datasets, https://old.cescg.org/CESCG-2004/web/Bruckner-Stefan/html/, fetched 2026-10-30 (unofficial)
