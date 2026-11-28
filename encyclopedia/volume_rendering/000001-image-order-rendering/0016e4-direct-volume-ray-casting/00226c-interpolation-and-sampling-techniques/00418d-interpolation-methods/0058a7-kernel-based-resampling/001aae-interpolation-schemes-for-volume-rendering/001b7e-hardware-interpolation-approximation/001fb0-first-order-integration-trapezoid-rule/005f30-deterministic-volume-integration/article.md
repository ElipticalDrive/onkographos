# Deterministic Volume Integration

Also called: Interval Integration, Trilinear Integration

## Overview
Deterministic Volume Integration is a ray-marching technique that improves upon standard point-sampling by integrating the trilinearly interpolated feature fields over the entire length of a ray segment within a voxel, rather than evaluating them at a single point [S1, S2]. This method addresses the error introduced by assuming density and color fields are constant between samples, providing a more accurate estimation of the volume integral compared to Monte Carlo approaches [S2, S3]. It is distinct from [Second Order Pre-Integrated Volume Rendering](3388-second-order-pre-integrated-volume-rendering.md) which handles quadratic variations, and [Analytic Integration for Linear Variation](5146-analytic-integration-for-linear-variation.md) which may use different basis functions or constraints [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a voxel grid containing learned feature fields (density $\sigma$ and intermediate features $\mathbf{h}$); a color decoder function $\theta_{color}$; and a positional encoding function $\gamma$ for the viewing direction [S1, S2].
*   **Output:** The accumulated color $\mathbf{C}$ and opacity $\alpha$ for the pixel, derived from the integrated density and features [S2].

**Data Structures:**
The method operates on a voxel grid where each voxel stores feature vectors [S1]. In the specific implementation described in the sources, the scene is divided into $N=8$ symmetric regions (cubes) sharing a single feature grid to reduce memory usage by a factor of $N$ [S1]. Each voxel's features are accessed via trilinear interpolation [S2]. A data layout strategy places features symmetrically to the scene center to favor continuity between density fields and reduce the number of unique features needed [S1].

**Steps:**
1.  **Ray-Voxel Intersection:** Determine the entry point $t_j^{in}$ and exit point $t_j^{out}$ for each voxel $j$ intersected by the ray [S2].
2.  **Feature Integration:** For each interval $[t_j^{in}, t_j^{out}]$, compute the integrated density $\sigma_j$ and integrated intermediate features $\mathbf{h}_j$ by integrating the trilinearly interpolated fields over the interval [S1, S2]. The integration is defined as:
    $$
    (\sigma_j, \mathbf{h}_j) = \int_{t_j^{in}}^{t_j^{out}} \hat{f}_i(\mathbf{x}(t), \mathbf{d}) \, dt, \quad i \in [1, 8]
    $$
    where $\hat{f}_i$ represents the trilinearly interpolated feature field for region $i$, and $\mathbf{x}(t)$ is the position on the ray at distance $t$ [S1].
3.  **Color Decoding:** Decode the color $\mathbf{c}_j$ for the voxel interval using the integrated intermediate features $\mathbf{h}_j$, the viewing direction $\mathbf{d}_j$, and the color decoder $\theta_{color}$ [S1, S2]:
    $$
    \mathbf{c}_j = \theta_{color}(\mathbf{h}_j, \gamma(\mathbf{d}_j))
    $$
    where $\gamma(\mathbf{d}_j)$ is the positional encoding of the viewing direction [S1].
4.  **Compositing:** Composite the results $(\sigma_j, \mathbf{h}_j, \mathbf{c}_j)$ along the ray using standard volume rendering accumulation equations [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trilinear_interpolate(grid, x, y, z):
    """Standard trilinear interpolation over a 3D grid."""
    # Clamp coordinates to grid bounds [0, W-1], [0, H-1], [0, D-1]
    ix = max(0, min(len(grid)-1, int(x)))
    iy = max(0, min(len(grid[0])-1, int(y)))
    iz = max(0, min(len(grid[0][0])-1, int(z)))
    return grid[ix][iy][iz]

def integrate_trilinear_field(grid, origin, direction, t_in, t_out):
    """Integrates the trilinearly interpolated field over [t_in, t_out] using the trapezoid rule."""
    x_in = origin + direction * t_in
    x_out = origin + direction * t_out
    sigma_in, h_in = trilinear_interpolate(grid, x_in)
    sigma_out, h_out = trilinear_interpolate(grid, x_out)
    dt = t_out - t_in
    # Trapezoid rule: average of endpoints multiplied by interval length
    sigma_j = 0.5 * (sigma_in + sigma_out) * dt
    h_j = 0.5 * (h_in + h_out) * dt
    return sigma_j, h_j

def deterministic_volume_integration(ray_origin, ray_direction, voxel_grid, color_decoder, positional_encoding):
    """
    ray_origin: 3D vector
    ray_direction: 3D vector (normalized)
    voxel_grid: 3D array of (density, features) tuples
    color_decoder: Function taking (features, direction_encoding) -> color
    positional_encoding: Function taking direction -> encoded vector
    """
    # 1. Compute ray-voxel intervals (standard slab traversal)
    intervals = compute_ray_voxel_intervals(ray_origin, ray_direction, voxel_grid.bounds)
    
    accumulated_color = [0.0, 0.0, 0.0]
    transmittance = 1.0
    
    for t_in, t_out in intervals:
        # 2. Integrate features over the interval
        sigma_j, h_j = integrate_trilinear_field(voxel_grid, ray_origin, ray_direction, t_in, t_out)
        
        # 3. Decode color
        direction_enc = positional_encoding(ray_direction)
        c_j = color_decoder(h_j, direction_enc)
        
        # 4. Accumulate
        # sigma_j is already the integrated density over [t_in, t_out]
        alpha = 1.0 - math.exp(-sigma_j)
        accumulated_color += alpha * transmittance * c_j
        transmittance *= (1.0 - alpha)
        
        # Early ray termination
        if transmittance < 1e-4:
            break
            
    return accumulated_color

def compute_ray_voxel_intervals(origin, direction, bounds):
    """Standard slab traversal to find t_min and t_max for each intersected voxel."""
    # [G] Standard implementation computes t_max on each axis, takes min/max,
    # and steps through voxels until exiting bounds. Returns list of (t_in, t_out).
    pass
```

**Parameters:**
*   $N=8$: Number of symmetric regions in the voxel grid configuration [S1].
*   Region size: Cube with side length equal to half the bounding box side [S1].
*   Decoder layers: Density decoders have an input layer, a hidden layer of 32 neurons, and an output layer of 33 neurons; the color decoder has an input layer of 32 neurons and an output layer of 3 neurons [S1]. All layers use ReLU activation except output layers, which have no activation for density [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The integration of trilinear fields is more computationally expensive than point sampling. Numerical integration errors may occur if the interval is large relative to the voxel size and low-order quadrature is used [G].
*   **Boundaries:** The symmetric configuration is designed to ensure continuity between regions at boundaries, reducing artifacts [S1].
*   **Transmittance Culling:** Early ray termination should use a small epsilon threshold to avoid division by zero or infinite loops [G].

## Subtopics

### Symmetric Voxel Grid Configuration
A data layout strategy where features are placed symmetrically to the scene center. This favors continuity between density fields and feature pruning, reducing the number of unique features needed by a factor of $N$ [S1]. Choose this when memory efficiency and inter-region continuity are priorities. Link: [NeRFLight](https://doi.org/10.1109/cvpr52729.2023.01195).

## Operating characteristics
*   **Time Complexity:** Higher than standard point-sampling trilinear interpolation because it requires integrating the feature field over the voxel interval rather than evaluating it at a single point [S2]. The cost depends on the integration method (analytic vs. numerical quadrature) [G].
*   **Memory Complexity:** Reduced by a factor of $N$ (where $N=8$ in the cited implementation) due to the shared feature grid across symmetric regions [S1].
*   **Performance:** Produces better estimation than Monte Carlo sampling because it accounts for the variation of density and color fields within the voxel interval [S2, S3].
*   **Strengths:** Improved visual quality over point-sampling; memory efficient via shared grids [S1, S2].
*   **Limitations:** Increased computational cost per voxel compared to point sampling [G].
*   **When to use:** When high-quality volume rendering is required and memory is constrained, particularly in neural radiance field contexts like NeRFLight [S1, S2].
*   **When not to use:** When real-time performance is critical and point-sampling quality is sufficient [G].

## Background
The method was introduced in the context of NeRFLight, a fast and light Neural Radiance Fields approach using a shared feature grid [S1, S2]. The technique was developed to improve the quality of the model by replacing random sampling with deterministic integration of trilinearly interpolated features [S2, S3]. Not given in the sources: exact publication venue and authors beyond the paper title.

## Sources
[S1] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://doi.org/10.1109/cvpr52729.2023.01195, fetched 2026-10-26
[S2] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://www.iri.upc.edu/files/scidoc/2861-NeRFLight%3A-Fast-and-Light-Neural-Radiance-Fields-using-a-Shared-Feature-Grid.pdf, fetched 2026-10-27
[S3] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://openaccess.thecvf.com/content/CVPR2023/html/Rivas-Manzaneque_NeRFLight_Fast_and_Light_Neural_Radiance_Fields_Using_a_Shared_CVPR_2023_paper.html, fetched 2026-10-27
