# Alternating Diagonal Crosshairs

Also called: Alternating diagonal target (2D), Corner-connected crosshairs

## Overview
The Alternating Diagonal Crosshairs method modifies the standard 4-separating crosshairs intersection target by pushing connecting segments to pixel corners, resulting in a single diagonal line segment per pixel [S1, unofficial]. It produces a 4-separating, 8-connected voxelization that forms a continuous 4-connected structure $C(\Pi)$ using only voxels in $\Pi$ [S1, unofficial]. Unlike the standard crosshairs target, it simplifies the intersection target to a single diagonal, trading aesthetic regularity for topological guarantees and sparsity [S1, unofficial].

## How it works
**Inputs:** A 2D input geometry and a target voxel grid resolution [S1, unofficial].
**Outputs:** A set of occupied voxels $\Pi$ representing the voxelization [S1, unofficial].
**Data Structures:** Each pixel $(i, j)$ is associated with an intersection target defined by a single diagonal line segment [S1, unofficial].

**Steps:**
1. Push the connecting segments of the crosshairs target to meet at pixel corners [S1, unofficial].
2. Form an intersection target consisting of a single diagonal line segment in each pixel [S1, unofficial].
3. Alternate between the two possible diagonals across the grid to maintain connectivity between voxels [S1, unofficial].
4. For each voxel, determine if the input geometry intersects its corresponding diagonal segment [S1, unofficial].
5. The resulting set of voxels $\Pi$ forms a continuous 4-connected structure $C(\Pi)$, ensuring 4-separability and 8-connectivity [S1, unofficial].

**Formulas:**
The diagonal segment $D_{i,j}$ for pixel $(i, j)$ is defined as:
$$D_{i,j} = \begin{cases} \{(i, j), (i+1, j+1)\} & \text{if } (i + j) \text{ is even} \\ \{(i+1, j), (i, j+1)\} & \text{if } (i + j) \text{ is odd} \end{cases}$$
where $(i, j)$ are integer pixel coordinates and the endpoints are in continuous pixel space [S1, unofficial].

**Pitfalls:**
The resulting voxelization is somewhat clunky, but it remains a subset of a cover of the input, never arbitrarily far from the input geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alternating_diagonal_crosshairs_2d(geometry, grid_width, grid_height):
    """
    Voxelize 2D geometry using alternating diagonal crosshairs.
    
    Args:
        geometry: Object with method intersects(p1, p2) -> bool
        grid_width: Width of the voxel grid in pixels
        grid_height: Height of the voxel grid in pixels
        
    Returns:
        Set of occupied voxel coordinates (i, j)
    """
    occupied_voxels = set()
    
    for j in range(grid_height):
        for i in range(grid_width):
            if (i + j) % 2 == 0:
                start = (i, j)
                end = (i + 1, j + 1)
            else:
                start = (i + 1, j)
                end = (i, j + 1)
            
            # Standard 2D line-segment vs. geometry intersection test [G]
            if geometry.intersects(start, end):
                occupied_voxels.add((i, j))
                
    return occupied_voxels
```

## Subtopics

### Alternating Space Diagonal Target (3D)
A modification of the 6-separating crosshairs target for 2D input in 3D [S1, unofficial]. It moves face connection points to voxel corners, merging six arms into two collinear segments, yielding a single space diagonal intersection target per voxel [S1, unofficial]. The four possible space diagonals are alternated in a pattern with a period of $2^3$ voxels to maintain connectivity between voxels [S1, unofficial]. The resulting voxelization is a 6-separating subset of a cover, exhibiting similar clunkiness to the 2D case [S1, unofficial]. It is the most efficient strategy when simplicity of the intersection target and small number of resulting voxels are prioritized over aesthetics, such as when the voxelization is not visualized directly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Produces a sparse voxelization with a small number of resulting voxels, which is beneficial when the voxelization is not visualized directly [S1, unofficial].
*   **Strengths:** Guarantees 4-separability and 8-connectivity in 2D, and 6-separability in 3D for 2D input [S1, unofficial]. It is a subset of a cover, ensuring topological fidelity [S1, unofficial].
*   **Limitations:** The voxelization is somewhat clunky and less regular than standard crosshairs [S1, unofficial].
*   **When to use:** When topological correctness and voxel count minimization are prioritized over aesthetic smoothness [S1, unofficial].

## Background
The method is part of a general voxelization scheme based on intersection targets, presented by Laine et al. in "A Topological Approach to Voxelization" (2013) [S1, unofficial]. It modifies the crosshairs target to maintain topological properties (connectivity and separability) while simplifying the intersection target structure [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
