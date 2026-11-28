# Voxel Evaluation with AABB Culling

Also called: GPU Voxel Evaluation, SDF Voxelization

## Overview
Voxel Evaluation with AABB Culling is a GPU-accelerated technique for computing Signed Distance Function (SDF) values for a chunk's voxel grid by restricting computation to voxels intersecting active SDF primitives [S1, unofficial]. It solves the problem of unnecessary GPU computation for voxels far from any terrain geometry [S1, unofficial].

## How it works
**Inputs:**
- `ChunkDimensions`: World-space dimensions of the chunk (float3) [S1, unofficial].
- `voxelRes`: Voxel resolution per LOD (int3) [S1, unofficial].
- `terrainSDFRegistry`: Registry containing active SDF primitives and their bounds [S1, unofficial].
- `chunkBounds`: World-space bounds of the current chunk [S1, unofficial].
- `chunk.MinCornerWS`: World-space minimum corner of the chunk [S1, unofficial].

**Outputs:**
- `_Voxels`: Buffer containing SDF values for the cull region [S1, unofficial].
- Compute shader dispatch parameters defining the work group [S1, unofficial].

**Data Structures:**
- `VoxelDebugData`: Struct for debugging, containing `voxels` (float array), `resolution` (int3), `gridMinCorner` (float3), and `voxelSize` (float3) [S1, unofficial].
- `TerrainSDFRegistry`: Provides `GetInnerBounds(chunkBounds)` which returns the intersection of active SDF bounds within the chunk [S1, unofficial].

**Steps:**

1.  **Calculate Voxel Size:**
    Compute the physical size of a single voxel in world space.
    $$ \text{voxelSize} = \frac{\text{ChunkDimensions}}{\text{voxelRes} - 1} $$
    The source divides by `voxelRes - 1`, implying the resolution defines the number of intervals or that the grid includes both endpoints [S1, unofficial].

2.  **Define Bounds:**
    Set `chunkBounds` to the full chunk extent and initialize `voxelBounds` to `chunkBounds` [S1, unofficial]. Calculate total voxel count: `int voxelCount = voxelRes.x * voxelRes.y * voxelRes.z` [S1, unofficial].

3.  **Culling (Optional):**
    If culling is enabled, query the registry for the inner bounds of active SDFs intersecting the chunk:
    $$ \text{voxelBounds} = \text{terrainSDFRegistry.GetInnerBounds}(\text{chunkBounds}) $$
    [S1, unofficial].

4.  **Degeneracy Check:**
    Compute `boundsDiff = voxelBounds.Max - voxelBounds.Min` [S1, unofficial]. If `math.any(boundsDiff == float.NegativeInfinity) || math.any(boundsDiff == float.PositiveInfinity)`, return immediately [S1, unofficial].

5.  **Calculate Region Resolution:**
    Determine the integer resolution of the cull region, adding a 1-voxel halo for marching cubes sampling stability:
    $$ \text{regionRes}_x = \left\lceil \frac{\text{boundsDiff}_x}{\text{voxelSize}_x} + 1 \right\rceil, \quad \text{regionRes}_y = \left\lceil \frac{\text{boundsDiff}_y}{\text{voxelSize}_y} + 1 \right\rceil, \quad \text{regionRes}_z = \left\lceil \frac{\text{boundsDiff}_z}{\text{voxelSize}_z} + 1 \right\rceil $$
    [S1, unofficial].

6.  **Calculate Voxel Count:**
    $$ \text{numVoxelsInSDFBounds} = \text{regionRes}_x \times \text{regionRes}_y \times \text{regionRes}_z $$
    If `numVoxelsInSDFBounds` is 0, return immediately [S1, unofficial].

7.  **Calculate Starting Index:**
    Compute the 1D linear index in the full chunk grid corresponding to the minimum corner of the cull region:
    $$ \text{startingIndex} = \text{GetStartingIndex}(\text{voxelBounds.Min}, \text{chunk.MinCornerWS}, \text{voxelSize}, \text{voxelRes}) $$
    [S1, unofficial].

8.  **Dispatch Compute Shader:**
    Dispatch the voxel evaluation shader with the following parameters: `_Res` (`voxelRes`), `_VoxelSizeWS` (`voxelSize`), `_GridMinWS` (`chunk.MinCornerWS`), `_StartingIndex` (`startingIndex`), `_InnerVoxelDims` (`regionRes`), `_Voxels` buffer, `_SDFCount`, and `_SDFs` buffer [S1, unofficial]. Thread group count: `((numVoxelsInSDFBounds + 63) / 64, 1, 1)` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_voxels_aabb_culling(
    chunk_dimensions: tuple,
    voxel_res: tuple,
    chunk_min_corner_ws: tuple,
    chunk_bounds_min: tuple,
    chunk_bounds_max: tuple,
    sdf_registry_inner_bounds_min: tuple,
    sdf_registry_inner_bounds_max: tuple,
    enable_culling: bool,
    sdf_count: int,
    sdf_buffer: list
) -> dict:
    # Step 1: Calculate Voxel Size
    voxel_size = (
        chunk_dimensions[0] / (voxel_res[0] - 1),
        chunk_dimensions[1] / (voxel_res[1] - 1),
        chunk_dimensions[2] / (voxel_res[2] - 1)
    )

    # Step 2: Define Bounds
    voxel_bounds_min = chunk_bounds_min
    voxel_bounds_max = chunk_bounds_max

    # Step 3: Culling
    if enable_culling:
        voxel_bounds_min = sdf_registry_inner_bounds_min
        voxel_bounds_max = sdf_registry_inner_bounds_max

    # Step 4: Degeneracy Check
    bounds_diff = (
        voxel_bounds_max[0] - voxel_bounds_min[0],
        voxel_bounds_max[1] - voxel_bounds_min[1],
        voxel_bounds_max[2] - voxel_bounds_min[2]
    )
    if (bounds_diff[0] in (float('-inf'), float('inf')) or
        bounds_diff[1] in (float('-inf'), float('inf')) or
        bounds_diff[2] in (float('-inf'), float('inf'))):
        return {}

    # Step 5: Calculate Region Resolution
    import math
    region_res = (
        int(math.ceil(bounds_diff[0] / voxel_size[0] + 1)),
        int(math.ceil(bounds_diff[1] / voxel_size[1] + 1)),
        int(math.ceil(bounds_diff[2] / voxel_size[2] + 1))
    )

    # Step 6: Calculate Voxel Count
    num_voxels = region_res[0] * region_res[1] * region_res[2]
    if num_voxels == 0:
        return {}

    # Step 7: Calculate Starting Index
    offset = (
        voxel_bounds_min[0] - chunk_min_corner_ws[0],
        voxel_bounds_min[1] - chunk_min_corner_ws[1],
        voxel_bounds_min[2] - chunk_min_corner_ws[2]
    )
    x = int(math.floor(offset[0] / voxel_size[0]))
    y = int(math.floor(offset[1] / voxel_size[1]))
    z = int(math.floor(offset[2] / voxel_size[2]))
    x = max(0, min(x, voxel_res[0] - 1))
    y = max(0, min(y, voxel_res[1] - 1))
    z = max(0, min(z, voxel_res[2] - 1))
    starting_index = x + y * voxel_res[0] + z * voxel_res[0] * voxel_res[1]

    # Step 8: Dispatch Parameters
    thread_groups_x = (num_voxels + 63) // 64
    return {
        "thread_groups": (thread_groups_x, 1, 1),
        "shader_params": {
            "_Res": voxel_res,
            "_VoxelSizeWS": voxel_size,
            "_GridMinWS": chunk_min_corner_ws,
            "_StartingIndex": starting_index,
            "_InnerVoxelDims": region_res,
            "_SDFCount": sdf_count,
            "_SDFs": sdf_buffer
        },
        "output_buffer": "_Voxels"
    }
```

**Parameters:**
- `voxelRes`: Must be odd [S1, unofficial].
- `ChunkDimensions`: Should be cube-like (x=y=z) to avoid distortion [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Even Resolution:** The source validation explicitly errors if any dimension of `voxelRes` is even [S1, unofficial].
- **Infinity Bounds:** If `GetInnerBounds` returns infinite extents, the method returns immediately to prevent invalid memory access [S1, unofficial].
- **Zero Voxels:** If the cull region resolution calculates to 0, processing is skipped [S1, unofficial].

## Subtopics

### [Chunked Parallel SDF Generation](4505-chunked-parallel-sdf-generation.md)
A parallel processing strategy that divides the world into independent chunks to compute Signed Distance Function (SDF) values for each chunk simultaneously using multi-threading, merging contributions from intersecting shapes via a minimum function. It solves the problem of sequential global SDF computation bottlenecks by enabling concurrent processing of spatial regions. Unlike single-threaded global approaches, this method allows for independent computation of disjoint world regions, significantly improving throughput for large scenes with multiple primitives.

## Operating characteristics
- **Time Complexity:** Proportional to the number of voxels in the cull region (`numVoxelsInSDFBounds`), not the full chunk volume [S1, unofficial].
- **Memory Complexity:** Proportional to `numVoxelsInSDFBounds` for the output buffer [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with the number of active SDFs as long as their combined bounding volume remains small relative to the chunk [S1, unofficial].
- **Strengths:** Drastically reduces GPU workload for sparse terrain or localized features [S1, unofficial].
- **Limitations:** Requires an additional query (`GetInnerBounds`) to determine the cull region [S1, unofficial].
- **When to use:** When SDF primitives are sparse or localized within large chunks [S1, unofficial].
- **When not to use:** When SDFs cover the entire chunk volume, as the culling overhead provides no benefit [S1, unofficial].

## Background
Originating from the "Marching-Cubes-Terrain" project by 2JacT, this method is part of a procedural terrain generation pipeline that uses Signed Distance Functions for smooth terrain shapes [S1, unofficial]. It integrates with a chunk-based system where each chunk is processed independently on the GPU [S1, unofficial]. The technique is designed to optimize the voxelization step for marching cubes, ensuring that only relevant voxels are evaluated for SDF values [S1, unofficial].

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
