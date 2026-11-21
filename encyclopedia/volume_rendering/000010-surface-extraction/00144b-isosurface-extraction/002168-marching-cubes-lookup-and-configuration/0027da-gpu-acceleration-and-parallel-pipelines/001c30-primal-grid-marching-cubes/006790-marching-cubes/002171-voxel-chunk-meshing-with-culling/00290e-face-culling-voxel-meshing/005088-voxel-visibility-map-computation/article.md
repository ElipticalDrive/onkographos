# Voxel Visibility Map Computation

Also called: Face Visibility Culling, Voxel Block Visibility Types, VoxelOcclusionOptimizer.ComputeVisibleFaces, VoxelVisibility

## Overview
Voxel Visibility Map Computation determines which faces of solid voxels within a chunk are visible by checking the transparency of adjacent voxels and boundary conditions, producing a 3D map of visible face flags [S4, unofficial]. It differs from its sibling [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md) by explicitly generating a pre-computed visibility map data structure that decouples face determination from mesh assembly, allowing downstream optimizers to operate on a unified visibility state [S4, unofficial].

## How it works
**Inputs:** A voxel chunk containing a 3D grid of voxels, where each voxel has a solid/empty state [S4, unofficial]. The chunk dimensions are $X_{size}, Y_{size}, Z_{size}$ [S4, unofficial].
**Outputs:** A 3D array `visibilityMap` of type `VoxelFace[,,]` with dimensions $X_{size} \times Y_{size} \times Z_{size}$, where each element is a bitmask indicating which of the six faces are visible [S4, unofficial].

**Data Structures:**
1.  **Voxel:** Contains a boolean `IsSolid` field [S4, unofficial]. In some implementations, it may also contain pipeline identifiers or transparency flags [S2, unofficial].
2.  **VoxelFace:** A bitmask enum with flags `None`, `Xpos`, `Xneg`, `Ypos`, `Yneg`, `Zpos`, `Zneg` representing the six faces of a voxel [S4, unofficial].
3.  **VoxelVisibilityMap:** A 3D array storing `VoxelFace` values for every coordinate in the chunk [S4, unofficial].

**Algorithm Steps:**
1.  Initialize `visibilityMap` with `VoxelFace.None` for all coordinates [S4, unofficial].
2.  Iterate through every coordinate $(x, y, z)$ in the chunk in ascending order of X, Y, and Z axes [S4, unofficial].
3.  Retrieve the voxel at $(x, y, z)$. If `voxel.IsSolid` is false, skip to the next coordinate (the visibility remains `None`) [S4, unofficial].
4.  For each of the six faces, determine visibility:
    *   Calculate neighbor coordinates using face-specific deltas:
        *   $Xpos: (x+1, y, z)$
        *   $Xneg: (x-1, y, z)$
        *   $Ypos: (x, y+1, z)$
        *   $Yneg: (x, y-1, z)$
        *   $Zpos: (x, y, z+1)$
        *   $Zneg: (x, y, z-1)$ [S4, unofficial].
    *   Check if the neighbor is transparent or out of bounds [S4, unofficial].
    *   If the neighbor is transparent or out of bounds, set the corresponding flag in `visibleFaces` [S4, unofficial].
5.  Store the computed `visibleFaces` in `visibilityMap[x, y, z]` [S4, unofficial].

**Helper Function: IsAdjacentVoxelTransparent**
This function checks if a neighbor voxel at $(nx, ny, nz)$ is transparent [S4, unofficial].
1.  If $(nx, ny, nz)$ is out of the chunk bounds, return `true` (treat out-of-bounds as transparent/visible) [S4, unofficial].
2.  Retrieve the neighbor voxel [S4, unofficial].
3.  If the neighbor is empty or transparent, return `true` [S4, unofficial].
4.  If the neighbor is solid, return `false` (the face is occluded) [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelFace:
    NONE = 0
    XPOS = 1
    XNEG = 2
    YPOS = 4
    YNEG = 8
    ZPOS = 16
    ZNEG = 32

def compute_visibility_map(chunk):
    """
    Computes the visibility map for a voxel chunk.
    
    Args:
        chunk: An object containing voxel data and dimensions.
        
    Returns:
        visibility_map: A 3D array of VoxelFace bitmasks.
    """
    x_size, y_size, z_size = chunk.dimensions
    visibility_map = [[[VoxelFace.NONE for _ in range(z_size)] for _ in range(y_size)] for _ in range(x_size)]
    
    for x in range(x_size):
        for y in range(y_size):
            for z in range(z_size):
                voxel = chunk.get_voxel(x, y, z)
                
                # If voxel is not solid, no faces are visible
                if not voxel.is_solid:
                    continue
                
                visible_faces = VoxelFace.NONE
                
                # Define face deltas and corresponding flags: Xpos, Xneg, Ypos, Yneg, Zpos, Zneg
                faces = [
                    (1, 0, 0, VoxelFace.XPOS),
                    (-1, 0, 0, VoxelFace.XNEG),
                    (0, 1, 0, VoxelFace.YPOS),
                    (0, -1, 0, VoxelFace.YNEG),
                    (0, 0, 1, VoxelFace.ZPOS),
                    (0, 0, -1, VoxelFace.ZNEG)
                ]
                
                for dx, dy, dz, flag in faces:
                    nx, ny, nz = x + dx, y + dy, z + dz
                    
                    # Check if neighbor is transparent or out of bounds
                    if is_neighbor_transparent(chunk, nx, ny, nz, x_size, y_size, z_size):
                        visible_faces |= flag
                
                visibility_map[x][y][z] = visible_faces
    
    return visibility_map

def is_neighbor_transparent(chunk, nx, ny, nz, x_size, y_size, z_size):
    """
    Checks if the neighbor voxel at (nx, ny, nz) is transparent.
    
    Args:
        chunk: The voxel chunk.
        nx, ny, nz: Neighbor coordinates.
        x_size, y_size, z_size: Chunk dimensions.
        
    Returns:
        True if the neighbor is transparent or out of bounds, False otherwise.
    """
    # Out of bounds is treated as transparent (visible face) [S4, unofficial]
    if not (0 <= nx < x_size and 0 <= ny < y_size and 0 <= nz < z_size):
        return True
    
    neighbor_voxel = chunk.get_voxel(nx, ny, nz)
    
    # If neighbor is empty or transparent, the face is visible [S4, unofficial]
    return not neighbor_voxel.is_solid
```

**Parameters:**
*   Chunk dimensions: Typically powers of 2 (e.g., 16x16x16) for efficient memory alignment and culling [S4, unofficial].
*   Voxel types: `Empty`, `Transparent`, `Opaque` [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Out of Bounds:** Neighbor coordinates outside the chunk bounds must be treated as transparent to ensure boundary faces are visible [S4, unofficial].
*   **Degenerate Input:** Empty chunks or chunks with no solid voxels result in an all-`None` visibility map [S4, unofficial].

## Subtopics

### [Voxel Face Occlusion Culling with Chunk Boundary Wrapping](13197-voxel-face-occlusion-culling-with-chunk-boundary-wrapping.md)
This method computes the occlusion status for all six faces of every voxel within a chunk, determining which faces are visible for mesh generation. It solves the problem of correctly identifying boundary conditions when a chunk is part of a larger, potentially toroidal or irregularly tiled world by wrapping coordinates to adjacent neighbor chunks. It differs from standard intra-chunk culling by explicitly handling cross-chunk neighbor lookups and coordinate wrapping, ensuring that faces on the chunk boundary are correctly marked as occluded or visible based on the state of neighboring chunks.

### [Face State Propagation](14159-face-state-propagation.md)
Face State Propagation is a method to reduce redundant neighbor checks when building voxel meshes by storing face visibility state directly within each voxel's data structure, allowing simultaneous determination of visibility for two adjacent voxels. It differs from naive neighbor checking by pre-computing and caching the visibility of shared faces, trading memory usage for reduced CPU checks during mesh generation.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is processed once, and for each voxel, six neighbor checks are performed [S4, unofficial].
*   **Memory Complexity:** $O(N)$ for the `visibilityMap` array, where each element is a `VoxelFace` bitmask [S4, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with chunk size. Larger chunks may benefit from parallelization, but the algorithm is inherently sequential in its current description [S4, unofficial].
*   **Strengths:** Simple to implement, efficient for sparse voxels (skips non-solid voxels), provides a clear visibility state for downstream meshing [S4, unofficial].
*   **Limitations:** Does not handle cross-chunk visibility without additional logic (e.g., [Voxel Face Occlusion Culling with Chunk Boundary Wrapping](13197-voxel-face-occlusion-culling-with-chunk-boundary-wrapping.md)), may require additional memory for the visibility map [S4, unofficial].
*   **When to Use:** When a pre-computed visibility map is needed for efficient mesh assembly [S4, unofficial].
*   **When Not to Use:** When memory is constrained and a full visibility map cannot be stored [S4, unofficial].

## Background
Voxel Visibility Map Computation is a technique used in voxel mesh optimization libraries to efficiently determine which faces of voxels should be included in the final mesh. It is part of a pipeline that includes occlusion culling, disjoint-set quad merging, and mesh assembly [S4, unofficial]. The method is implemented in libraries such as `VoxelMeshOptimizerLibrary` [S4, unofficial].

## Sources
[S1] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S3] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S4] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
