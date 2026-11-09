# Alternating Diagonal Crosshairs

Also called: Alternating diagonal target (2D), Corner-connected crosshairs

## Overview
The Alternating Diagonal Crosshairs method is a voxelization technique that modifies the standard crosshairs intersection target by pushing connecting segments to pixel corners, resulting in a single diagonal line segment per pixel [S1, unofficial]. It produces a 4-separating, 8-connected voxelization that is a subset of a cover of the input geometry, ensuring it remains bounded relative to the input [S1, unofficial]. Unlike the standard crosshairs target which uses multiple segments, this method simplifies the intersection target to a single diagonal, trading aesthetic regularity for topological guarantees and sparsity [S1, unofficial].

## How it works
**Inputs:** A 2D input geometry (e.g., a polygon or implicit surface) and a target voxel grid resolution.
**Outputs:** A set of occupied voxels $\Pi$ representing the voxelization.

**Data Structures:**
The method operates on a 2D grid. Each pixel $(i, j)$ is associated with an intersection target defined by a single diagonal line segment. The diagonal is chosen from two possibilities: the main diagonal (bottom-left to top-right) or the anti-diagonal (top-left to bottom-right).

**Steps:**
1.  **Initialize Grid:** Create a 2D boolean array or set to store occupied voxels.
2.  **Determine Diagonal Pattern:** Assign a diagonal orientation to each pixel $(i, j)$ based on its coordinates to ensure connectivity. The pattern must alternate to maintain 4-connectivity of the connection graph $C(\Pi)$. A common pattern is determined by the parity of $(i + j)$:
    *   If $(i + j)$ is even, use the main diagonal.
    *   If $(i + j)$ is odd, use the anti-diagonal.
    *   *Note:* The source mentions a period of $2^2$ (4) for the 2D case implicitly through the alternating nature required for connectivity [S1, unofficial].
3.  **Define Intersection Target:** For each pixel $(i, j)$, the intersection target $\tau_{i,j}$ is the line segment connecting the two corners defined by the chosen diagonal.
    *   Main diagonal: Connects $(i, j)$ to $(i+1, j+1)$ in pixel coordinates.
    *   Anti-diagonal: Connects $(i+1, j)$ to $(i, j+1)$ in pixel coordinates.
4.  **Voxelization:** For each voxel in the grid, determine if it intersects the input geometry. A voxel is occupied if the input geometry intersects its corresponding diagonal segment $\tau_{i,j}$ [S1, unofficial].
    *   *Implementation Detail:* Since the target is a line segment, the intersection test is a 2D line-segment vs. geometry intersection test.
5.  **Connectivity Verification:** The resulting set of voxels $\Pi$ forms a continuous 4-connected structure $C(\Pi)$ using only voxels in $\Pi$, ensuring 4-separability and 8-connectivity [S1, unofficial].

**Pitfalls:**
*   The resulting voxelization is "clunky" due to the sparse, diagonal-only structure [S1, unofficial].
*   It is a subset of a cover, so it never deviates arbitrarily far from the input geometry, but it may appear less smooth than other methods [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alternating_diagonal_crosshairs_2d(geometry, grid_width, grid_height):
    """
    Voxelize 2D geometry using alternating diagonal crosshairs.
    
    Args:
        geometry: Object with method intersects(line_segment) -> bool
        grid_width: Width of the voxel grid in pixels
        grid_height: Height of the voxel grid in pixels
        
    Returns:
        Set of occupied voxel coordinates (i, j)
    """
    occupied_voxels = set()
    
    for j in range(grid_height):
        for i in range(grid_width):
            # Determine diagonal orientation based on parity of (i + j)
            if (i + j) % 2 == 0:
                # Main diagonal: bottom-left to top-right
                # In pixel coordinates, assuming (0,0) is top-left:
                # Start: (i, j+1), End: (i+1, j) -> Wait, standard image coords:
                # Let's use Cartesian-like for clarity: (x,y) where x=i, y=j
                # Main diagonal: (i, j) to (i+1, j+1)
                start_point = (i, j)
                end_point = (i + 1, j + 1)
            else:
                # Anti-diagonal: top-left to bottom-right
                # Start: (i+1, j), End: (i, j+1)
                start_point = (i + 1, j)
                end_point = (i, j + 1)
            
            line_segment = LineSegment(start_point, end_point)
            
            # Check if the geometry intersects the diagonal segment
            if geometry.intersects(line_segment):
                occupied_voxels.add((i, j))
                
    return occupied_voxels
```

## Subtopics

### Alternating Space Diagonal Target (3D)
A modification of the 6-separating crosshairs target for 2D input in 3D [S1, unofficial]. It moves face connection points to voxel corners, merging six arms into two collinear segments, yielding a single space diagonal intersection target per voxel [S1, unofficial]. The four possible space diagonals are alternated in a pattern with a period of $2^3$ voxels to maintain connectivity [S1, unofficial]. The resulting voxelization is a 6-separating subset of a cover, exhibiting similar clunkiness to the 2D case [S1, unofficial]. It is the most efficient strategy when simplicity of the intersection target and small number of resulting voxels are prioritized over aesthetics, such as when the voxelization is not visualized directly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the grid, $O(N)$, where $N$ is the total number of pixels/voxels, assuming constant-time intersection tests.
*   **Memory Complexity:** Linear with respect to the number of voxels, $O(N)$, to store the occupied voxel set.
*   **Performance:** Produces a sparse voxelization with fewer voxels than standard crosshairs, which can be beneficial for memory and processing if the voxelization is not visualized directly [S1, unofficial].
*   **Strengths:** Guarantees 4-separability and 8-connectivity in 2D, and 6-separability in 3D (for 2D input). It is a subset of a cover, ensuring topological fidelity.
*   **Limitations:** The voxelization is "clunky" and less regular, which may be undesirable for visualization purposes [S1, unofficial].
*   **When to use:** When topological correctness (separability/connectivity) is critical and aesthetic smoothness is secondary, or when minimizing voxel count is a priority.

## Background
The method is part of a general voxelization scheme based on intersection targets, presented by Laine et al. in "A Topological Approach to Voxelization" (2013) [S1, unofficial]. It is a modification of the crosshairs target, designed to maintain topological properties (connectivity and separability) while simplifying the intersection target structure. The approach is rooted in the need for robust, topology-preserving voxelization for applications like volume rendering and geometric modeling.

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
