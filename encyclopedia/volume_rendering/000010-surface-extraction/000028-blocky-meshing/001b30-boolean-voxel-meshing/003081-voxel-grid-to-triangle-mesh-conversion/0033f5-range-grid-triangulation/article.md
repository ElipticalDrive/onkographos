# Range Grid Triangulation


## Overview
Range Grid Triangulation converts a 2D range grid, where each grid cell stores an index to a 3D vertex or an invalid marker, into a triangular mesh [S1, unofficial]. It differs from standard marching cubes by operating on pre-projected 2D grid data rather than 3D scalar fields, and handles invalid vertices by constructing triangles only from valid corners [S1, unofficial].

## How it works
**Inputs:**
- `grid`: A 1D array of integers. `grid[i]` is the index of the 3D vertex corresponding to grid cell `i`, or `GRID_INVALID` if the cell is empty/invalid [S1, unofficial].
- `remove_slivers`: Boolean flag (default `true`) indicating whether to remove degenerate 'sliver' triangles after triangulation [S1, unofficial].

**Outputs:**
- A list of triangles, where each triangle is defined by three 3D vertex indices.

**Data Structures:**
- `GRID_INVALID`: A constant integer representing an invalid vertex index (typically -1 or similar, implementation-dependent) [S1, unofficial].
- `vertices`: A list of 3D points (x, y, z) indexed by the integers stored in `grid` [S1, unofficial].

**Steps:**
1. **Count valid triangles:** Iterate over all grid quads. A quad is defined by four vertices: `ll` (lower-left), `lr` (lower-right), `ul` (upper-left), `ur` (upper-right). Count the number of valid vertices (`nvalid`) in each quad (where `grid[index] != GRID_INVALID`).
   - If `nvalid == 4`, add 2 to the total triangle count (`ntris`).
   - If `nvalid == 3`, add 1 to `ntris` [S1, unofficial].
2. **Create triangles for quads with 4 valid vertices:** Calculate squared Euclidean distances for both diagonals: `d1 = dist_sq(ll, ur)` and `d2 = dist_sq(lr, ul)`.
   - If `d1 < d2`, create triangles `(ll, lr, ur)` and `(ll, ur, ul)`.
   - Otherwise, create triangles `(ll, lr, ul)` and `(lr, ur, ul)` [S1, unofficial].
3. **Create triangles for quads with 3 valid vertices:** Identify the missing vertex (the one with `grid[index] < 0` or `== GRID_INVALID`).
   - If `ll` is missing, create triangle `(lr, ur, ul)`.
   - If `lr` is missing, create triangle `(ll, ur, ul)`.
   - If `ul` is missing, create triangle `(ll, lr, ur)`.
   - If `ur` is missing, create triangle `(ll, lr, ul)` [S1, unofficial].
4. **Remove slivers:** If `remove_slivers` is `true` and the face list is not empty, call `remove_sliver_faces` to clean up degenerate triangles [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def range_grid_triangulation(grid, vertices, remove_slivers=True):
    """
    Convert a 2D range grid to a triangular mesh.
    
    Args:
        grid: List[int], 1D array where grid[i] is vertex index or GRID_INVALID.
        vertices: List[Vec3], 3D points corresponding to vertex indices.
        remove_slivers: bool, whether to remove degenerate triangles.
        
    Returns:
        List[Tuple[int, int, int]], list of triangle vertex indices.
    """
    GRID_INVALID = -1  # Assumed constant based on context [S1, unofficial]
    triangles = []
    
    # Helper to check validity
    def is_valid(idx):
        return idx != GRID_INVALID

    # Helper to calculate squared distance
    def dist_sq(v1, v2):
        return (v1.x - v2.x)**2 + (v1.y - v2.y)**2 + (v1.z - v2.z)**2

    # Iterate over grid cells to form quads
    # Note: The exact mapping of 1D grid index to 2D quad coordinates depends on the grid layout.
    # Assuming a standard 2D grid of size W x H, where i = y * W + x.
    # A quad consists of cells (x, y), (x+1, y), (x, y+1), (x+1, y+1).
    # We iterate x from 0 to W-2 and y from 0 to H-2.
    
    # Determine grid dimensions from grid size
    # This assumes the grid represents a 2D plane. The source does not specify W and H explicitly.
    # We assume the caller provides W and H or they are derived from the context.
    # For the purpose of this algorithm description, we assume W and H are known.
    # If not, the implementation must handle the dimensionality.
    # [G] Assuming W and H are provided or derivable.
    
    # Let's assume the grid is a 2D array flattened to 1D.
    # We need W and H. If not given, we cannot iterate.
    # However, the source implies iterating over "all grid quads".
    # We will assume W and H are inputs or derived.
    
    # For the sake of complete pseudocode, we assume W and H are available.
    # If the grid is 1D, the mapping to 2D is required.
    
    # Let's assume the function signature includes W and H for clarity.
    # But the source notes only list 'grid'.
    # [G] We assume W and H are derived from the grid size or passed separately.
    # A common convention is that the grid represents a 2D slice.
    
    # To make the pseudocode self-contained, we assume W and H are inputs.
    # If the source does not provide them, the implementer must determine the grid topology.
    
    # Re-reading source: "Iterate over all grid quads (defined by ll, lr, ul, ur)."
    # This implies a 2D structure.
    
    # We will add W and H to the inputs for completeness.
    pass

def range_grid_triangulation_detailed(grid, vertices, W, H, remove_slivers=True):
    GRID_INVALID = -1
    triangles = []
    
    def is_valid(idx):
        return idx != GRID_INVALID

    def dist_sq(v1, v2):
        return (v1.x - v2.x)**2 + (v1.y - v2.y)**2 + (v1.z - v2.z)**2

    for y in range(H - 1):
        for x in range(W - 1):
            # Indices of the four corners of the quad
            ll_idx = y * W + x
            lr_idx = y * W + (x + 1)
            ul_idx = (y + 1) * W + x
            ur_idx = (y + 1) * W + (x + 1)
            
            ll_valid = is_valid(grid[ll_idx])
            lr_valid = is_valid(grid[lr_idx])
            ul_valid = is_valid(grid[ul_idx])
            ur_valid = is_valid(grid[ur_idx])
            
            nvalid = sum([ll_valid, lr_valid, ul_valid, ur_valid])
            
            if nvalid == 4:
                # Get vertex positions
                v_ll = vertices[grid[ll_idx]]
                v_lr = vertices[grid[lr_idx]]
                v_ul = vertices[grid[ul_idx]]
                v_ur = vertices[grid[ur_idx]]
                
                # Calculate squared distances for diagonals
                d1 = dist_sq(v_ll, v_ur)
                d2 = dist_sq(v_lr, v_ul)
                
                if d1 < d2:
                    triangles.append((grid[ll_idx], grid[lr_idx], grid[ur_idx]))
                    triangles.append((grid[ll_idx], grid[ur_idx], grid[ul_idx]))
                else:
                    triangles.append((grid[ll_idx], grid[lr_idx], grid[ul_idx]))
                    triangles.append((grid[lr_idx], grid[ur_idx], grid[ul_idx]))
                    
            elif nvalid == 3:
                # Identify missing vertex
                if not ll_valid:
                    triangles.append((grid[lr_idx], grid[ur_idx], grid[ul_idx]))
                elif not lr_valid:
                    triangles.append((grid[ll_idx], grid[ur_idx], grid[ul_idx]))
                elif not ul_valid:
                    triangles.append((grid[ll_idx], grid[lr_idx], grid[ur_idx]))
                elif not ur_valid:
                    triangles.append((grid[ll_idx], grid[lr_idx], grid[ul_idx]))
    
    if remove_slivers and triangles:
        triangles = remove_sliver_faces(triangles, vertices)
        
    return triangles

def remove_sliver_faces(triangles, vertices):
    """
    Remove degenerate triangles (slivers).
    A sliver is typically a triangle with zero or near-zero area.
    [G] Implementation detail not fully specified in source, but standard practice is to check area.
    """
    valid_triangles = []
    for t in triangles:
        v0 = vertices[t[0]]
        v1 = vertices[t[1]]
        v2 = vertices[t[2]]
        
        # Calculate area using cross product
        # Area = 0.5 * |(v1-v0) x (v2-v0)|
        cross = (v1 - v0).cross(v2 - v0)
        area_sq = cross.length_sq()
        
        # If area is non-zero, keep the triangle
        if area_sq > 1e-9: # Threshold for degeneracy
            valid_triangles.append(t)
            
    return valid_triangles
```

**Parameters:**
- `remove_slivers`: Boolean flag (default `true`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Invalid Vertices:** The algorithm explicitly handles quads with 3 valid vertices by creating a single triangle from the valid corners [S1, unofficial].
- **Sliver Triangles:** Degenerate triangles may be created during triangulation. The `remove_sliver_faces` step is used to clean these up [S1, unofficial].
- **Precision:** Squared distances are used to avoid square root operations, which can introduce precision issues if not handled carefully [S1, unofficial].

## Operating characteristics
**Time Complexity:** O(N), where N is the number of grid cells, as each quad is processed once [S1, unofficial].
**Memory Complexity:** O(N) for the output triangle list [S1, unofficial].
**Strengths:** Simple to implement, handles invalid data gracefully, minimizes sliver triangles by choosing optimal diagonals [S1, unofficial].
**Limitations:** Only works on 2D range grids, requires pre-computed vertex positions [S1, unofficial].

## Background
The method is described in the context of high-resolution sparse voxel DAGs [S1, unofficial]. It is part of a pipeline for converting voxel data into triangle meshes [S1, unofficial].

## Sources
[S1] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
