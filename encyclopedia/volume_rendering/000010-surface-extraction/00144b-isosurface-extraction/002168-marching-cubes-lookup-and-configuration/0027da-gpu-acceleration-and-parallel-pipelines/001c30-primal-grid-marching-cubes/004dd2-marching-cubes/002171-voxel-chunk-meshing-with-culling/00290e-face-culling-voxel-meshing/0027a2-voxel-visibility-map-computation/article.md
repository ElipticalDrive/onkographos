# Voxel Visibility Map Computation

Also called: Face Visibility Culling, Voxel Block Visibility Types, VoxelOcclusionOptimizer.ComputeVisibleFaces, VoxelVisibility

## Overview
Voxel Visibility Map Computation determines which faces of solid voxels are visible by checking if adjacent voxels are transparent or out of bounds, producing a 3D map of visible face flags [S3, unofficial]. It solves the problem of efficiently culling non-visible voxel faces before mesh generation, reducing the triangle count and rendering load [S3, unofficial]. It differs from its siblings by explicitly computing a per-voxel bitmask of visible faces based on neighbor transparency, rather than generating geometry directly or relying on post-generation culling. It builds on [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md) by providing the visibility data required for that meshing step.

## How it works
**Inputs:**
- A chunk of voxels, defined by dimensions $W \times H \times D$ (X, Y, Z depths) [S3, unofficial].
- A voxel data structure where each voxel has a property `IsSolid` (boolean) [S3, unofficial].
- A neighbor lookup mechanism `GetBlockSafe(x, y, z)` that returns the block/voxel at coordinates, treating out-of-bounds coordinates as transparent [S3, unofficial].
- A helper function `IsAdjacentVoxelTransparent(neighbor)` that returns true if the neighbor is not solid or is empty [S3, unofficial].

**Outputs:**
- A 3D array `visibilityMap` of type `VoxelFace[,,]` with dimensions $W \times H \times D$ [S3, unofficial].
- Each entry `visibilityMap[x, y, z]` is a bitmask representing the visible faces of the voxel at $(x, y, z)$.

**Data Structures:**
- `VoxelFace`: A bitmask enum with flags `None`, `Xpos`, `Xneg`, `Ypos`, `Yneg`, `Zpos`, `Zneg` [S3, unofficial].
  - `None`: No visible faces.
  - `Xpos`: Positive X face visible.
  - `Xneg`: Negative X face visible.
  - `Ypos`: Positive Y face visible.
  - `Yneg`: Negative Y face visible.
  - `Zpos`: Positive Z face visible.
  - `Zneg`: Negative Z face visible.
- `VoxelVisibility`: An enum with variants `Empty`, `Translucent`, `Opaque` used to decide face visibility [S1, unofficial]. Note: The implementation in [S3, unofficial] uses `IsSolid` and neighbor transparency checks directly, while [S1, unofficial] and [S2, unofficial] reference block-level transparency properties. The core logic relies on the neighbor being transparent [S2, S3, unofficial].

**Steps:**
1. Initialize `visibilityMap` with all entries set to `VoxelFace.None` [S3, unofficial].
2. Iterate through every coordinate $(x, y, z)$ in the chunk in ascending order for X, Y, and Z axes [S3, unofficial].
3. Retrieve the voxel at $(x, y, z)$.
4. If `voxel.IsSolid` is false, set `visibilityMap[x, y, z]` to `VoxelFace.None` and continue to the next coordinate [S3, unofficial].
5. Initialize `visibleFaces` to `VoxelFace.None` [S3, unofficial].
6. For each of the six faces, check the neighbor voxel:
   - **Zpos**: Check neighbor at $(x, y, z+1)$. If `IsAdjacentVoxelTransparent` returns true, set `Zpos` flag in `visibleFaces` [S3, unofficial].
   - **Zneg**: Check neighbor at $(x, y, z-1)$. If `IsAdjacentVoxelTransparent` returns true, set `Zneg` flag in `visibleFaces` [S3, unofficial].
   - **Xneg**: Check neighbor at $(x-1, y, z)$. If `IsAdjacentVoxelTransparent` returns true, set `Xneg` flag in `visibleFaces` [S3, unofficial].
   - **Xpos**: Check neighbor at $(x+1, y, z)$. If `IsAdjacentVoxelTransparent` returns true, set `Xpos` flag in `visibleFaces` [S3, unofficial].
   - **Ypos**: Check neighbor at $(x, y+1, z)$. If `IsAdjacentVoxelTransparent` returns true, set `Ypos` flag in `visibleFaces` [S3, unofficial].
   - **Yneg**: Check neighbor at $(x, y-1, z)$. If `IsAdjacentVoxelTransparent` returns true, set `Yneg` flag in `visibleFaces` [S3, unofficial].
7. Store `visibleFaces` in `visibilityMap[x, y, z]` [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_map(chunk_width, chunk_height, chunk_depth, get_voxel, is_neighbor_transparent):
    """
    Computes the visibility map for a voxel chunk.
    
    Args:
        chunk_width (int): Width of the chunk (X dimension).
        chunk_height (int): Height of the chunk (Y dimension).
        chunk_depth (int): Depth of the chunk (Z dimension).
        get_voxel (function): Returns the voxel at (x, y, z).
        is_neighbor_transparent (function): Returns True if the neighbor at (nx, ny, nz) is transparent.
        
    Returns:
        list: A 3D list of integers representing the VoxelFace bitmask for each voxel.
    """
    # Initialize visibility map with None (0)
    visibility_map = [[[0 for _ in range(chunk_depth)] for _ in range(chunk_height)] for _ in range(chunk_width)]
    
    # Define face flags
    FACE_NONE = 0
    FACE_XPOS = 1
    FACE_XNEG = 2
    FACE_YPOS = 4
    FACE_YNEG = 8
    FACE_ZPOS = 16
    FACE_ZNEG = 32
    
    # Iterate through every coordinate in ascending order
    for x in range(chunk_width):
        for y in range(chunk_height):
            for z in range(chunk_depth):
                voxel = get_voxel(x, y, z)
                
                # If voxel is not solid, no faces are visible
                if not voxel.is_solid:
                    visibility_map[x][y][z] = FACE_NONE
                    continue
                
                visible_faces = FACE_NONE
                
                # Check Z+ neighbor
                if is_neighbor_transparent(x, y, z + 1):
                    visible_faces |= FACE_ZPOS
                
                # Check Z- neighbor
                if is_neighbor_transparent(x, y, z - 1):
                    visible_faces |= FACE_ZNEG
                
                # Check X- neighbor
                if is_neighbor_transparent(x - 1, y, z):
                    visible_faces |= FACE_XNEG
                
                # Check X+ neighbor
                if is_neighbor_transparent(x + 1, y, z):
                    visible_faces |= FACE_XPOS
                
                # Check Y+ neighbor
                if is_neighbor_transparent(x, y + 1, z):
                    visible_faces |= FACE_YPOS
                
                # Check Y- neighbor
                if is_neighbor_transparent(x, y - 1, z):
                    visible_faces |= FACE_YNEG
                
                visibility_map[x][y][z] = visible_faces
                
    return visibility_map
```

**Parameters:**
- Chunk dimensions: Typically powers of two (e.g., 16x16x16) for efficient memory alignment and caching [S3, unofficial].
- Neighbor lookup: Out-of-bounds coordinates are treated as transparent [S3, unofficial].

**Edge Cases and Pitfalls:**
- Out-of-bounds neighbors: When checking adjacent voxels, if the neighbor coordinate is out of the chunk bounds, it is treated as transparent, making the face visible [S3, unofficial].
- Precision: Not applicable as the method is discrete.
- Degenerate input: Empty chunks or chunks with no solid voxels result in an all-zero visibility map [S3, unofficial].

## Subtopics

### [Voxel Face Occlusion Culling with Chunk Boundary Wrapping](13197-voxel-face-occlusion-culling-with-chunk-boundary-wrapping.md)
This method computes the occlusion status for all six faces of every voxel within a chunk, determining which faces are visible for mesh generation. It solves the problem of correctly identifying boundary conditions when a chunk is part of a larger, potentially toroidal or irregularly tiled world by wrapping coordinates to adjacent neighbor chunks. It differs from standard intra-chunk culling by explicitly handling cross-chunk neighbor lookups and coordinate wrapping, ensuring that faces on the chunk boundary are correctly marked as occluded or visible based on the state of neighboring chunks.

## Operating characteristics
- **Time Complexity**: $O(W \times H \times D)$, where $W, H, D$ are the chunk dimensions. Each voxel is processed once, and for each voxel, six neighbor checks are performed [S3, unofficial].
- **Memory Complexity**: $O(W \times H \times D)$ for the visibility map [S3, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Linear with respect to the number of voxels in the chunk [S3, unofficial].
- **Strengths**: Simple to implement, efficient for dense chunks, reduces mesh complexity significantly [S3, unofficial].
- **Limitations**: Requires full chunk access, does not handle inter-chunk dependencies without wrapping [S3, unofficial].
- **When to use**: When generating meshes for voxel chunks where face culling is required before mesh generation [S3, unofficial].
- **When not to use**: When the world is sparse and most voxels are empty, or when inter-chunk dependencies are complex and not handled by wrapping [S3, unofficial].

## Background
The method is part of the VoxelMeshOptimizerLibrary, a .NET 8 library for voxel mesh optimization [S3, unofficial]. It is used in the context of voxel rendering to cull non-visible faces, reducing the triangle count and improving rendering performance [S3, unofficial]. The library targets engine-agnostic usage and includes features like disjoint-set quad merging and OBJ export for inspection [S3, unofficial].

## Sources
[S1] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S3] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
