# SDF-based Binary Voxelization

Also called: Level Set Voxelization, SDF Voxelization

## Overview
SDF-based Binary Voxelization converts a continuous Signed Distance Field (SDF) into a discrete binary volume representation. It solves the problem of discretizing implicit surfaces defined by distance fields into a grid of occupied and unoccupied voxels. It differs from [Dense Volume SDF Generation](4499-dense-volume-sdf-generation.md) by discarding the continuous distance values in favor of a hard binary classification, and from [Signed Distance Field (SDF) Generation from Scanned Surfaces](854-signed-distance-field-sdf-generation-from-scanned-surfaces.md) by operating on an existing SDF rather than generating one from point clouds.

## How it works
**Inputs:** A 3D grid of signed distance values (e.g., `MeshSignedDistanceGrid`).
**Outputs:** A 3D boolean bitmap (`Bitmap3`) where each voxel indicates occupancy.

**Data Structures:**
*   `MeshSignedDistanceGrid`: A 3D array storing `float` values representing the signed distance to the surface at each grid point.
*   `Bitmap3`: A 3D array of `bool` values. Dimensions match the input SDF grid.

**Steps:**
1.  Initialize a `Bitmap3` object with dimensions identical to the input `MeshSignedDistanceGrid`.
2.  Iterate through every index $(i, j, k)$ in the `Bitmap3` grid.
3.  Retrieve the signed distance value $d$ from the SDF at the corresponding index: $d = \text{SDF}[i, j, k]$.
4.  Evaluate the occupancy condition:
    *   If $d < 0$, the point is inside the surface. Set `Bitmap3[i, j, k] = true`.
    *   Otherwise ($d \ge 0$), the point is outside or on the surface. Set `Bitmap3[i, j, k] = false`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_to_binary_voxelization(sdf_grid: Grid3D[float]) -> Grid3D[bool]:
    """
    Converts a signed distance field grid to a binary voxel grid.
    
    Args:
        sdf_grid: A 3D grid where each cell contains a float representing 
                  the signed distance to the nearest surface.
    
    Returns:
        A 3D boolean grid where True indicates inside the surface (negative distance)
        and False indicates outside (non-negative distance).
    """
    # Get dimensions from the input SDF grid
    dim_x = sdf_grid.width
    dim_y = sdf_grid.height
    dim_z = sdf_grid.depth
    
    # Initialize output binary grid with same dimensions
    # Implementation note: Initialize with False (outside) by default
    binary_grid = Grid3D[bool](width=dim_x, height=dim_y, depth=dim_z, default_value=False)
    
    # Iterate over all indices in the grid
    for x in range(dim_x):
        for y in range(dim_y):
            for z in range(dim_z):
                # Retrieve signed distance value at current index
                distance = sdf_grid.get_value(x, y, z)
                
                # Set voxel to true (inside) if distance is negative, otherwise false
                if distance < 0:
                    binary_grid.set_value(x, y, z, True)
                else:
                    binary_grid.set_value(x, y, z, False)
                    
    return binary_grid
```

**Edge Cases and Pitfalls:**
*   **Zero Distance:** Points exactly on the surface ($d=0$) are classified as outside (false). This creates a "hollow" shell if the surface thickness is negligible compared to voxel size.
*   **Precision:** Floating-point comparison errors near $d=0$ may cause jitter in the binary boundary. A small epsilon threshold is sometimes used, but the standard method uses strict inequality $d < 0$.
*   **Boundary Conditions:** The grid boundaries must align with the SDF domain. No extrapolation is performed; voxels outside the SDF domain are not generated.

## Subtopics

### [LOD Ground Truth Generation from Surface Intersections](4466-lod-ground-truth-generation-from-surface-intersections.md)
Constructs a Level-of-Detail (LOD) representation for a scene by downsampling high-resolution binary voxels and Signed Distance Fields (SDFs), extracting surface intersections along axes, and storing them in a sparse pointer/data structure per grid cell. It differs from standard binary voxelization by preserving continuous surface geometry information (intersections and normals) at lower resolutions, enabling more accurate ground truth generation for neural rendering or implicit surface reconstruction tasks.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the total number of voxels in the grid. Each voxel requires one lookup and one comparison.
*   **Memory Complexity:** $O(N)$ bits (or bytes, depending on implementation) for the output bitmap. The input SDF requires $O(N)$ floats.
*   **Performance:** Extremely fast as it involves no geometric intersection calculations or ray marching, only array traversal.
*   **Strengths:** Simple to implement; deterministic; preserves the exact resolution of the input SDF.
*   **Limitations:** Loses sub-voxel geometric detail; suffers from aliasing if the SDF resolution is insufficient relative to the surface curvature.
*   **When to use:** When a discrete occupancy grid is required for downstream processing (e.g., marching cubes, neural radiance fields) and the SDF is already available.
*   **When not to use:** When sub-voxel accuracy is critical and the SDF resolution is low; when memory is extremely constrained and a sparse representation is preferred.

## Background
This method is a standard post-processing step in pipelines that generate meshes from SDFs. It is described in the context of creating "Minecraft-style" voxel meshes from signed distance fields. The technique is attributed to gradientspace in their tutorial on 3D bitmaps and mesh winding numbers.

## Sources
[S1] 3D Bitmaps, Minecraft Cubes, and Mesh Winding Numbers — gradientspace, https://www.gradientspace.com/tutorials/2017/12/14/3d-bitmaps-and-minecraft-meshes, fetched 2026-10-29 (unofficial)
