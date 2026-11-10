# Distribution over Levels (MIP-mapping)
Also called: Hierarchical Filtering, Octree MIP-map Generation, Octree MIP-map filtering, Three-pass MIP-mapping filtering

## Overview
Distribution over Levels (MIP-mapping) filters radiance values from the lowest level of a sparse voxel octree and stores the results in higher levels to form a MIP-map hierarchy [S1]. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels [S1][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Radiance values (energy and direction) injected into the leaf nodes of the sparse voxel octree via photon splatting from light sources [S1].
*   **Output:** Filtered radiance values stored in the interior nodes (higher levels) of the octree, forming a MIP-map hierarchy [S1].

**Data Structures:**
*   **Octree Node:** Contains a 3x3x3 brick of voxels [S1].
*   **Voxel Representation:** Each voxel stores a compact Gaussian-Lobes representation of incoming light directions, enabling view-dependent treatment of the Normal Distribution Function (NDF) and Bidirectional Reflectance Distribution Function (BRDF) [S1].

**Algorithm Steps:**
The method avoids the inefficiency of launching one thread per higher-level voxel, which causes shared voxels to be computed up to eight times and leads to unbalanced scheduling due to varying computation costs [S1][S2]. Instead, it uses three separate passes with roughly equal thread costs [S1][S2].
1.  **Pass 1: Center Voxel.** Compute the filtered value for the center voxel of the 3x3x3 brick using 27 voxel values from the lower level [S1][S2].
2.  **Pass 2: Face Centers.** Compute half of the filtered response for the voxels situated in the center of the node's faces, involving 18 voxel values from the lower level [S1][S2].
3.  **Pass 3: Corners.** Launch threads for the corner voxels, which compute a partial filtering of voxels from a single octant [S1][S2].
4.  **Finalization (Value Transfer).** After the three passes, the higher-level voxels contain partial results similar to leaf nodes after initial photon splatting [S1][S2]. The previously defined "Value Transfer to Neighboring Bricks" step is applied to sum values across bricks and finalize the filtering result [S1][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distribute_radiance_over_levels(octree, level_index):
    # Identify all bricks at level_index + 1 that need filtering
    bricks_to_filter = [brick for brick in octree[level_index + 1].bricks if brick.is_active()]

    # Pass 1: Center Voxel
    for brick in bricks_to_filter:
        center_voxel = brick.voxels[1, 1, 1]
        lower_voxels = gather_3x3x3_neighbors(octree, brick, level_index)
        # Sources do not specify the exact weighting formula [G]
        center_voxel.value = sum(v.value for v in lower_voxels) / len(lower_voxels)

    # Pass 2: Face Centers
    face_indices = [(1, 0, 1), (1, 2, 1), (0, 1, 1), (2, 1, 1), (1, 1, 0), (1, 1, 2)]
    for brick in bricks_to_filter:
        for idx in face_indices:
            face_voxel = brick.voxels[idx]
            lower_voxels = gather_half_neighbors(octree, brick, level_index, idx)
            face_voxel.value = 0.5 * sum(v.value for v in lower_voxels) / len(lower_voxels)

    # Pass 3: Corners
    corner_indices = [(0, 0, 0), (0, 0, 2), (0, 2, 0), (0, 2, 2), (2, 0, 0), (2, 0, 2), (2, 2, 0), (2, 2, 2)]
    for brick in bricks_to_filter:
        for idx in corner_indices:
            corner_voxel = brick.voxels[idx]
            lower_voxels = gather_octant_neighbors(octree, brick, level_index, idx)
            corner_voxel.value = sum(v.value for v in lower_voxels) / len(lower_voxels)

    # Finalization: Apply Value Transfer to Neighboring Bricks
    for brick in octree[level_index + 1].bricks:
        brick.finalize()

def gather_3x3x3_neighbors(octree, brick, level):
    # Gather 3x3x3 voxels from the lower level surrounding the current brick
    neighbors = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                neighbors.append(octree[level].get_voxel_at(brick.global_pos + (dx, dy, dz)))
    return neighbors

def gather_half_neighbors(octree, brick, level, face_idx):
    # Gather 18 voxels from the lower level corresponding to the half-filtering requirement
    neighbors = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                # Exclude the 9 voxels on the opposite face and center line to get 18
                if not ((dx == 0 and dy == 0) or (dx == 0 and dz == 0) or (dy == 0 and dz == 0)):
                    neighbors.append(octree[level].get_voxel_at(brick.global_pos + (dx, dy, dz)))
    return neighbors

def gather_octant_neighbors(octree, brick, level, corner_idx):
    # Gather voxels from the single octant corresponding to the corner's position
    neighbors = []
    offset_x = -1 if corner_idx[0] == 0 else 1
    offset_y = -1 if corner_idx[1] == 0 else 1
    offset_z = -1 if corner_idx[2] == 0 else 1
    neighbors.append(octree[level].get_voxel_at(brick.global_pos + (offset_x, offset_y, offset_z)))
    return neighbors
```

**Parameters and Typical Values:**
*   **Octree Depth:** Not given in the sources.
*   **Gaussian-Lobes Count:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparse Octrees:** The basic method assumes a complete octree, but sparse octrees waste resources filtering zero values in empty nodes [S2]. Launching a thread per light-view-map pixel to find the relevant node avoids filtering empty nodes [S2].
*   **Shared Voxels:** A naive approach of launching one thread per higher-level voxel causes shared voxels to be computed up to eight times, leading to redundant work and unbalanced scheduling [S1][S2]. The three-pass method mitigates this by splitting the computation [S1][S2].

## Subtopics

### [Octree MIP-mapping](4127-octree-mip-mapping.md)
Octree MIP-mapping propagates and filters voxel data from leaf nodes to interior nodes, creating a multi-resolution hierarchy for efficient ray-grid traversal. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels.

### Gaussian-Lobes Representation for Radiance Filtering
A compact representation used to store the filtered distribution of incoming light directions in the octree mipmap levels, allowing for view-dependent treatment of NDF and BRDF [S1]. It is filtered into higher octree levels (mipmap) using a screen-space quad-tree analysis for parallel efficiency [S1].

## Operating characteristics
*   **Time Complexity:** The three-pass approach ensures that all threads have roughly the same cost, leading to balanced scheduling [S1][S2]. Performance depends on the number of non-empty bricks, and avoiding filtering of zero values is crucial for sparse octrees [S2].
*   **Memory Complexity:** Requires storing radiance values (energy and direction) in all levels of the octree, including the MIP-map levels [S1]. The Gaussian-Lobes representation adds overhead per voxel but allows for efficient storage of directional information [S1].
*   **Strengths:** Efficient parallel filtering with balanced thread costs. Handles sparse data effectively when combined with light-view-map based thread launching.
*   **Limitations:** Computationally intensive if the octree is dense. Requires careful handling of boundary conditions and shared voxels. The "Value Transfer" step adds an additional pass over the data.
*   **When to use:** When rendering indirect illumination with cone tracing on sparse voxel octrees, especially when view-dependent lighting effects are desired.
*   **When not to use:** For very dense scenes where the overhead of maintaining the MIP-map hierarchy outweighs the benefits, or when real-time performance is critical and the GPU cannot handle the three-pass filtering efficiently.

## Background
Originating from the work on Interactive Indirect Illumination Using Voxel Cone Tracing, this method was introduced to efficiently handle indirect lighting in dynamic scenes [S1]. It builds on the concept of sparse voxel octrees and cone tracing [S1]. The method was developed to address the inefficiencies of naive MIP-map generation in sparse structures, such as redundant computation and unbalanced scheduling [S1]. It is used in real-time rendering engines that support global illumination effects, particularly those involving dynamic lights and complex geometry [S1].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
