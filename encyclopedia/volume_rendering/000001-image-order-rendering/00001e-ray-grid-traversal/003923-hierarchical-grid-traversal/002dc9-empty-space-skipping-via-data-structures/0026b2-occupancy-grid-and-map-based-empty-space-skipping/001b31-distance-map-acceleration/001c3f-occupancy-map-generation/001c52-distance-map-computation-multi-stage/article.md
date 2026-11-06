# Distance Map Computation (Multi-Stage)

Also called: Compute Distance Map, Distance Field Compute, Multi-Stage Distance Map Computation, Occupancy Distance Map

## Overview
A GPU compute shader dispatch routine that computes a distance map from an occupancy map, using a swap buffer for iterative or multi-pass distance calculation [S1, unofficial]. It enables empty space skipping during volume ray marching by providing the distance to the nearest occupied voxel for any given position [S1, unofficial]. Unlike [Occupancy Mipmap Generation](2700-occupancy-mipmap-generation.md) which creates hierarchical mipmaps, this method produces a single-resolution distance field via a specific three-stage dispatch sequence [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: `textureOccupancyMap` (3D texture, `GL_R8UI`, `GL_READ_WRITE`) [S1, unofficial].
*   Output: `textureDistanceMap` (3D texture, `GL_R8UI`, `GL_READ_WRITE`) [S1, unofficial].
*   Temporary: `textureSwap` (3D texture, `GL_R8UI`, `GL_READ_WRITE`) [S1, unofficial].
*   Uniform: UBO (4 bytes, binding point 0) storing the current stage index [S1, unofficial].

**Data Structures:**
*   `textureOccupancyMap`: 3D texture, format `GL_R8UI` [S1, unofficial].
*   `textureDistanceMap`: 3D texture, format `GL_R8UI` [S1, unofficial].
*   `textureSwap`: 3D texture, format `GL_R8UI` [S1, unofficial].
*   UBO: 4-byte integer buffer, binding point 0 [S1, unofficial].

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
import math

def compute_distance_map_multi_stage(occupancy_map_dims):
    # 1. Bind textures (simulated host-side setup)
    # image_unit_0 = distance_map (GL_R8UI, READ_WRITE)
    # image_unit_1 = occupancy_map (GL_R8UI, READ_WRITE)

    # 2. Create UBO (4 bytes)
    ubo = bytearray(4)

    # 3. Calculate work groups (block size = 8)
    block_size = 8
    num_work_groups = [
        math.ceil(occupancy_map_dims[0] / block_size),
        math.ceil(occupancy_map_dims[1] / block_size),
        math.ceil(occupancy_map_dims[2] / block_size)
    ]

    # 4. Stage 1: UBO stage = 0, dispatch (wg[1], wg[2], 1)
    ubo[0] = 0
    # dispatch_compute(num_work_groups[1], num_work_groups[2], 1)
    # memory_barrier()

    # 5. Stage 2: Bind swap to image unit 1, UBO stage = 1, dispatch (wg[0], wg[2], 1)
    # image_unit_1 = swap_buffer
    ubo[0] = 1
    # dispatch_compute(num_work_groups[0], num_work_groups[2], 1)
    # memory_barrier()

    # 6. Stage 3: UBO stage = 2, dispatch (wg[0], wg[1], 1)
    ubo[0] = 2
    # dispatch_compute(num_work_groups[0], num_work_groups[1], 1)
    # memory_barrier()

    # 7. Cleanup
    # unbind_ubo(binding=0)
    # delete_buffer(ubo)
```

**Parameters:**
*   Block size: 8 (implied by division by 8 in work group calculation) [S1, unofficial].
*   UBO size: 4 bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The source notes that workgroup calculation and dispatch details are truncated [S1, unofficial].
*   Buffer allocation and data population for the UBO are initiated but truncated [S1, unofficial].
*   Precision issues may arise if the distance map format (`GL_R8UI`) cannot represent the maximum distance in the volume, as `GL_R8UI` saturates at 255 [G].

## Subtopics

### [Empty Space Map Generation](1771-empty-space-map-generation.md)
Empty Space Map Generation is a preprocessing optimization for volume ray marching that creates a boolean map indicating which voxels contain no visible geometry based on a transfer function threshold. It allows the renderer to skip entire empty regions during ray traversal, significantly reducing the number of density samples and transfer function evaluations. The method integrates the transfer function's density criteria directly into the skipping logic, ensuring that only voxels contributing to the final image are processed.

### Distance Field Precomputation
Precomputing distance fields for the scalar fields involved in the fiber surface definition to facilitate fast intersection detection via the rule of signs [S2]. The computation time is fixed and depends only on the distance field dimensions [S2]. A 64x64 array size is used [S2]. Precomputation time is 1-2 milliseconds for 4 FSCP with 4 segments [S2].

### AADF Exit Plane Calculation
Calculates the exit plane of an empty region defined by an Axis-Aligned Distance Field (AADF), which stores distances to the nearest occupied voxel in each of the 6 directions [S3, unofficial]. The shift for each axis is determined by the ray direction: if `ray_dir.x < 0`, shift 0 (-X) is used, else shift 5 (+X) is used; similarly for Y (10/15) and Z (20/25) [S3, unofficial]. The bounds (distances) for each axis are extracted from the packed uint, with each distance occupying 5 bits [S3, unofficial]. The lower and upper bounds of the empty region in voxel coordinates are calculated as `coord - bounds` (clamped to 0) and `coord + 1 + bounds` (clamped to `grid_ext`) [S3, unofficial]. The exit plane coordinate is returned by checking the ray direction: if negative, the lower bound is returned, otherwise the upper bound is returned [S3, unofficial]. The `cell_ext` parameter defines the cell size in voxel units: `VX_BRICK_EXT * VX_CHUNK_EXT` for chunks, `VX_BRICK_EXT` for bricks, and 1 for voxels [S3, unofficial].

### [Distance Transformation (Proximity Clouds)](5720-distance-transformation-proximity-clouds.md)
A technique for fast 3D grid traversal using a 3D distance map generated from binary volumes, where each cell contains the distance to the nearest non-empty cell.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Measured Performance:** Not given in the sources. [S1, unofficial]
*   **Scaling:** Not given in the sources. [S1, unofficial]
*   **Strengths:** Enables efficient empty space skipping [S1, unofficial].
*   **Limitations:** Requires multiple dispatches and memory barriers [S1, unofficial].
*   **When to use:** When empty space skipping is required for volume rendering [S1, unofficial].
*   **When not to use:** When memory bandwidth is a critical bottleneck due to multiple passes [G].

## Background
Origin: [S1, unofficial] describes the method as a GPU compute shader dispatch routine. [S2] discusses distance field precomputation for fiber surface ray casting, noting that distance field computation is on the order of milliseconds and not a bottleneck [S2].

## Sources
[S1] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
[S2] Direct Multifield Volume Ray Casting of Fiber Surfaces, https://doi.org/10.1109/tvcg.2016.2599040, fetched 2026-10-26
[S3] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
