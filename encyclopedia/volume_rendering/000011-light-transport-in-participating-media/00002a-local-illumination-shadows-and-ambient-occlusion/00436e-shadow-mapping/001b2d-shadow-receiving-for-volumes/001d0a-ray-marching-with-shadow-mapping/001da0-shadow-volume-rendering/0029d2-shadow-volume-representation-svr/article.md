# Shadow Volume Representation (SVR)

Also called: GSV, Global Light Propagation, Global Shadow Volume, SVR (Shadow Volume Representation)

## Overview
Shadow Volume Representation (SVR) is a sparse data structure that stores pre-computed global opacity values along rays from volume points to a light source, enabling efficient estimation of direct light contribution in participating media [S1]. It distinguishes itself from full-resolution ray marching by decoupling the high-resolution integration of local piecewise opacities from the storage of global shadowing, which allows for sparse sampling density and interactive resolution adjustment [S1].

## How it works
**Inputs:**
*   Local piecewise opacities $\alpha_{l_p}$ for each voxel in the volume, computed from a pre-processing step [S1].
*   Light source position and direction [S1].
*   SVR resolution parameter (typically 1:4 relative to the data volume) [S1].

**Outputs:**
*   A sparse data structure containing global opacity $\alpha_g$ for sampled points, representing the accumulated shadowing from the light source [S1].

**Data Structures:**
*   **SVR Grid:** Sparse multi-resolution grid (e.g., flat blocking structure) storing global opacity values [S1]. Resolution is typically 1:4 relative to the data volume [S1].
*   **Local Opacity Storage:** Array storing local piecewise integration results for each point along the ray [S1]. Results must be stored for every point to allow global sampling to start at an arbitrary point [S1].

**Algorithm Steps:**
1.  **Local Piecewise Integration:** For each voxel in the volume, compute the local piecewise opacity $\alpha_{l_p}$ along the ray segment towards the light source [S1]. This result must be stored for every point to enable arbitrary start points for global sampling [S1].
2.  **Global Opacity Computation:** For each sample point $s_0$ in the SVR grid, compute the global opacity $\alpha_g(s_0)$ by integrating the pre-computed local opacities along the ray from $s_0$ to the light source [S1]. The global opacity is calculated using the product formula for transparency:
    $$
    \alpha_g(s_0) = 1 - \prod_{i=0}^k (1 - \alpha_{l_p}(s_i))
    $$
    where $\alpha_{l_p}(s_i)$ are the local piecewise opacities at sample points $s_i$ along the ray to the light source, and $k$ is the number of segments [S1].
3.  **Sparse Sampling:** Store the computed $\alpha_g(s_0)$ in the SVR grid [S1]. The SVR resolution is typically set to 1:4 relative to the data volume, meaning each sample in the SVR covers $4^3 = 64$ voxels in the data volume [S1].
4.  **Empty Block Handling:** If using a multi-resolution structure with empty blocks, advance the ray to the next non-empty block before computing $\alpha_g$ to avoid missing occlusion from neighboring non-empty blocks [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_svr(local_opacities, light_direction, svr_resolution, data_volume_shape, block_occupancy):
    """
    Computes global opacity for SVR grid.
    
    Args:
        local_opacities: 3D array of local piecewise opacities alpha_lp for each voxel.
        light_direction: Normalized vector pointing from volume to light source.
        svr_resolution: Tuple (res_x, res_y, res_z) for SVR grid.
        data_volume_shape: Tuple (vol_x, vol_y, vol_z) for data volume.
        block_occupancy: 3D array indicating which blocks are non-empty.
        
    Returns:
        svr_grid: Dictionary mapping (i, j, k) to global opacity alpha_g.
    """
    svr_grid = {}
    step_x = data_volume_shape[0] // svr_resolution[0]
    step_y = data_volume_shape[1] // svr_resolution[1]
    step_z = data_volume_shape[2] // svr_resolution[2]

    for i in range(svr_resolution[0]):
        for j in range(svr_resolution[1]):
            for k in range(svr_resolution[2]):
                s0_x, s0_y, s0_z = i * step_x, j * step_y, k * step_z

                # Handle empty blocks by advancing ray to next non-empty block
                if not block_occupancy[s0_x // step_x, s0_y // step_y, s0_z // step_z]:
                    adv_x, adv_y, adv_z = s0_x, s0_y, s0_z
                    while True:
                        # Standard grid traversal step [G]
                        adv_x += 1 if light_direction[0] > 0 else (-1 if light_direction[0] < 0 else 0)
                        adv_y += 1 if light_direction[1] > 0 else (-1 if light_direction[1] < 0 else 0)
                        adv_z += 1 if light_direction[2] > 0 else (-1 if light_direction[2] < 0 else 0)
                        if not (0 <= adv_x < data_volume_shape[0] and 0 <= adv_y < data_volume_shape[1] and 0 <= adv_z < data_volume_shape[2]):
                            adv_x, adv_y, adv_z = None, None, None
                            break
                        if block_occupancy[adv_x // step_x, adv_y // step_y, adv_z // step_z]:
                            break
                    if adv_x is None:
                        continue
                    s0_x, s0_y, s0_z = adv_x, adv_y, adv_z

                # Compute global opacity by integrating local opacities along ray to light
                alpha_trans = 1.0
                curr_x, curr_y, curr_z = s0_x, s0_y, s0_z
                max_steps = data_volume_shape[0] + data_volume_shape[1] + data_volume_shape[2]
                for _ in range(max_steps):
                    if not (0 <= curr_x < data_volume_shape[0] and 0 <= curr_y < data_volume_shape[1] and 0 <= curr_z < data_volume_shape[2]):
                        break
                    alpha_trans *= (1.0 - local_opacities[curr_x, curr_y, curr_z])
                    # Standard grid traversal step [G]
                    curr_x += 1 if light_direction[0] > 0 else (-1 if light_direction[0] < 0 else 0)
                    curr_y += 1 if light_direction[1] > 0 else (-1 if light_direction[1] < 0 else 0)
                    curr_z += 1 if light_direction[2] > 0 else (-1 if light_direction[2] < 0 else 0)

                svr_grid[(i, j, k)] = 1.0 - alpha_trans
    return svr_grid
```

**Parameters:**
*   **SVR Resolution:** The grid resolution of the Shadow Volume Representation [S1]. Typically 1:4 relative to the data volume, meaning each sample covers $4^3 = 64$ voxels [S1]. This resolution can be adjusted interactively [S1].

**Edge Cases and Pitfalls:**
*   **Empty Blocks:** In flat blocking multi-resolution structures, rays originating in empty blocks (ignored by the structure) may miss occlusion from neighboring non-empty blocks [S1]. Solution: Advance ray to next non-empty block before computing $\alpha_g$ [S1].
*   **Arbitrary Start Points:** The local piecewise integration must be performed and stored for each point along the ray to allow the global sampling to start at an arbitrary point [S1].
*   **Interpolation Errors:** Since linear interpolation is used to estimate local segments, some small errors are introduced in the global opacity calculation [S1].

## Operating characteristics
*   **Time Complexity:** The global integration is performed at high resolution to maintain quality, but storage and subsequent sampling are at lower SVR resolution [S1].
*   **Memory Complexity:** Reduced storage compared to full-resolution shadow volumes due to sparse representation and lower SVR resolution (e.g., 1:4) [S1].
*   **Strengths:** Allows interactive adjustment of SVR resolution; enables sparse sampling density while maintaining high-resolution ray integration quality [S1].
*   **Limitations:** Potential for missing occlusion if empty blocks are not handled correctly; small errors due to linear interpolation of local segments [S1].
*   **When to Use:** When interactive adjustment of shadow resolution is needed, or when storage/computation of full-resolution shadow volumes is prohibitive [S1].

## Background
SVR was introduced in the context of Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration [S1]. It is part of a method that includes local piecewise integration, global light integration, and estimations of in-scattering, which is incrementally refined [S1].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
