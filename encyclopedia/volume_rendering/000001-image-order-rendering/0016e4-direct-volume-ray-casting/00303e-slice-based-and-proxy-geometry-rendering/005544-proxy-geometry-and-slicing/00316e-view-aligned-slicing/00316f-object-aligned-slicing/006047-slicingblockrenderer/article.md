# SlicingBlockRenderer

Also called: Slice-based direct volume renderer with subvolume blocking

## Overview
SlicingBlockRenderer is a slice-based direct volume rendering method that organizes volume data into subvolumes (blocks) and renders them as textured quads (slices) sorted by distance from the camera to handle transparency correctly [S1, unofficial]. It builds on [Object-Aligned Slicing](6531-object-aligned-slicing.md) by introducing explicit blocking to improve memory locality and culling efficiency, distinguishing itself from non-blocking slice renderers by managing discrete GPU resources per block [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volume data partitioned into blocks (subvolumes), camera position and view direction, transfer function parameters.
- Output: Rendered image with correct alpha blending.

**Data Structures:**
- **Block:** Represents a subvolume. Contains a status flag `GPU_RES` indicating if texture data is uploaded to the GPU.
- **Vertex Buffer:** A single buffer containing geometry for all possible slice orientations.
  - Layout: 6 sets of quads corresponding to 3 axes (YZ, XZ, XY) and 2 viewing directions (positive/negative) per axis.
  - Each set contains `numSlices` slices.
  - Each slice is a quad (4 vertices).
  - Total vertices = $6 \times \text{numSlices} \times 4$.
  - Indices are structured to draw these quads sequentially.
- **Sampler:** Configured with linear min/mag filtering and clamp-to-border wrapping for S, T, and R axes [S1, unofficial]. Bound to `BLOCK_TEXTURE_UNIT`.

**Parameters:**
- `m_tfuncScaleValue`: Scale factor for the transfer function. Default: $1.0$ [S1, unofficial]. Passed to shader uniform `VOLUME_TRANSF_SCALER_UNIFORM_STR`.
- `m_shouldUseLighting`: Boolean flag to toggle lighting. Default: `false` [S1, unofficial].
- `m_drawNonEmptySlices`: Boolean flag to toggle slice rendering. Default: `true` [S1, unofficial].
- `m_drawNonEmptyBoundingBoxes`: Boolean flag to toggle bounding box rendering. Default: `false` [S1, unofficial].

**Numbered Steps:**
1. **Sampler Setup:** Create a sampler object with linear min/mag filtering and clamp-to-border wrapping for S, T, and R axes. Bind it to `BLOCK_TEXTURE_UNIT` [S1, unofficial].
2. **Block Sorting:** In the draw loop, identify non-empty blocks. Sort these blocks by distance from the camera eye in descending order (furthest first) to satisfy the painters algorithm for transparency [S1, unofficial].
3. **View Axis Determination:** Compute the base vertex and element offset for the vertex buffer based on the view direction.
   - Determine the axis with the largest absolute component in the view vector to select the slice set (YZ, XZ, or XY).
   - Determine if viewing in the positive or negative direction to select the correct vertex range within that set [S1, unofficial].
4. **Rendering Loop:** For each non-empty block in the sorted list:
   - Check if the block's status includes `GPU_RES`. If not, skip the block [S1, unofficial].
   - Set the world matrix for the block.
   - Bind the block's texture to `BLOCK_TEXTURE_UNIT`.
   - Set the MVP matrix uniform.
   - Draw the slices using `glDrawElementsBaseVertex` with the computed base vertex and element offset [S1, unofficial].

**Formulas:**
Let $N$ be the number of slices along the viewing axis (`numSlicesPerBlock`). Let $V_{quad} = 4$ be the vertices per quad. The base vertex index for the vertex buffer is calculated as follows:

- **View along X (YZ slices):**
  - Positive X: $BaseVertex = 0$
  - Negative X: $BaseVertex = 1 \times V_{quad} \times N$

- **View along Y (XZ slices):**
  - Positive Y: $BaseVertex = 2 \times V_{quad} \times N$
  - Negative Y: $BaseVertex = 3 \times V_{quad} \times N$

- **View along Z (XY slices):**
  - Positive Z: $BaseVertex = 4 \times V_{quad} \times N$
  - Negative Z: $BaseVertex = 5 \times V_{quad} \times N$

Where $N$ is `numSlicesPerBlock.x` for X-view, `.y` for Y-view, and `.z` for Z-view [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slicing_block_renderer(blocks, camera, view_vector, vertex_buffer, index_buffer, sampler, shader):
    # 1. Setup Sampler (Assumed done once during init, but conceptually part of state)
    # sampler.min_filter = LINEAR
    # sampler.mag_filter = LINEAR
    # sampler.wrap_s = CLAMP_TO_BORDER
    # sampler.wrap_t = CLAMP_TO_BORDER
    # sampler.wrap_r = CLAMP_TO_BORDER
    # bind_sampler(sampler, BLOCK_TEXTURE_UNIT)

    # 2. Sort Blocks
    # Calculate distance from camera eye to block origin
    sorted_blocks = sorted(blocks, key=lambda b: distance(camera.eye, b.origin), reverse=True)

    # 3. Determine View Axis and Base Vertex
    abs_x = abs(view_vector.x)
    abs_y = abs(view_vector.y)
    abs_z = abs(view_vector.z)

    # Determine dominant axis
    if abs_x >= abs_y and abs_x >= abs_z:
        axis = 'X'
        is_pos = view_vector.x > 0
    elif abs_y >= abs_x and abs_y >= abs_z:
        axis = 'Y'
        is_pos = view_vector.y > 0
    else:
        axis = 'Z'
        is_pos = view_vector.z > 0

    # Get number of slices along the viewing axis for a block
    # Note: numSlicesPerBlock might vary per block or be uniform; assuming uniform for base calculation logic in source
    # If uniform, use global numSlicesPerBlock. If not, this base vertex logic applies per block's specific N.
    # The source implies a single vertex buffer layout for all blocks, so N is likely fixed per block type or global.
    # We assume N is available as numSlicesPerBlock[axis]
    N = numSlicesPerBlock[axis]
    V_quad = 4

    if axis == 'X':
        base_vertex = 0 if is_pos else 1 * V_quad * N
    elif axis == 'Y':
        base_vertex = 2 * V_quad * N if is_pos else 3 * V_quad * N
    else: # axis == 'Z'
        base_vertex = 4 * V_quad * N if is_pos else 5 * V_quad * N

    # 4. Render Loop
    for block in sorted_blocks:
        # Check GPU resource status
        if not (block.status & GPU_RES):
            continue

        # Set World Matrix
        shader.set_uniform('WORLD_MATRIX', block.world_matrix)

        # Bind Texture
        bind_texture(block.texture, BLOCK_TEXTURE_UNIT)

        # Set MVP Matrix
        shader.set_uniform('MVP_MATRIX', camera.mvp_matrix * block.world_matrix)

        # Set Transfer Function Scale
        shader.set_uniform('VOLUME_TRANSF_SCALER_UNIFORM_STR', m_tfuncScaleValue)

        # Draw Slices
        # glDrawElementsBaseVertex uses index_buffer, count, type, base_vertex
        draw_elements_base_vertex(index_buffer, count=total_indices_for_slices, type=UNSIGNED_INT, base_vertex=base_vertex)
```

**Edge Cases and Pitfalls:**
- **Transparency Sorting:** Blocks must be sorted from furthest to nearest relative to the camera to ensure correct blending order. Failure to sort results in incorrect transparency artifacts [S1, unofficial].
- **Resource Availability:** Slices are only drawn if the block's status includes `GPU_RES`. This ensures texture data is uploaded before rendering. Skipping this check may cause rendering errors or crashes if texture is not bound [S1, unofficial].
- **Precision:** The base vertex calculation relies on integer arithmetic. Ensure `numSlices` and vertex counts fit within the index buffer type limits.

## Operating characteristics
- **Time Complexity:** Sorting blocks takes $O(B \log B)$ where $B$ is the number of blocks. Rendering is proportional to the number of non-empty blocks and slices.
- **Memory Complexity:** Requires storing vertex/index buffers for all slice orientations and texture data for each block. The vertex buffer size is fixed based on the maximum number of slices per block.
- **Performance:** Performance depends on the number of blocks, texture upload bandwidth, and sorting overhead. Efficient culling of empty blocks improves performance.
- **Strengths:** Good memory locality due to blocking. Efficient culling of empty blocks. Simple geometry (quads) allows for fast rasterization.
- **Limitations:** Sorting overhead for large numbers of blocks. Texture memory usage scales with the number of blocks.
- **When to Use:** Suitable for large volume datasets where blocking improves cache performance and culling is beneficial.
- **When Not to Use:** When the number of blocks is small, the sorting overhead may not be justified. When real-time performance is critical and sorting is a bottleneck.

## Background
The SlicingBlockRenderer is an implementation of slice-based direct volume rendering with subvolume blocking. It is associated with the work of Jim Pelton [S1, unofficial]. The method is part of a broader family of volume rendering techniques that use proxy geometry (slices) to approximate the volume.

## Sources
[S1] jimpelton/subvol, https://github.com/jimpelton/subvol, fetched 2026-10-26 (unofficial)
