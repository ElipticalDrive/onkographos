# 3-axis rasterization voxelization

Also called: 6-separating voxelization of 2D input, CNS*11 method, FBP09 method

## Overview
This method produces a 6-separating voxelization of two-dimensional input geometry by rasterizing primitives along three axis-aligned projection directions in three separate passes [S1, unofficial]. It is designed for applications where ray marching follows 6-connected paths, allowing for a sparser voxel set than 26-separating methods while still guaranteeing that discretized rays do not penetrate separating surfaces [S1, unofficial]. It is distinguished from sibling methods like [Triangle Projection Voxelization](5432-triangle-projection-voxelization.md) by its reliance on point-depth evaluation at pixel centers rather than full geometric intersection tests [G], and from [Chunked UV Rasterization with Face ID Storage](2598-chunked-uv-rasterization-with-face-id-storage.md) by its specific 3-pass axis-aligned structure [G].

## How it works
**Inputs:** A set of two-dimensional input primitives (e.g., triangles) defined in three-dimensional space [S1, unofficial].
**Outputs:** A set of occupied voxels $S_d$ that form a 6-separating cover [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array or hash map storing voxel occupancy [G].
*   **Render Target:** A 2D buffer corresponding to the projection plane for each pass [G].

**Parameters:**
*   **Projection Axes:** The three axis-aligned directions: $X$, $Y$, and $Z$ [S1, unofficial].
*   **Voxel Size:** The physical dimension of a voxel [G].
*   **Pixel-to-Voxel Mapping:** The transformation from screen space to voxel grid coordinates [G].

**Steps:**
1.  Initialize the output voxel set $S_d$ as empty [G].
2.  For each of the three axes $A \in \{X, Y, Z\}$:
    a.  Project the input primitives onto the 2D plane perpendicular to axis $A$ [S1, unofficial].
    b.  Rasterize the projected primitives into a 2D render target [S1, unofficial].
    c.  For each pixel $p$ in the render target that is covered by the rasterization:
        i.  Determine the center of the voxel column corresponding to pixel $p$ along axis $A$ [G].
        ii. Evaluate the depth of the input primitive at the center $C_p$ [S1, unofficial].
        iii. Identify the voxel whose center is closest in depth to the evaluated primitive depth [S1, unofficial].
        iv. Add this voxel to $S_d$ [S1, unofficial].
3.  Return $S_d$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def three_axis_rasterization_voxelization(primitives, voxel_size, grid_origin, grid_extent):
    """
    primitives: List of geometric primitives.
    voxel_size: Scalar physical dimension of a voxel.
    grid_origin: 3D vector (x, y, z) of the grid's minimum corner.
    grid_extent: 3D vector (x, y, z) of the grid's maximum corner.
    Returns: Set of occupied voxel indices (ix, iy, iz).
    """
    occupied_voxels = set()
    axes = [0, 1, 2]  # X, Y, Z

    for axis in axes:
        proj_axes = [a for a in axes if a != axis]
        u_axis, v_axis = proj_axes[0], proj_axes[1]

        # The sources do not specify the exact rasterization traversal.
        for u_idx in range(grid_extent[u_axis] - grid_origin[u_axis]):
            for v_idx in range(grid_extent[v_axis] - grid_origin[v_axis]):
                # The sources do not specify the exact rasterization coverage test.
                if rasterize_coverage(primitives, u_idx, v_idx, u_axis, v_axis):
                    u_center = grid_origin[u_axis] + (u_idx + 0.5) * voxel_size
                    v_center = grid_origin[v_axis] + (v_idx + 0.5) * voxel_size

                    # The sources do not specify the exact depth evaluation routine.
                    depth_val = evaluate_primitive_depth(primitives, u_center, v_center, axis)

                    # Include voxel whose center is closest in depth to the evaluated primitive depth
                    col_z = int(round((depth_val - grid_origin[axis]) / voxel_size))
                    occupied_voxels.add((u_idx, v_idx, col_z))

    return occupied_voxels
```

**Edge Cases and Pitfalls:**
*   **Point Evaluation:** The method requires point evaluations of depth [S1, unofficial].
*   **6-Separation Guarantee:** The method produces 6-separating results even for curved primitives, a property that is difficult to ascertain without topological analysis [S1, unofficial].
*   **Symmetry:** The crosshairs target is a symmetrical realization of the general intersection target [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Strengths:** Produces a sparse voxel set compared to 26-separating methods [S1, unofficial]. Simple to implement on GPU using standard rasterization pipelines [G]. Works with curved primitives [S1, unofficial].
*   **Limitations:** Only 6-separating, not 26-separating [S1, unofficial]. Requires three passes [S1, unofficial]. Precision of depth evaluation can affect accuracy [G].
*   **When to Use:** When 6-connected ray marching is sufficient and sparsity is desired [S1, unofficial].
*   **When Not to Use:** When 26-connected paths are required or when higher separability is needed [S1, unofficial].

## Background
The method is equivalent to the GPU-based methods used by Forest et al. [FBP09] and Crassin et al. [CNS*11] [S1, unofficial]. The theoretical foundation for separability requirements is provided by Cohen-Or and Kaufman [COK97] [S1, unofficial]. The crosshairs intersection target is a minimal 6-separating target for two-dimensional input in three-dimensional space [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
