# Sparse Marching Cubes

Also called: Sparse Marching Cubes from Points

## Overview
Sparse Marching Cubes generates a triangle mesh from a sparse set of 3D points with associated scalar values (e.g., Signed Distance Fields), avoiding the memory overhead of dense 3D grids [S1, unofficial]. It differs from standard Marching Cubes by dynamically gathering corner values for active voxels via lookup, rather than iterating over a fixed dense volume [S1, unofficial].

## How it works
**Inputs:**
1.  `coords`: Integer tensor of shape $[N, 3]$ containing the $(x, y, z)$ coordinates of active voxels or points [S1, unofficial].
2.  `corners` OR `point_values`:
    *   Option A: `corners` Float tensor of shape $[N, 8]$ containing pre-computed scalar values at the 8 corners of each voxel [S1, unofficial].
    *   Option B: `point_values` Float tensor of shape $[N]$ or $[N, 1]$ containing scalar values at the coordinates in `coords` [S1, unofficial].
3.  `iso`: Float value representing the iso-level to contour at [S1, unofficial].
4.  `default_value`: Float value assumed for corner points not present in the input `coords` (used only in Option B). Typically $1.0$ (outside) [S1, unofficial].
5.  `ensure_consistency`: Boolean flag; if `True`, averages corner values at shared locations to ensure topological consistency [S1, unofficial].

**Outputs:**
1.  `verts`: Float tensor of shape $[V, 3]$ with vertex positions [S1, unofficial].
2.  `tris` (or `faces`): Int tensor of shape $[T, 3]$ with triangle indices [S1, unofficial].

**Algorithm:**

1.  **Voxel Identification**: Identify the set of active voxels. If input is `coords` and `point_values`, these coordinates define the centers of the active voxels. If input is `coords` and `corners`, these define the active voxels directly [S1, unofficial].

2.  **Corner Value Gathering**: For each active voxel, determine the scalar values at its 8 corners.
    *   If `corners` are provided, use them directly [S1, unofficial].
    *   If `point_values` are provided, look up the scalar value for each of the 8 corner coordinates in the input set. If a corner coordinate is not in the input set, assign `default_value` [S1, unofficial].

3.  **Consistency Check (Optional)**: If `ensure_consistency` is `True`, identify shared corners between adjacent active voxels. Average the scalar values at these shared locations to ensure that neighboring voxels agree on the field value at the shared vertex [S1, unofficial].

4.  **Case Index Calculation**: For each voxel, compute the marching cubes case index (0-255) based on the 8 corner values relative to `iso`.
    Let $v_i$ be the scalar value at corner $i$ ($i=0..7$).
    $$ b_i = \begin{cases} 1 & \text{if } v_i < iso \\ 0 & \text{otherwise} \end{cases} $$
    $$ \text{index} = \sum_{i=0}^{7} b_i 2^i $$
    [S1, unofficial]

5.  **Mesh Generation**: For each voxel, use the standard Marching Cubes lookup table to generate triangles based on the `index`. Vertices are interpolated along edges where the sign changes (or crosses `iso`) [S1, unofficial].

6.  **Vertex Merging**: Merge duplicate vertices that result from shared edges/faces between adjacent voxels to form a coherent mesh [S1, unofficial].

**Pseudocode (Python-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_marching_cubes(coords, corners, iso, ensure_consistency=False):
    """
    Performs sparse marching cubes on a set of voxels with pre-computed corners.
    
    Args:
        coords: Tensor [N, 3], integer coordinates of voxels.
        corners: Tensor [N, 8], scalar values at the 8 corners of each voxel.
        iso: Float, iso-level.
        ensure_consistency: Bool, average shared corner values.
        
    Returns:
        verts: Tensor [V, 3], vertex positions.
        tris: Tensor [T, 3], triangle indices.
    """
    # 1. Ensure consistency of shared corners
    if ensure_consistency:
        # Identify shared corners between adjacent voxels
        # Average their values in 'corners' tensor
        corners = average_shared_corners(coords, corners)
        
    # 2. Calculate case indices
    # v < iso gives boolean mask [N, 8]
    is_below = corners < iso
    # Convert to integer bits
    bits = is_below.int()
    # Compute index: bit 0 is corner 0, bit 1 is corner 1, etc.
    powers = torch.tensor([1, 2, 4, 8, 16, 32, 64, 128], dtype=torch.int32, device=corners.device)
    cube_indices = torch.sum(bits * powers, dim=1)
    
    # 3. Generate triangles
    # Use standard Marching Cubes logic for each cube_index
    # Interpolate vertices on edges
    # Return merged verts and tris
    verts, tris = generate_mesh_from_indices(coords, corners, cube_indices, iso)
    
    return verts, tris

def sparse_marching_cubes_from_points(coords, point_values, iso, default_value=1.0):
    """
    Performs sparse marching cubes from a sparse set of points.
    
    Args:
        coords: Tensor [N, 3], integer coordinates of points.
        point_values: Tensor [N], scalar values at coords.
        iso: Float, iso-level.
        default_value: Float, value for missing corners.
        
    Returns:
        verts: Tensor [V, 3], vertex positions.
        faces: Tensor [T, 3], triangle indices.
    """
    # 1. Build a lookup map from coordinate to value
    # Map (x,y,z) -> value
    lookup = build_lookup_map(coords, point_values)
    
    # 2. For each point (voxel center), gather 8 corner values
    # Corner offsets for a voxel at (x,y,z) are typically (0,0,0) to (1,1,1)
    # relative to the voxel grid. Assuming coords are voxel centers or integer grid points.
    # Note: The source implies coords are the points with defined values.
    # We need to determine the 8 corners of the voxel associated with each point.
    # If coords are voxel centers, corners are centers +/- 0.5.
    # If coords are integer grid points, they might be corners themselves.
    # Based on "Sparse Marching Cubes from Points" description, it gathers 8 corner values.
    
    N = coords.shape[0]
    corners = torch.zeros((N, 8), dtype=torch.float32, device=coords.device)
    
    for i in range(N):
        center = coords[i]
        # Define 8 corner offsets relative to the voxel
        # Assuming the voxel is defined by the integer coordinate as a corner or center.
        # Standard MC assumes voxel corners are integer coordinates.
        # If 'coords' are the points with values, and we treat them as voxel centers,
        # we need to look up values at center +/- 0.5.
        # However, the source says "coords... coordinates of active voxels".
        # Let's assume coords are the integer coordinates of the voxel's minimum corner (0,0,0) relative to voxel.
        # Then corners are (x,y,z) + (dx,dy,dz) for dx,dy,dz in {0,1}.
        
        base = center
        for j in range(8):
            dx = (j >> 0) & 1
            dy = (j >> 1) & 1
            dz = (j >> 2) & 1
            corner_coord = base + torch.tensor([dx, dy, dz], dtype=torch.int32)
            
            if corner_coord in lookup:
                corners[i, j] = lookup[corner_coord]
            else:
                corners[i, j] = default_value
                
    # 3. Call standard sparse MC
    return sparse_marching_cubes(coords, corners, iso, ensure_consistency=False)
```

## Subtopics

### Sparse Marching Cubes from SRF
Extracts 3D meshes from Sparse Radiance Fields (SRF) by performing one pass of Marching Cubes on the 3D voxel space, as models like SPARF and SuRFNet operate within this space [S2].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of active voxels $N$, assuming constant-time lookup for corner values.
*   **Memory Complexity**: Proportional to the number of active voxels and their corners, significantly less than dense grid methods which scale with the volume resolution $R^3$.
*   **Performance**: Accelerated via CUDA, processing thousands of voxels simultaneously [S1, unofficial].
*   **Strengths**: Ideal for sparse SDF representations where only a list of points and values are available; avoids dense grid memory overhead [S1, unofficial].
*   **Limitations**: Requires careful handling of `default_value` for missing corners to avoid artifacts; consistency flag adds overhead for shared corner averaging [S1, unofficial].

## Background
Sparse Marching Cubes is implemented in libraries such as `spcumc` (Sparse Marching Cubes) to handle sparse voxel data efficiently on GPUs [S1, unofficial]. It is used in contexts involving sparse SDFs and is applicable to extracting meshes from Sparse Radiance Fields (SRF) like those generated by SPARF and SuRFNet [S2].

## Sources
[S1] hwanhuh/spcumc, https://github.com/hwanhuh/spcumc, fetched 2026-10-27 (unofficial)
[S2] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
