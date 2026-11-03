# Per-Region Gradient Averaging


## Overview
Per-Region Gradient Averaging computes the 3D gradient vector for each voxel in a scalar volume by averaging directional gradients across multiple discrete scalar regions, using a binary neighbor check to detect region boundaries [S1, unofficial]. It builds on [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) but replaces continuous interpolation with discrete region membership checks [G].

## How it works
**Inputs:** A 3D volume grid of scalar values, a list of scalar regions (integers or identifiers), and the volume dimensions $(D_x, D_y, D_z)$ [G].
**Outputs:** A 3D grid of gradient vectors $(g_x, g_y, g_z)$ and their magnitudes [S1, unofficial].

**Data Structures:**
*   `self.data`: A flat list or array of `VoxelGradient` objects of size $D_x \times D_y \times D_z$ [S1, unofficial].
*   `VoxelGradient`: A structure containing `prev_gx`, `prev_gy`, `prev_gz` (floats) and `magnitude` (computed) [S1, unofficial].

**Algorithm Steps:**
1. Initialize `self.data` with `VoxelGradient` objects for all voxels [S1, unofficial].
2. Replace the original volume data with a boolean mask or integer label map indicating region membership for each voxel [S1, unofficial].
3. For each voxel $(x, y, z)$ in the volume:
   a. Initialize `prev_gx = 0`, `prev_gy = 0`, `prev_gz = 0`, and `n = 0` [S1, unofficial].
   b. For each region $R$ in the list of regions:
      i. Compute the directional gradient components $(g_x, g_y, g_z)$ for region $R$ at $(x, y, z)$:
         $$g_x = \frac{x_{plus} - x_{minus}}{2}$$
         $$g_y = \frac{y_{plus} - y_{minus}}{2}$$
         $$g_z = \frac{z_{plus} - z_{minus}}{2}$$
         where $x_{plus} = 1000$ if $V(x+1, y, z) \in R$ else $0$, $x_{minus} = 1000$ if $V(x-1, y, z) \in R$ else $0$, and similarly for $y$ and $z$ [S1, unofficial]. The value $1000$ is a binary weight assigned to neighbors belonging to the current region to amplify boundary detection [S1, unofficial].
      ii. If $(g_x, g_y, g_z)$ is non-zero:
          $$new\_gx = \frac{prev\_gx \times n + g_x}{n + 1}$$
          $$new\_gy = \frac{prev\_gy \times n + g_y}{n + 1}$$
          $$new\_gz = \frac{prev\_gz \times n + g_z}{n + 1}$$
          Update `prev_gx`, `prev_gy`, `prev_gz` to `new_gx`, `new_gy`, `new_gz` respectively, and increment `n` by 1 [S1, unofficial].
   c. Set the voxel's gradient to $(prev\_gx, prev\_gy, prev\_gz)$ [S1, unofficial].
   d. Compute the magnitude: $\sqrt{prev\_gx^2 + prev\_gy^2 + prev\_gz^2}$ [S1, unofficial].
4. Compute the maximum gradient magnitude lazily on the first call to `get_max_gradient_magnitude` by scanning `self.data` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_per_region_gradients(volume_data, regions, dim_x, dim_y, dim_z):
    # Data structure initialization
    class VoxelGradient:
        def __init__(self):
            self.prev_gx = 0.0
            self.prev_gy = 0.0
            self.prev_gz = 0.0
            self.magnitude = 0.0

    voxels = [VoxelGradient() for _ in range(dim_x * dim_y * dim_z)]

    # Replace volume data with boolean mask/region labels (assumed prepared externally)
    # region_map[x, y, z] stores the region ID for each voxel
    region_map = volume_data  # [G] Assumed to be a 3D array of region IDs

    for x in range(dim_x):
        for y in range(dim_y):
            for z in range(dim_z):
                prev_gx = 0.0
                prev_gy = 0.0
                prev_gz = 0.0
                n = 0

                for region in regions:
                    # Boundary handling: clamp coordinates or return 0 for out-of-bounds
                    def get_voxel(cx, cy, cz):
                        if 0 <= cx < dim_x and 0 <= cy < dim_y and 0 <= cz < dim_z:
                            return region_map[cx, cy, cz]
                        return 0  # [G] Assumed boundary handling

                    val_plus_x = get_voxel(x + 1, y, z)
                    val_minus_x = get_voxel(x - 1, y, z)
                    val_plus_y = get_voxel(x, y + 1, z)
                    val_minus_y = get_voxel(x, y - 1, z)
                    val_plus_z = get_voxel(x, y, z + 1)
                    val_minus_z = get_voxel(x, y, z - 1)

                    weight = 1000.0
                    gx = ((weight if val_plus_x == region else 0.0) - (weight if val_minus_x == region else 0.0)) / 2.0
                    gy = ((weight if val_plus_y == region else 0.0) - (weight if val_minus_y == region else 0.0)) / 2.0
                    gz = ((weight if val_plus_z == region else 0.0) - (weight if val_minus_z == region else 0.0)) / 2.0

                    if gx != 0 or gy != 0 or gz != 0:
                        new_gx = (prev_gx * n + gx) / (n + 1)
                        new_gy = (prev_gy * n + gy) / (n + 1)
                        new_gz = (prev_gz * n + gz) / (n + 1)
                        prev_gx = new_gx
                        prev_gy = new_gy
                        prev_gz = new_gz
                        n += 1

                idx = x * dim_y * dim_z + y * dim_z + z
                voxels[idx].prev_gx = prev_gx
                voxels[idx].prev_gy = prev_gy
                voxels[idx].prev_gz = prev_gz
                voxels[idx].magnitude = (prev_gx**2 + prev_gy**2 + prev_gz**2)**0.5

    return voxels
```

## Subtopics

### Total Variation Gradient Addition
Calculates a weight $w = \text{weight} \times \max(\text{world\_size}) / 128$ and applies a total variation gradient to the density, k0, or diffuse field using `total_variation_add_grad(w, w, w, dense_mode)` [S2, unofficial]. This variant is used in differentiable volume rendering contexts to regularize gradients [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(D_x D_y D_z \times N_r)$, where $N_r$ is the number of regions [G]. Each voxel iterates over all regions and checks 6 neighbors [S1, unofficial].
*   **Memory Complexity:** $O(D_x D_y D_z)$ for storing the gradient vectors [G].
*   **Strengths:** Effective for volumes with distinct, discrete scalar regions (e.g., segmented medical images, procedural volumes with sharp boundaries) [G]. Amplifies boundary gradients significantly [S1, unofficial].
*   **Limitations:** Computationally expensive for large numbers of regions [G]. Sensitive to the choice of the binary weight (1000) [S1, unofficial]. Not suitable for smooth, continuous scalar fields where standard finite differences are more appropriate [G].
*   **When to use:** When the volume data is naturally segmented into discrete regions and sharp boundary detection is prioritized over smooth gradient estimation [G].
*   **When not to use:** For continuous scalar fields or when computational efficiency is critical with many regions [G].

## Background
Originated in the context of raycasting rendering implementations for volumes with discrete scalar regions [S1, unofficial]. The method is part of a broader effort to improve gradient-based lighting and shading in volume rendering by leveraging region-aware boundary information [G].

## Sources
[S1] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S2] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
