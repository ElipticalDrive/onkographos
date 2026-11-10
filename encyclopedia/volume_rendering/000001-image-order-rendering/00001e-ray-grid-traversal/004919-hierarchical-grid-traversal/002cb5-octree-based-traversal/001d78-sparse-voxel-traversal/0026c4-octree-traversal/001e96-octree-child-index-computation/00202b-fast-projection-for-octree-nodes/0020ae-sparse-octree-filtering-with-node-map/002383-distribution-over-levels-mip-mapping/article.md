# Distribution over Levels (MIP-mapping)
Also called: Hierarchical Filtering, Octree MIP-map Generation, Octree MIP-map filtering, Three-pass MIP-mapping filtering

## Overview
Distribution over Levels is a hierarchical filtering technique for sparse voxel octrees that propagates radiance data from leaf nodes to interior nodes to create a multi-resolution MIP-map hierarchy. It solves the problem of aliasing and high-frequency noise in indirect illumination by smoothing voxel values across levels, enabling efficient cone tracing at coarser resolutions. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels [S1][S2]. Unlike [Sparse Octree Filtering with Node Map](2337-sparse-octree-filtering-with-node-map.md), which uses a 2D node map to reduce thread launch overhead, this method focuses on the computational balance of the filtering passes themselves to ensure uniform thread cost.

## How it works
**Inputs and Outputs:**
*   **Input:** Radiance values (energy and direction) stored in the leaf nodes of a sparse voxel octree. These values are typically injected via photon splatting from light sources.
*   **Output:** Filtered radiance values stored in the interior nodes (higher levels) of the octree, forming a MIP-map hierarchy.

**Data Structures:**
*   **Octree Node:** Contains a 3x3x3 brick of voxels. Each voxel stores a compact representation of incoming light.
*   **Gaussian-Lobes Representation:** A compact structure used to store the filtered distribution of incoming light directions, allowing for view-dependent treatment of the Normal Distribution Function (NDF) and Bidirectional Reflectance Distribution Function (BRDF) [S1].

**Algorithm Steps:**
The method avoids the inefficiency of launching one thread per higher-level voxel (which causes redundant computation for shared voxels and unbalanced scheduling) by using three separate passes with roughly equal thread costs [S1][S2].

1.  **Pass 1: Center Voxel.** Compute the filtered value for the center voxel of the 3x3x3 brick. This requires fetching and combining 27 voxel values from the lower level (the entire 3x3x3 neighborhood) [S1][S2].
2.  **Pass 2: Face Centers.** Compute half of the filtered response for the voxels situated in the center of the node's faces. Because only half the value is computed, this pass involves only 18 voxel values from the lower level [S1][S2].
3.  **Pass 3: Corners.** Launch threads for the corner voxels. These compute a partial filtering of voxels from a single octant [S1][S2].
4.  **Finalization (Value Transfer).** After the three passes, the higher-level voxels contain partial results, similar to the state of leaf nodes after initial photon splatting. The previously defined "Value Transfer to Neighboring Bricks" step is applied to sum values across bricks, finalizing the filtering result [S1][S2].

**Pseudocode:**
The following pseudocode illustrates the three-pass structure. It assumes a function `FilterBrick` that performs the weighted averaging of lower-level voxels.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distribute_radiance_over_levels(octree, level_index):
    """
    Filters radiance from level_index to level_index + 1.
    Assumes octree nodes are 3x3x3 bricks.
    """
    # Identify all bricks at level_index + 1 that need filtering
    bricks_to_filter = get_bricks_at_level(octree, level_index + 1)
    
    # Pass 1: Center Voxel
    # Computes full filter using 27 lower-level voxels
    for brick in bricks_to_filter:
        center_voxel = brick.voxels[1, 1, 1]
        # Fetch 27 voxels from lower level (3x3x3 neighborhood)
        lower_voxels = fetch_3x3x3_neighbors(octree, brick, level_index)
        center_voxel.value = filter_27_voxels(lower_voxels)
        
    # Pass 2: Face Centers
    # Computes half response using 18 lower-level voxels
    # Faces are at indices: (1,0,1), (1,2,1), (0,1,1), (2,1,1), (1,1,0), (1,1,2)
    face_indices = [
        (1, 0, 1), (1, 2, 1),
        (0, 1, 1), (2, 1, 1),
        (1, 1, 0), (1, 1, 2)
    ]
    for brick in bricks_to_filter:
        for idx in face_indices:
            face_voxel = brick.voxels[idx]
            # Fetch 18 voxels from lower level (half-neighborhood)
            lower_voxels = fetch_half_neighbor_voxels(octree, brick, level_index, idx)
            face_voxel.value = 0.5 * filter_18_voxels(lower_voxels)
            
    # Pass 3: Corners
    # Computes partial filtering from a single octant
    # Corners are at indices: (0,0,0), (0,0,2), (0,2,0), (0,2,2), (2,0,0), (2,0,2), (2,2,0), (2,2,2)
    corner_indices = [
        (0, 0, 0), (0, 0, 2), (0, 2, 0), (0, 2, 2),
        (2, 0, 0), (2, 0, 2), (2, 2, 0), (2, 2, 2)
    ]
    for brick in bricks_to_filter:
        for idx in corner_indices:
            corner_voxel = brick.voxels[idx]
            # Fetch voxels from single octant
            lower_voxels = fetch_octant_voxels(octree, brick, level_index, idx)
            corner_voxel.value = partial_filter_octant(lower_voxels)
            
    # Finalization: Apply Value Transfer to Neighboring Bricks
    # This step sums partial results across adjacent bricks to get the final value
    apply_value_transfer(octree, level_index + 1)

def fetch_3x3x3_neighbors(octree, brick, level):
    # Implementation detail: Gather 3x3x3 voxels from the lower level
    # surrounding the current brick's position.
    # Returns a list of 27 voxel values.
    pass

def fetch_half_neighbor_voxels(octree, brick, level, face_idx):
    # Implementation detail: Gather 18 voxels from the lower level
    # corresponding to the half-filtering requirement for face centers.
    pass

def fetch_octant_voxels(octree, brick, level, corner_idx):
    # Implementation detail: Gather voxels from the single octant
    # corresponding to the corner's position.
    pass

def filter_27_voxels(voxels):
    # Weighted average of 27 voxels
    pass

def filter_18_voxels(voxels):
    # Weighted average of 18 voxels
    pass

def partial_filter_octant(voxels):
    # Partial filtering from a single octant
    pass

def apply_value_transfer(octree, level):
    # Sum values across neighboring bricks to finalize the MIP-map level
    pass
```

**Parameters and Typical Values:**
*   **Octree Depth:** Determined by the scene's detail and memory constraints.
*   **Gaussian-Lobes:** The number of lobes determines the accuracy of the directional light distribution. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparse Octrees:** The method assumes a complete octree in its basic form. In sparse octrees, launching threads for all nodes (including empty ones) consumes resources. Filtering zero values in empty nodes is inefficient. A solution is to launch a thread per light-view-map pixel to find the relevant node and execute filtering only where photons exist [S2].
*   **Shared Voxels:** A naive approach of launching one thread per higher-level voxel causes shared voxels to be computed up to eight times, leading to redundant work and unbalanced scheduling due to varying computation costs [S1][S2]. The three-pass method mitigates this by splitting the computation.

## Subtopics

### [Octree MIP-mapping](4127-octree-mip-mapping.md)
Octree MIP-mapping is the process of propagating and filtering voxel data from the leaf nodes of a sparse voxel octree up to the interior nodes, creating a multi-resolution hierarchy for efficient ray-grid traversal. It solves the problem of representing high-frequency surface details at fine levels while providing smooth, low-frequency approximations at coarse levels, preventing aliasing during rendering. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels.

### Gaussian-Lobes Representation for Radiance Filtering
A compact representation used to store the filtered distribution of incoming light directions in the octree mipmap levels, allowing for view-dependent treatment of NDF and BRDF. It is filtered into higher octree levels (mipmap) using a screen-space quad-tree analysis for parallel efficiency [S1].

## Operating characteristics
*   **Time Complexity:** The three-pass approach ensures that all threads have roughly the same cost, leading to balanced scheduling on GPU hardware. However, the exact complexity depends on the number of non-empty bricks. In sparse octrees, avoiding filtering of zero values is crucial for performance [S2].
*   **Memory Complexity:** Requires storing radiance values (energy and direction) in all levels of the octree, including the MIP-map levels. The Gaussian-Lobes representation adds overhead per voxel but allows for efficient storage of directional information [S1].
*   **Strengths:** Efficient parallel filtering with balanced thread costs. Handles sparse data effectively when combined with light-view-map based thread launching. Prevents aliasing by smoothing high-frequency details.
*   **Limitations:** Computationally intensive if the octree is dense. Requires careful handling of boundary conditions and shared voxels. The "Value Transfer" step adds an additional pass over the data.
*   **When to use:** When rendering indirect illumination with cone tracing on sparse voxel octrees, especially when view-dependent lighting effects are desired.
*   **When not to use:** For very dense scenes where the overhead of maintaining the MIP-map hierarchy outweighs the benefits, or when real-time performance is critical and the GPU cannot handle the three-pass filtering efficiently.

## Background
Originating from the work on Interactive Indirect Illumination Using Voxel Cone Tracing, this method was introduced to efficiently handle indirect lighting in dynamic scenes. It builds on the concept of sparse voxel octrees and cone tracing [S1]. The method was developed to address the inefficiencies of naive MIP-map generation in sparse structures, such as redundant computation and unbalanced scheduling. It is used in real-time rendering engines that support global illumination effects, particularly those involving dynamic lights and complex geometry.

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
