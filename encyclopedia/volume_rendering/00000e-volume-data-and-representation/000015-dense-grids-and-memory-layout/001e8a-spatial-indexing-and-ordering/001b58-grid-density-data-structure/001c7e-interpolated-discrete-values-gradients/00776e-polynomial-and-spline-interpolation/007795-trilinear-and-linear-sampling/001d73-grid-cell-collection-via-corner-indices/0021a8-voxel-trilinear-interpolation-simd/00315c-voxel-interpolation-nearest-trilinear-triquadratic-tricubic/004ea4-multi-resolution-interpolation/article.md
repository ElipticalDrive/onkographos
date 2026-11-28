# Multi-resolution Interpolation

Also called: Point-search interpolation, Variable resolution interpolation

## Overview
Multi-resolution interpolation is a technique for sampling volumetric data stored in hierarchical structures (such as Sparse Voxel Octrees) where adjacent blocks may have different resolutions. It solves the problem of interpolating values across block boundaries where simple trilinear interpolation fails due to non-uniform voxel spacing. It sets itself apart from standard [Voxel Interpolation](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md) by dynamically adjusting the sampling grid to the finest common resolution among neighbors, ensuring smooth transitions between Level-of-Detail (LoD) boundaries [S1].

## How it works
**Inputs:**
- A 3D sampling point $p_w$ in world coordinates.
- A target scale $l$ (resolution level).
- A hierarchical grid structure (e.g., SVO) where each voxel block stores its last updated scale.

**Outputs:**
- Interpolated value (e.g., opacity, radiance) at $p_w$.

**Data Structures:**
- Each voxel block in the hierarchy contains a metadata field `last_updated_scale` indicating the finest resolution at which data has been integrated.
- Voxel values are stored at integer coordinates $(i, j, k)$ relative to their block origin.

**Algorithm Steps:**
1. **Coordinate Conversion:** Convert the world sampling point $p_w$ to voxel coordinates $x^l_w$ at the target scale $l$.
   $$ x^l_w = \text{world_to_voxel}(p_w, l) $$
2. **Base Point Fetch:** Identify the base voxel at scale $l$ corresponding to $x^l_w$.
3. **Neighbor Search:** Fetch the 8 neighboring voxels required for trilinear interpolation at scale $l$.
4. **Resolution Constraint Check:** For each of the 8 neighbors, check if the neighbor's `last_updated_scale` is greater than the current scale $l$.
   - If `last_updated_scale` > $l$, the neighbor has finer data that has not yet been downsampled or synchronized to scale $l$. This violates the uniformity constraint.
5. **Coarse Search (Recursive):** If any neighbor violates the constraint:
   - Decrease the scale $l$ (move to a coarser resolution).
   - Repeat steps 1–4 at the new coarser scale.
   - This ensures interpolation occurs on the "common finest grid" shared by all 8 neighbors [S1].
6. **Interpolation:** Once all 8 neighbors satisfy the constraint (i.e., their data is available at scale $l$ or coarser), perform standard trilinear interpolation using the values at scale $l$ [S1].

**Initialization of Uninitialized Voxels:**
- Fine voxels that have not yet been initialized (e.g., during delta propagation) are handled by extrapolating values from parent grids at block boundaries. This avoids synchronization issues and ensures smooth initialization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_interpolate(p_w, grid, target_scale):
    """
    Interpolate value at p_w using multi-resolution point-search.
    """
    current_scale = target_scale
    
    while True:
        # Convert world point to voxel coordinates at current scale
        voxel_coords = world_to_voxel(p_w, current_scale)
        
        # Get the 8 neighbors in the grid at current_scale
        neighbors = get_8_neighbors(grid, voxel_coords, current_scale)
        
        # Check resolution constraint for all neighbors
        violation_found = False
        for neighbor in neighbors:
            if neighbor is None:
                # Handle boundary conditions or empty space
                continue
            if neighbor.last_updated_scale > current_scale:
                violation_found = True
                break
        
        if not violation_found:
            # All neighbors are valid at this scale
            values = [neighbor.value for neighbor in neighbors]
            return trilinear_interpolate(values, voxel_coords, current_scale)
        
        # If violation, search at coarser scale
        current_scale = current_scale - 1
        if current_scale < 0:
            raise ValueError("Scale too coarse to find valid neighbors")
```

## Subtopics

### [Plane-based Quadratic 4D Interpolation](23427-plane-based-quadratic-4d-interpolation.md)
A method to interpolate multi-resolution volumetric fields (opacity and radiance) stored in a Sparse Voxel Octree (SVO) using local plane samples (function value and spatial gradient) to achieve quadratic accuracy in 3D space and linear accuracy in Level-of-Detail (LoD) space.

## Operating characteristics
- **Time Complexity:** Worst-case complexity depends on the depth of the hierarchy and the frequency of resolution mismatches. In uniform regions, it is $O(1)$. At boundaries, it may require traversing up the hierarchy, but typically converges quickly [S1].
- **Memory Complexity:** Requires storing metadata (`last_updated_scale`) for each voxel block. No additional memory for interpolation coefficients compared to standard trilinear [S1].
- **Strengths:** Ensures smooth rendering at LoD boundaries without visible artifacts; avoids complex linear system solving required by other methods [S1].
- **Limitations:** Requires checking resolution constraints for all 8 neighbors; initialization of fine voxels requires careful extrapolation to avoid artifacts [S1].
- **When to use:** When rendering volumetric data with adaptive resolution (e.g., SLAM, dynamic scenes) where blocks are updated at different rates.
- **When not to use:** For static, uniform-resolution grids where standard trilinear interpolation is sufficient and faster.

## Background
Originated in the context of adaptive-resolution octree-based volumetric SLAM (Simultaneous Localization and Mapping). The method was introduced to handle the challenge of rendering volumetric data where the camera moves closer to surfaces, causing progressive smoothing and refinement of the octree structure. It addresses the issue that lastly integrated data may not have the same resolution as neighboring blocks, which would otherwise require expensive linear system solving for interpolation coefficients [S1].

## Sources
[S1] Adaptive-resolution octree-based volumetric SLAM, https://www.doc.ic.ac.uk/~sleutene/publications/Vespa_3DV19.pdf, fetched 2026-10-29
