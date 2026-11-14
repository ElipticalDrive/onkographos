# Trilinear Volume Sampling for Texture Generation

Also called: Surface query interpolation, TexF Feature Interpolation, Trilinear Texture Sampling, Trilinear Volume Sampling, Trilinear interpolation in TexF

## Overview
Trilinear Volume Sampling retrieves scalar or vector attributes from a 3D voxel grid at continuous surface positions by interpolating values from the surrounding 2x2x2 neighborhood of active grid points [S1, unofficial]. It enables spatially coherent attribute coding and direct surface queries without UV mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D attribute volume (e.g., color, material properties) and a query position $p$ in world space [S3, unofficial].
*   **Output:** The interpolated attribute value $c(p)$ at position $p$ [S1, unofficial].

**Data Structures:**
*   **3D Texture/Tensor:** Stores attribute values. In CUDA implementations, the underlying storage may be `ushort` but accessed with `cudaReadModeNormalizedFloat` to treat values as normalized floats in $[0, 1]$ [S2, unofficial].
*   **Grid Points:** The vertices of the voxel cell containing $p$ [S1, unofficial].

**Steps:**
1.  **Coordinate Transformation:** Map the query position $p$ from world space to grid indices by subtracting the AABB origin and dividing by voxel size [S3, unofficial].
2.  **Neighborhood Identification:** Identify the 2x2x2 neighborhood of active grid points $q_i$ surrounding the query position $p$ [S1, unofficial].
3.  **Weight Calculation:** Compute trilinear weights $\omega_i$ for each neighbor based on the relative position of $p$ within the grid cell [S1, unofficial].
4.  **Interpolation:** Compute the weighted sum of the attribute values $t(q_i)$ at the grid points [S1, unofficial].

**Formula:**
The interpolated attribute $c(p)$ is calculated as [S1, unofficial]:

$$
c(p) = \sum_{i \in N(p)} \omega_i t(q_i)
$$

Where:
*   $N(p)$ is the set of active grid points within the 2x2x2 neighborhood of $p$ [S1, unofficial].
*   $t(q_i)$ is the attribute value at grid point $q_i$ [S1, unofficial].
*   $\omega_i$ are the trilinear weights determined by the query position within the grid cell [S1, unofficial]. For a position $(x, y, z)$ within a unit cube, the weight for a corner $(x_c, y_c, z_c)$ is $(1 - |x - x_c|)(1 - |y - y_c|)(1 - |z - z_c|)$, normalized such that $\sum \omega_i = 1$ [G].

**Parameters:**
*   **Sampling Mode:** 'trilinear' [S3, unofficial].
*   **Texture Read Mode:** `cudaReadModeNormalizedFloat` if the underlying type is `ushort` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trilinear_sample_3d(attr_volume, query_pos, grid_size, aabb_origin, voxel_size):
    """
    Samples a 3D attribute volume at a continuous position using trilinear interpolation.
    
    Args:
        attr_volume: 3D tensor/array containing attribute values.
        query_pos: float3, world-space query position.
        grid_size: tuple (D, H, W), dimensions of the voxel grid.
        aabb_origin: float3, origin of the voxel grid in world space.
        voxel_size: float, size of each voxel.
        
    Returns:
        float or float3: Interpolated attribute value(s).
    """
    # 1. Transform world position to grid coordinates
    grid_x = (query_pos.x - aabb_origin.x) / voxel_size
    grid_y = (query_pos.y - aabb_origin.y) / voxel_size
    grid_z = (query_pos.z - aabb_origin.z) / voxel_size
    
    # 2. Identify the 2x2x2 neighborhood using floor for correct negative coordinate handling
    ix = math.floor(grid_x)
    iy = math.floor(grid_y)
    iz = math.floor(grid_z)
    
    # Clamp indices to valid range [0, size-1] to handle boundary cases
    ix = max(0, min(ix, grid_size[0] - 1))
    iy = max(0, min(iy, grid_size[1] - 1))
    iz = max(0, min(iz, grid_size[2] - 1))
    
    # 3. Calculate fractional offsets (weights)
    fx = grid_x - ix
    fy = grid_y - iy
    fz = grid_z - iz
    
    # 4. Retrieve values from the 8 neighbors
    # Assuming attr_volume is indexed as [x, y, z] or [z, y, x].
    # Adjust indexing order to match actual memory layout.
    v000 = attr_volume[ix, iy, iz]
    v100 = attr_volume[min(ix + 1, grid_size[0] - 1), iy, iz]
    v010 = attr_volume[ix, min(iy + 1, grid_size[1] - 1), iz]
    v110 = attr_volume[min(ix + 1, grid_size[0] - 1), min(iy + 1, grid_size[1] - 1), iz]
    v001 = attr_volume[ix, iy, min(iz + 1, grid_size[2] - 1)]
    v101 = attr_volume[min(ix + 1, grid_size[0] - 1), iy, min(iz + 1, grid_size[2] - 1)]
    v011 = attr_volume[ix, min(iy + 1, grid_size[1] - 1), min(iz + 1, grid_size[2] - 1)]
    v111 = attr_volume[min(ix + 1, grid_size[0] - 1), min(iy + 1, grid_size[1] - 1), min(iz + 1, grid_size[2] - 1)]
    
    # 5. Compute trilinear weights
    w000 = (1 - fx) * (1 - fy) * (1 - fz)
    w100 = fx * (1 - fy) * (1 - fz)
    w010 = (1 - fx) * fy * (1 - fz)
    w110 = fx * fy * (1 - fz)
    w001 = (1 - fx) * (1 - fy) * fz
    w101 = fx * (1 - fy) * fz
    w011 = (1 - fx) * fy * fz
    w111 = fx * fy * fz
    
    # 6. Interpolate
    result = (w000 * v000 + w100 * v100 + w010 * v010 + w110 * v110 +
              w001 * v001 + w101 * v101 + w011 * v011 + w111 * v111)
              
    return result
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** If the query position lies outside the grid, indices must be clamped or the query rejected. The pseudocode clamps to the nearest valid voxel [G].
*   **Active Grid Points:** In sparse representations, only "active" grid points should be considered [S1, unofficial].

## Subtopics

### [Volume Sampling](7111-volume-sampling.md)
Retrieves the scalar density value from a 3D texture at specified normalized texture coordinates. It serves as the fundamental data retrieval primitive for raycasting-based volume rendering, providing the raw intensity values required for ray marching, isosurface detection, and maximum intensity projection. Unlike its parent method, which handles coordinate transformation and interpolation, this method focuses strictly on the final fetch operation from the sampled texture resource. Choose it when only the raw texture fetch is needed without coordinate mapping.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, as it involves a fixed number of memory accesses (8) and arithmetic operations [G].
*   **Memory Complexity:** Depends on the size of the 3D attribute volume [G].
*   **Strengths:** Provides smooth, continuous attribute values from discrete voxel data; enables direct surface queries without UV mapping; spatially coherent [S1, unofficial].
*   **Limitations:** Requires the query position to be mapped to the grid; boundary handling can introduce artifacts if not managed [G].
*   **When to Use:** When continuous attribute interpolation is needed from a voxel grid, especially for surface-based rendering or implicit geometry [G].
*   **When Not to Use:** When exact discrete values are required, or when the grid is extremely sparse and a sparse data structure (like a tree) is more efficient [G].

## Background
The method is central to Texture Fields (TexF), a representation that preserves local 3D spatial relationships among texture attributes, supporting direct surface queries and spatially coherent attribute coding [S1, unofficial]. It is mathematically analogous to bilinear sampling in UV rendering but operates in 3D [S1, unofficial]. The approach allows attributes to be shared with grid-based implicit geometry, such as an SDF, for joint geometry–appearance modeling [S1, unofficial].

## Sources
[S1] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
[S2] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S3] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S4] tkris007/webGL-volume-rendering, https://github.com/tkris007/webGL-volume-rendering, fetched 2026-10-27 (unofficial)
