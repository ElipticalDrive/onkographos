# Footprint Function Mechanism

Also called: 2D reconstruction kernel, Footprint Function, Splatting Dimension Reduction (Footprint), footprint table

## Overview
The Footprint Function Mechanism implements 3D spherically symmetric splatting kernels in volume rendering by reducing computational dimensionality from 3D to 2D [S1, unofficial]. It projects volumetric data onto a 2D image plane by pre-computing the integration of the 3D kernel along one axis and storing the result as a 2D lookup table [S1, unofficial]. This approach avoids per-pixel 3D kernel evaluations by relying on 2D table lookups and interpolation, as detailed in [Footprint Evaluation](1180-footprint-evaluation.md) [S1, unofficial].

## How it works
**Inputs:**
*   A 3D spherically symmetric splat kernel function $h(x, y, z)$ [S1, unofficial].
*   A regular volumetric grid with spacing parameters [G].
*   Camera projection parameters (orthographic or perspective) [G].

**Outputs:**
*   A 2D lookup table representing the footprint function [S1, unofficial].
*   A rendered image where each pixel contains the accumulated contribution of all splats [G].

**Data Structures:**
*   **Footprint Table:** A 2D array of floating-point values. The dimensions correspond to the spatial extent of the kernel in the image plane. The table is indexed by integer offsets $(i, j)$ from the center, representing discrete samples of the continuous function [S1, unofficial].
*   **Voxel Data:** Standard volumetric grid values (density/scalar) and positions [G].

**Formulas:**
The footprint function $F(x, y)$ is defined as the integral of the 3D kernel $h$ along one dimension (typically $w$ or $z$), assuming spherical symmetry:

$$
F(x, y) = \int_{-\infty}^{\infty} h(x, y, w) \, dw
$$

where $(x, y)$ is the displacement from the projected sample point on the image plane [S2, unofficial].

The contribution to a pixel at projected coordinates $(x_D, y_D)$ with density $\rho$ is:

$$
\text{Contribution}(x, y) = \rho \cdot F(x - x_D, y - y_D)
$$

[S2, unofficial].

**Steps:**
1.  **Pre-compute the Footprint Table:** Numerically integrate $h(x, y, w)$ along the $w$ axis for a grid of $(x, y)$ values covering the 2D extent, and store the results in a finely sampled 2D lookup table [S1, unofficial]. If the volume grid spacing is regular, the view projection is orthographic, and the splat kernel is radially symmetric, this table is computed once and reused [S1, unofficial]. Otherwise, it must be recomputed per view (for non-symmetric kernels) or per voxel (for perspective projections) [S1, unofficial].
2.  **Project Voxels:** For each voxel in the volume, calculate its projected center $(x_D, y_D)$ on the image plane using the current camera projection matrix [G].
3.  **Determine Extent:** Identify the bounding box of the footprint table that overlaps with the image plane region influenced by the voxel [S1, unofficial].
4.  **Sample and Composite:** For each pixel $(x_p, y_p)$ within the footprint's extent: calculate the displacement vector $(dx, dy) = (x_p - x_D, y_p - y_D)$; map $(dx, dy)$ to the indices of the pre-computed 2D footprint table; interpolate the value from the footprint table; retrieve the voxel's density $\rho$; compute the weighted contribution $\rho \cdot \text{interpolated\_footprint}(dx, dy)$; composite this value into the pixel's existing buffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute_footprint(kernel_func, extent, resolution):
    """Pre-compute 2D footprint table by integrating 3D kernel along w-axis."""
    table_size = 2 * resolution + 1
    footprint_table = [[0.0] * table_size for _ in range(table_size)]
    center = resolution
    dw = 2 * extent / resolution
    for i in range(table_size):
        for j in range(table_size):
            x = (i - center) * (extent / resolution)
            y = (j - center) * (extent / resolution)
            integral = 0.0
            for k in range(resolution):
                w = -extent + k * dw
                integral += kernel_func(x, y, w) * dw
            footprint_table[i][j] = integral
    return footprint_table, extent, resolution

def render_with_footprint(volume_data, footprint_table, extent, resolution, image_width, image_height):
    """Render volume using pre-computed footprint table."""
    image = [[0.0] * image_width for _ in range(image_height)]
    table_size = 2 * resolution + 1
    center = resolution
    
    for voxel in volume_data:
        # [G] Compute 2D image coordinates from 3D voxel position using camera matrix
        px, py = [G] compute_projection(voxel.position)
        
        min_x = max(0, int(px - extent))
        max_x = min(image_width, int(px + extent))
        min_y = max(0, int(py - extent))
        max_y = min(image_height, int(py + extent))
        
        for y in range(min_y, max_y):
            for x in range(min_x, max_x):
                dx = x - px
                dy = y - py
                idx_x = int(center + dx * (resolution / extent))
                idx_y = int(center + dy * (resolution / extent))
                
                if 0 <= idx_x < table_size and 0 <= idx_y < table_size:
                    foot_val = footprint_table[idx_y][idx_x]
                    contribution = voxel.density * foot_val
                    image[y][x] += contribution
    return image
```

**Parameters:**
*   **Footprint Table Resolution:** Not given in the sources. [G]
*   **Integration Steps:** Not given in the sources. [G]

**Edge Cases and Pitfalls:**
*   **Perspective Projection:** The footprint shape changes with depth. The table cannot be reused across the entire volume; it must be recomputed per voxel or approximated [S1, unofficial].
*   **Non-Symmetric Kernels:** If the kernel is not radially symmetric, the integration axis matters, and the table must be recomputed for each view orientation [S1, unofficial].
*   **Precision:** Numerical integration errors in pre-computation can lead to bias. Interpolation errors in the 2D table can cause blurring or artifacts [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Significant speedup over direct 3D kernel evaluation per pixel. Simple to implement with standard 2D interpolation routines [G].
*   **Limitations:** Requires pre-computation. Less flexible for non-symmetric or anisotropic kernels without significant overhead. Perspective projections require per-voxel table generation or approximation, which can negate performance gains [S1, unofficial].
*   **When to use:** Orthographic projections, regular grids, and symmetric kernels [S1, unofficial].
*   **When not to use:** Perspective projections with high accuracy requirements, or non-symmetric kernels, unless the per-voxel recomputation cost is acceptable [S1, unofficial].

## Background
The concept was introduced by Westover in the context of volume rendering, specifically to improve the efficiency of splatting algorithms [S1, unofficial]. The original splatting algorithm considered the splat as a 3D spherically symmetric reconstruction kernel [S1, unofficial]. Westover observed that since the splat is reconstructed into a 2D image raster, it can be implemented as a 2D reconstruction kernel by integrating the 3D kernel along one dimension [S1, unofficial]. This technique is often referred to as "footprint function" or "2D reconstruction kernel" [S1, unofficial]. Later work by Westover also introduced accumulating splats onto a 2D sheet aligned with the volume axis most parallel to the view plane to further improve quality [S1, unofficial].

## Sources
[S1] DTIC ADA363073: Perspective Splatting, https://archive.org/details/DTIC_ADA363073, fetched 2026-10-26 (unofficial)
[S2] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
