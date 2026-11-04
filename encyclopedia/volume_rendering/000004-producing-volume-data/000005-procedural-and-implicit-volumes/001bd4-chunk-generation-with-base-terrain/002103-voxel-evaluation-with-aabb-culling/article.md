# Voxel Evaluation with AABB Culling

Also called: GPU Voxel Evaluation, SDF Voxelization

## Overview
Voxel Evaluation with AABB Culling is a GPU-accelerated technique for computing Signed Distance Function (SDF) values for a chunk's voxel grid by restricting computation to voxels intersecting active SDF primitives. It solves the problem of unnecessary GPU computation for voxels far from any terrain geometry. Unlike [Chunk-Based Voxel Terrain Generation](4140-chunk-based-voxel-terrain-generation.md) which may process the entire chunk volume, this method uses Axis-Aligned Bounding Box (AABB) culling to identify a minimal sub-region of interest, significantly reducing the number of voxels processed per frame.

## How it works
**Inputs:**
- `ChunkDimensions`: World-space dimensions of the chunk (float3).
- `voxelRes`: Voxel resolution per LOD (int3). Must be odd.
- `terrainSDFRegistry`: Registry containing active SDF primitives and their bounds.
- `chunkBounds`: World-space bounds of the current chunk.
- `chunk.MinCornerWS`: World-space minimum corner of the chunk.

**Outputs:**
- `_Voxels`: Buffer containing SDF values for the cull region.
- Compute shader dispatch parameters defining the work group.

**Data Structures:**
- `VoxelDebugData`: Struct for debugging, containing `voxels` (float array), `resolution` (int3), `gridMinCorner` (float3), and `voxelSize` (float3).
- `TerrainSDFRegistry`: Provides `GetInnerBounds(chunkBounds)` which returns the intersection of active SDF bounds within the chunk.

**Steps:**

1.  **Calculate Voxel Size:**
    Compute the physical size of a single voxel in world space.
    $$ \text{voxelSize} = \frac{\text{ChunkDimensions}}{\text{voxelRes} - 1} $$
    *Note: The source uses `voxelRes - 1` in the denominator, implying the resolution defines the number of intervals or that the grid includes both endpoints.* [S1, unofficial]

2.  **Define Bounds:**
    Set `chunkBounds` to the full chunk extent. Initialize `voxelBounds` to `chunkBounds`.

3.  **Culling (Optional):**
    If culling is enabled, query the registry for the inner bounds of active SDFs intersecting the chunk:
    $$ \text{voxelBounds} = \text{terrainSDFRegistry.GetInnerBounds}(\text{chunkBounds}) $$

4.  **Degeneracy Check:**
    Check if `voxelBounds` contains invalid values (infinity).
    If $\exists i \in \{x,y,z\}$ such that $\text{boundsDiff}_i = -\infty$ or $\text{boundsDiff}_i = +\infty$, abort processing for this chunk. [S1, unofficial]
    *Where $\text{boundsDiff} = \text{voxelBounds.Max} - \text{voxelBounds.Min}$.*

5.  **Calculate Region Resolution:**
    Determine the integer resolution of the cull region, adding a 1-voxel halo for marching cubes sampling stability.
    $$ \text{regionRes}_i = \left\lceil \frac{\text{boundsDiff}_i}{\text{voxelSize}_i} + 1 \right\rceil $$
    for $i \in \{x, y, z\}$. [S1, unofficial]

6.  **Calculate Voxel Count:**
    $$ \text{numVoxelsInSDFBounds} = \text{regionRes}_x \times \text{regionRes}_y \times \text{regionRes}_z $$
    If `numVoxelsInSDFBounds` is 0, return immediately.

7.  **Calculate Starting Index:**
    Compute the 1D linear index in the full chunk grid corresponding to the minimum corner of the cull region.
    $$ \text{startingIndex} = \text{GetStartingIndex}(\text{voxelBounds.Min}, \text{chunk.MinCornerWS}, \text{voxelSize}, \text{voxelRes}) $$
    *Implementation detail: This maps the world-space minimum of the cull region to the nearest voxel index within the chunk's local grid.* [S1, unofficial]

8.  **Dispatch Compute Shader:**
    Dispatch the voxel evaluation shader with the following parameters:
    - `_Res`: `voxelRes` (int3)
    - `_VoxelSizeWS`: `voxelSize` (float3)
    - `_GridMinWS`: `chunk.MinCornerWS` (float3)
    - `_StartingIndex`: `startingIndex` (int)
    - `_InnerVoxelDims`: `regionRes` (int3)
    - `_Voxels`: Output buffer
    - `_SDFCount`: Number of active SDFs
    - `_SDFs`: Buffer of SDF primitives

    Thread group count:
    $$ \left( \left\lceil \frac{\text{numVoxelsInSDFBounds} + 63}{64} \right\rceil, 1, 1 \right) $$
    This configures a 1D grid of thread groups, each handling up to 64 voxels. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_voxels_aabb_culling(
    chunk_dimensions: float3,
    voxel_res: int3,
    chunk_min_corner_ws: float3,
    chunk_bounds: Bounds,
    sdf_registry: SDFRegistry,
    enable_culling: bool
) -> ComputeDispatch:
    """
    Computes voxel SDF values using AABB culling.
    Returns dispatch parameters for the GPU compute shader.
    """
    # Step 1: Calculate Voxel Size
    # voxelSize = ChunkDimensions / (voxelRes - 1)
    voxel_size = chunk_dimensions / (voxel_res - 1)
    
    # Step 2: Define Bounds
    voxel_bounds = chunk_bounds
    
    # Step 3: Culling
    if enable_culling:
        voxel_bounds = sdf_registry.get_inner_bounds(chunk_bounds)
    
    # Step 4: Degeneracy Check
    bounds_diff = voxel_bounds.max - voxel_bounds.min
    if any(bounds_diff == float('-inf')) or any(bounds_diff == float('inf')):
        return None # Abort
    
    # Step 5: Calculate Region Resolution (with 1-voxel halo)
    # regionRes = ceil(boundsDiff / voxelSize + 1)
    region_res = int3(
        ceil(bounds_diff.x / voxel_size.x + 1),
        ceil(bounds_diff.y / voxel_size.y + 1),
        ceil(bounds_diff.z / voxel_size.z + 1)
    )
    
    # Step 6: Calculate Voxel Count
    num_voxels = region_res.x * region_res.y * region_res.z
    if num_voxels == 0:
        return None # Abort
    
    # Step 7: Calculate Starting Index
    # GetStartingIndex maps world-space min of cull region to chunk-local 1D index
    starting_index = get_starting_index(
        voxel_bounds.min,
        chunk_min_corner_ws,
        voxel_size,
        voxel_res
    )
    
    # Step 8: Dispatch Parameters
    # Thread group count: ceil(numVoxels / 64)
    thread_groups_x = (num_voxels + 63) // 64
    
    dispatch = ComputeDispatch(
        thread_groups=(thread_groups_x, 1, 1),
        shader_params={
            "_Res": voxel_res,
            "_VoxelSizeWS": voxel_size,
            "_GridMinWS": chunk_min_corner_ws,
            "_StartingIndex": starting_index,
            "_InnerVoxelDims": region_res,
            "_SDFCount": sdf_registry.count,
            "_SDFs": sdf_registry.buffer
        },
        output_buffer="_Voxels"
    )
    
    return dispatch

def get_starting_index(cull_min_ws: float3, chunk_min_ws: float3, voxel_size: float3, res: int3) -> int:
    """
    Calculates the 1D linear index in the chunk grid for the voxel closest to cull_min_ws.
    """
    # Offset from chunk min
    offset = cull_min_ws - chunk_min_ws
    
    # Voxel coordinates relative to chunk start
    # Note: Using floor or round depends on exact grid alignment. 
    # Source implies a direct mapping. Assuming voxel centers or corners align.
    # Standard mapping: index = floor(offset / voxelSize)
    # However, source uses (voxelRes - 1) in size calc, suggesting grid points.
    # We assume standard integer indexing for the starting voxel.
    
    x = floor(offset.x / voxel_size.x)
    y = floor(offset.y / voxel_size.y)
    z = floor(offset.z / voxel_size.z)
    
    # Clamp to chunk bounds if necessary (though cull bounds should be inside)
    x = max(0, min(x, res.x - 1))
    y = max(0, min(y, res.y - 1))
    z = max(0, min(z, res.z - 1))
    
    # 1D Index = x + y * res.x + z * res.x * res.y
    return x + y * res.x + z * res.x * res.y
```

**Parameters:**
- `voxelRes`: Must be odd. Even resolutions trigger an error in validation. [S1, unofficial]
- `ChunkDimensions`: Should be cube-like (x=y=z) to avoid distortion, though not strictly enforced by the math. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Even Resolution:** The source validation explicitly errors if any dimension of `voxelRes` is even. [S1, unofficial]
- **Infinity Bounds:** If `GetInnerBounds` returns infinite extents, the method returns immediately to prevent invalid memory access or infinite loops. [S1, unofficial]
- **Zero Voxels:** If the cull region resolution calculates to 0, processing is skipped. [S1, unofficial]

## Subtopics

### [Chunked Parallel SDF Generation](4505-chunked-parallel-sdf-generation.md)
A parallel processing strategy that divides a world into independent chunks to compute Signed Distance Function (SDF) values for each chunk simultaneously using multi-threading, merging contributions from intersecting shapes via a minimum function. It solves the problem of sequential global SDF computation bottlenecks by enabling concurrent processing of spatial regions. Unlike single-threaded global approaches, this method allows for independent computation of disjoint world regions, significantly improving throughput for large scenes with multiple primitives.

## Operating characteristics
- **Time Complexity:** Proportional to the number of voxels in the cull region (`numVoxelsInSDFBounds`), not the full chunk volume. This is $O(V_{cull})$ where $V_{cull} \ll V_{chunk}$ when SDFs are sparse or localized.
- **Memory Complexity:** Proportional to `numVoxelsInSDFBounds` for the output buffer. The input SDF data is constant per frame.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with the number of active SDFs as long as their combined bounding volume remains small relative to the chunk. If SDFs cover the entire chunk, performance degrades to full voxel evaluation.
- **Strengths:** Drastically reduces GPU workload for sparse terrain or localized features. Avoids processing empty space.
- **Limitations:** Requires an additional query (`GetInnerBounds`) to determine the cull region. Overhead of culling logic must be less than the savings from skipped voxels.
- **When to use:** When SDF primitives are sparse or localized within large chunks.
- **When not to use:** When SDFs cover the entire chunk volume, as the culling overhead provides no benefit.

## Background
Originating from the "Marching-Cubes-Terrain" project by 2JacT, this method is part of a procedural terrain generation pipeline that uses Signed Distance Functions for smooth terrain shapes. It integrates with a chunk-based system where each chunk is processed independently on the GPU. The technique is designed to optimize the voxelization step for marching cubes, ensuring that only relevant voxels are evaluated for SDF values. [S1, unofficial]

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
