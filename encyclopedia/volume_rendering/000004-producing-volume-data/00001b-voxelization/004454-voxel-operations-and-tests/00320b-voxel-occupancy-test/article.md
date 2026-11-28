# Voxel Occupancy Test

Also called: Voxel Mask, Voxel Mask Output

## Overview
The Voxel Occupancy Test determines whether continuous query points lie within occupied voxels of a discrete 3D grid, producing a binary mask [S1, S5]. It operates on a pre-constructed voxel grid structure rather than explicit axis-aligned bounds [G].

## How it works
**Inputs:**
1. An array of query points $P$ with shape $(N, 3)$, where each row is a continuous coordinate $(x, y, z)$ [S1, S5].

**Outputs:**
1. A boolean array or binary `UInt8` mask $M$ of length $N$, where $M[i] = 1$ (or `True`) if point $P[i]$ is inside an occupied voxel, and $0$ (or `False`) otherwise [S1, S5][S4, unofficial].

**Data Structures:**
A voxel grid data structure representing the spatial domain, typically created from a point cloud or mesh with a defined `voxel_size` [S1, S5]. The occupancy status is stored as a boolean or integer flag per voxel [G].

**Steps:**
1. **Coordinate Mapping**: For each query point $P_i = (x, y, z)$, map the continuous coordinates to discrete voxel indices $(i, j, k)$ by subtracting the grid origin and dividing by the voxel size. The specific axis ordering must match the grid's storage convention [G].
2. **Boundary Check**: Verify if the computed indices $(i, j, k)$ fall within the valid bounds of the voxel grid. If any component is negative or greater than or equal to the grid dimensions, the point is outside [S2, unofficial].
3. **Occupancy Query**: Access the voxel at $(i, j, k)$ in the grid data structure.
4. **Result Assignment**: If the voxel is marked as occupied, set the corresponding output mask value to $1$ (or `True`); otherwise, set it to $0$ (or `False`) [G].
5. **Aggregation**: If multiple points map to the same voxel, the mask records the status for each point individually. Duplicate points and multiple points per voxel collapse to a single $1$ in a global mask, but the primary test returns per-point results [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_occupancy_test(voxel_grid, query_points):
    """
    Tests if query points are within occupied voxels.
    
    Args:
        voxel_grid: Object with attributes origin (tuple), voxel_size (float),
                    dimensions (tuple), and is_occupied(i, j, k) method.
        query_points: List of tuples or list of lists, shape (N, 3).
        
    Returns:
        mask: List of integers (0 or 1) of length N.
    """
    N = len(query_points)
    mask = [0] * N
    origin = voxel_grid.origin
    v_size = voxel_grid.voxel_size
    dims = voxel_grid.dimensions

    for idx in range(N):
        p = query_points[idx]
        
        # Check for non-finite coordinates
        if not (math.isfinite(p[0]) and math.isfinite(p[1]) and math.isfinite(p[2])):
            mask[idx] = 0
            continue
            
        # Coordinate mapping
        i = int((p[0] - origin[0]) / v_size)
        j = int((p[1] - origin[1]) / v_size)
        k = int((p[2] - origin[2]) / v_size)
        
        # Boundary check
        if i < 0 or i >= dims[0] or j < 0 or j >= dims[1] or k < 0 or k >= dims[2]:
            mask[idx] = 0
            continue
            
        # Occupancy query
        occupied = voxel_grid.is_occupied(i, j, k)
        
        # Result assignment
        if occupied:
            mask[idx] = 1
            
    return mask
```

**Parameters:**
- `voxel_size`: The side length of a voxel. Typical values depend on the scene scale (e.g., $0.05$ in Open3D examples) [S1, S5].
- `origin`: The world coordinate of the grid's $(0,0,0)$ voxel. Default is often $(0,0,0)$ [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-Bounds Points**: Points outside the destination grid are silently dropped with no warning, resulting in an all-zero mask if the origin or spacing is mismatched [S4, unofficial].
- **Non-Finite Coordinates**: Points with NaN or Inf coordinates are skipped [S4, unofficial].
- **I/O Performance**: If the mask is written in point order, a spatially unsorted cloud produces scattered writes, which is I/O-bound with out-of-core storage; pre-sorting the point cloud spatially helps [S4, unofficial].
- **Coordinate Mismatch**: If the origin or spacing of the query points does not match the voxel grid's geometry, points may be incorrectly classified or dropped [S4, unofficial].

## Subtopics

### Voxel Contains Scene Geometry Test
A method to determine if any scene geometry exists inside a voxel, used to flag voxels for subdivision [S6, unofficial]. It differs from standard point-inclusion tests by using rendering techniques to detect geometry presence rather than geometric intersection [S6, unofficial]. It determines if scene geometry intersects a voxel volume [S6, unofficial]. The steps involve sampling scene depth at the voxel corners, where only some cubemap texels are used to test for intersection [S6, unofficial]. Alternatively, it uses occlusion queries by placing the camera at the center of the voxel, rendering into each face of a cubemap (first drawing quads for the voxel faces, then the scene), and checking if any draw calls pass the occlusion query [S6, unofficial]. It should be chosen for adaptive subdivision or irradiance volume computation where geometric presence within a volume is required, rather than point containment [S6, unofficial].

## Operating characteristics
Time and memory complexity: Not given in the sources. Performance: Fast for batched queries; performance degrades if the point cloud is unsorted and writes are scattered to memory [S4, unofficial]. Strengths: Simple, efficient for point-in-voxel queries; supports batch processing. Limitations: Does not handle partial voxel occupancy (binary only); sensitive to coordinate system mismatches [S4, unofficial]. When to use: When you need to know which points from a cloud lie inside a voxelized representation [G]. When not to use: When you need to know which voxel a point is in for spatial indexing (use [Mask Cache Lookup](6434-mask-cache-lookup.md) or index generation), or when you need to detect geometry inside a voxel using rendering (use [Voxel Contains Scene Geometry Test](1281-voxel-occupancy-test.md#voxel-contains-scene-geometry-test)) [G].

## Background
The Voxel Occupancy Test is a fundamental operation in voxel-based graphics and geometry processing [G]. It is commonly used in point cloud processing pipelines to convert continuous point data into discrete voxel representations [G]. The method `check_if_included` is part of the Open3D library's voxel grid API, which allows testing if points are within an occupied voxel after voxelization [S1, S5]. The concept is also used in game development for irradiance volume computation and adaptive subdivision [S6, unofficial].

## Sources
[S1] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S2] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
[S3] tonadr1022/vulkan_voxels, https://github.com/tonadr1022/vulkan_voxels, fetched 2026-10-27 (unofficial)
[S4] FILT: Voxelize Point Cloud, https://github.com/BlueQuartzSoftware/simplnx/pull/1724, fetched 2026-10-27 (unofficial)
[S5] Voxelization - Open3D 0.19.0 documentation, https://www.open3d.org/docs/release/tutorial/geometry/voxelization.html, fetched 2026-10-28
[S6] PDF Irradiance Volumes for Games, https://chrisoat.com/papers/Oat_GDC2005_IrradianceVolumesForGames.pdf, fetched 2026-10-28 (unofficial)
