# 3-axis rasterization voxelization

Also called: 6-separating voxelization of 2D input, CNS*11 method, FBP09 method

## Overview
This method produces a 6-separating voxelization of two-dimensional input geometry by rasterizing primitives along three axis-aligned projection directions in three separate passes [S1, unofficial]. It is designed for applications where ray marching follows 6-connected paths, allowing for a sparser voxel set than 26-separating methods while still guaranteeing that discretized rays do not penetrate separating surfaces [S1, unofficial]. It is distinguished from sibling methods like [Triangle Projection Voxelization](5432-triangle-projection-voxelization.md) by its reliance on point-depth evaluation at pixel centers rather than full geometric intersection tests, and from [Chunked UV Rasterization with Face ID Storage](2598-chunked-uv-rasterization-with-face-id-storage.md) by its specific 3-pass axis-aligned structure.

## How it works
**Inputs:** A set of 2D input primitives (e.g., triangles) defined in 3D space.
**Outputs:** A set of occupied voxels $S_d$ that form a 6-separating cover.

**Data Structures:**
*   **Voxel Grid:** A 3D array or hash map storing voxel occupancy. The resolution is defined by the target voxel size.
*   **Render Target:** A 2D buffer (texture or image) corresponding to the projection plane for each pass.

**Parameters:**
*   **Projection Axes:** The three axis-aligned directions: $X$, $Y$, and $Z$.
*   **Voxel Size:** The physical dimension of a voxel.
*   **Pixel-to-Voxel Mapping:** The transformation from screen space (rasterization output) to voxel grid coordinates.

**Steps:**
1.  Initialize the output voxel set $S_d$ as empty.
2.  For each of the three axes $A \in \{X, Y, Z\}$:
    a.  Project the input primitives onto the 2D plane perpendicular to axis $A$.
    b.  Rasterize the projected primitives into a 2D render target. The rasterization must cover the area of the primitives.
    c.  For each pixel $p$ in the render target that is covered by the rasterization:
        i.  Determine the center of the voxel column corresponding to pixel $p$ along axis $A$. Let this center be $C_p$.
        ii. Evaluate the depth of the input primitive at the center $C_p$. This is a point evaluation: find the $z$-coordinate (or coordinate along axis $A$) of the primitive surface at the $(x, y)$ location of the pixel center.
        iii. Identify the voxel whose center is closest in depth to the evaluated primitive depth.
        iv. Add this voxel to $S_d$.
3.  Return $S_d$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def three_axis_rasterization_voxelization(primitives, voxel_grid, axes=[0, 1, 2]):
    """
    primitives: List of geometric primitives (e.g., triangles)
    voxel_grid: 3D data structure to store occupied voxels
    axes: List of 3 axis indices (0=X, 1=Y, 2=Z)
    """
    occupied_voxels = set()
    
    for axis in axes:
        # Determine the projection plane axes
        other_axes = [a for a in axes if a != axis]
        
        # Project primitives to 2D and rasterize
        # This step is handled by the graphics pipeline
        # We assume a function that returns the depth at pixel centers
        # for all pixels covered by the rasterized primitives.
        pixel_depths = rasterize_and_get_depths(primitives, other_axes[0], other_axes[1], axis)
        
        for pixel, depth_at_center in pixel_depths.items():
            # Map pixel to voxel column index
            voxel_col_index = pixel_to_voxel_column(pixel, other_axes[0], other_axes[1])
            
            # Find the voxel in the column closest to the evaluated depth
            # The voxel column is a line of voxels along the current axis
            closest_voxel = find_closest_voxel_in_column(voxel_col_index, depth_at_center, axis)
            
            occupied_voxels.add(closest_voxel)
            
    return occupied_voxels

def rasterize_and_get_depths(primitives, u_axis, v_axis, depth_axis):
    """
    Simulates rasterization and returns depth at pixel centers.
    In practice, this is done by the GPU.
    """
    # This is a conceptual representation.
    # The actual implementation relies on the GPU rasterizer.
    # For each pixel covered by the primitive, we evaluate the primitive's
    # equation at the pixel's center coordinates in the u-v plane.
    pass

def find_closest_voxel_in_column(column_index, depth, axis):
    """
    Finds the voxel in the given column whose center is closest to the depth.
    """
    # Calculate the voxel index along the axis
    # voxel_index = floor(depth / voxel_size)
    # The exact mapping depends on the coordinate system and origin.
    pass
```

**Edge Cases and Pitfalls:**
*   **Point Evaluation:** The method requires point evaluations of depth. If the primitive is curved, the depth at the pixel center must be accurately computed [S1, unofficial].
*   **6-Separation Guarantee:** The method produces 6-separating results even for curved primitives, a property that is difficult to ascertain without topological analysis [S1, unofficial].
*   **Symmetry:** The crosshairs target is a symmetrical realization of the general intersection target. Non-symmetrical targets may also work but are less standard [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of primitives and $P$ is the number of pixels in the render target. The constant factor is 3 due to the three passes.
*   **Memory Complexity:** $O(V)$, where $V$ is the number of voxels in the grid. The render target size is $O(P)$.
*   **Strengths:** Produces a sparse voxel set compared to 26-separating methods. Simple to implement on GPU using standard rasterization pipelines. Works with curved primitives.
*   **Limitations:** Only 6-separating, not 26-separating. Requires three passes. Precision of depth evaluation can affect accuracy.
*   **When to Use:** When 6-connected ray marching is sufficient and sparsity is desired.
*   **When Not to Use:** When 26-connected paths are required or when higher separability is needed.

## Background
The method is based on the topological analysis of voxelization by Laine et al. [S1, unofficial]. It is equivalent to the GPU-based methods used by Forest et al. [FBP09] and Crassin et al. [CNS*11]. The theoretical foundation is provided by Cohen-Or and Kaufman [COK97], who analyzed the connectivity requirements for voxelization. The crosshairs intersection target is a minimal 6-separating target for 2D input in 3D.

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
