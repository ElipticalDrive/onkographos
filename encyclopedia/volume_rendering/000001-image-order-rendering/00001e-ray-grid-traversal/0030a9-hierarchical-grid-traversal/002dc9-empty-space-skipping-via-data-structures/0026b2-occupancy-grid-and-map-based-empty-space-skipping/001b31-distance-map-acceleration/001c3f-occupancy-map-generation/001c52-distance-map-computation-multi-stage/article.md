# Distance Map Computation (Multi-Stage)

Also called: Compute Distance Map, Distance Field Compute, Multi-Stage Distance Map Computation, Occupancy Distance Map

## Overview
A GPU compute shader dispatch routine that computes a distance map from an occupancy map, using a swap buffer for iterative or multi-pass distance calculation [S1, unofficial]. It enables empty space skipping during volume ray marching by providing the distance to the nearest occupied voxel for any given position [S1, unofficial]. Unlike [Occupancy Mipmap Generation](2700-occupancy-mipmap-generation.md) which creates hierarchical mipmaps, this method produces a single-resolution distance field via a specific three-stage dispatch sequence [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: `textureOccupancyMap` (GL_R8UI, GL_READ_WRITE).
*   Output: `textureDistanceMap` (GL_R8UI, GL_READ_WRITE).
*   Temporary: `textureSwap` (GL_R8UI, GL_READ_WRITE).
*   Uniform: UBO (4 bytes) storing the current stage index.

**Data Structures:**
*   `textureOccupancyMap`: 3D texture, format `GL_R8UI`.
*   `textureDistanceMap`: 3D texture, format `GL_R8UI`.
*   `textureSwap`: 3D texture, format `GL_R8UI`.
*   UBO: 4-byte integer, binding point 0.

**Steps:**
1.  Bind `textureDistanceMap` to image unit 0 and `textureOccupancyMap` to image unit 1, both with `GL_R8UI` format and `GL_READ_WRITE` access [S1, unofficial].
2.  Create a Uniform Buffer Object (UBO) of size 4 bytes, bind it to binding point 0, and use `glBindBufferRange` to map the entire buffer [S1, unofficial].
3.  Calculate the number of work groups for the 3D grid based on `sizeOccupancyMap` dimensions divided by 8, using ceiling division [S1, unofficial].
4.  **Stage 1:** Set UBO stage value to 0, dispatch compute with dimensions `(numWorkGroups[1], numWorkGroups[2], 1)`, and issue a memory barrier [S1, unofficial].
5.  **Stage 2:** Bind `textureSwap` to image unit 1, set UBO stage value to 1, dispatch compute with dimensions `(numWorkGroups[0], numWorkGroups[2], 1)`, and issue a memory barrier [S1, unofficial].
6.  **Stage 3:** Set UBO stage value to 2, dispatch compute with dimensions `(numWorkGroups[0], numWorkGroups[1], 1)`, and issue a memory barrier [S1, unofficial].
7.  Clean up by unbinding the UBO and deleting the buffer object [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the dispatch sequence
# Note: The actual compute shader kernel logic for distance propagation is not provided in the source [S1, unofficial].
# The following represents the host-side dispatch logic described.

def compute_distance_map_multi_stage(occupancy_map, distance_map, swap_buffer, volume_size):
    # 1. Bind textures
    bind_image_unit(0, distance_map, format="GL_R8UI", access="GL_READ_WRITE")
    bind_image_unit(1, occupancy_map, format="GL_R8UI", access="GL_READ_WRITE")
    
    # 2. Create UBO
    ubo = create_ubo(size=4)
    bind_ubo(binding=0, buffer=ubo)
    bind_buffer_range(binding=0, buffer=ubo, offset=0, size=4)
    
    # 3. Calculate work groups
    # Assuming occupancy map dimensions are known
    dim = texture_size(occupancy_map)
    block_size = 8
    num_work_groups_x = ceil(dim.x / block_size)
    num_work_groups_y = ceil(dim.y / block_size)
    num_work_groups_z = ceil(dim.z / block_size)
    
    # 4. Stage 1
    set_ubo_value(ubo, 0)
    dispatch_compute(num_work_groups_y, num_work_groups_z, 1)
    memory_barrier()
    
    # 5. Stage 2
    bind_image_unit(1, swap_buffer, format="GL_R8UI", access="GL_READ_WRITE")
    set_ubo_value(ubo, 1)
    dispatch_compute(num_work_groups_x, num_work_groups_z, 1)
    memory_barrier()
    
    # 6. Stage 3
    set_ubo_value(ubo, 2)
    dispatch_compute(num_work_groups_x, num_work_groups_y, 1)
    memory_barrier()
    
    # 7. Cleanup
    unbind_ubo(binding=0)
    delete_buffer(ubo)
```

**Parameters:**
*   Block size: 8 (implied by division by 8 in work group calculation) [S1, unofficial].
*   UBO size: 4 bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The source notes that workgroup calculation and dispatch details are truncated [S1, unofficial].
*   Buffer allocation and data population for the UBO are initiated but truncated [S1, unofficial].
*   Precision issues may arise if the distance map format (`GL_R8UI`) cannot represent the maximum distance in the volume [G].

## Subtopics

### [Empty Space Map Generation](1771-empty-space-map-generation.md)
Empty Space Map Generation is a preprocessing optimization for volume ray marching that creates a boolean map indicating which voxels contain no visible geometry based on a transfer function threshold. It allows the renderer to skip entire empty regions during ray traversal, significantly reducing the number of density samples and transfer function evaluations. The method integrates the transfer function's density criteria directly into the skipping logic, ensuring that only voxels contributing to the final image are processed.

### Distance Field Precomputation
Precomputing distance fields for the scalar fields involved in the fiber surface definition to facilitate fast intersection detection via the rule of signs [S2]. The time is fixed and depends only on the distance field dimensions [S2]. A 64x64 array size is used [S2]. Precomputation time is 1-2 milliseconds for 4 FSCP with 4 segments [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Enables efficient empty space skipping [S1, unofficial].
*   **Limitations:** Requires multiple dispatches and memory barriers [S1, unofficial].
*   **When to use:** When empty space skipping is required for volume rendering [S1, unofficial].
*   **When not to use:** When memory bandwidth is a critical bottleneck due to multiple passes [G].

## Background
Origin: [S1, unofficial] describes the method as a GPU compute shader dispatch routine. [S2] discusses distance field precomputation for fiber surface ray casting, noting that distance field computation is on the order of milliseconds and not a bottleneck [S2].

## Sources
[S1] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
[S2] Direct Multifield Volume Ray Casting of Fiber Surfaces, https://doi.org/10.1109/tvcg.2016.2599040, fetched 2026-10-26
