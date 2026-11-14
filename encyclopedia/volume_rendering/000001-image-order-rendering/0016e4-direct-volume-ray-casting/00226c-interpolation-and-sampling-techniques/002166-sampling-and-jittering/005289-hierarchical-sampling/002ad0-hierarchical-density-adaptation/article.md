# Hierarchical Density Adaptation

Also called: On-the-fly coarse sampling, Point Selection Criteria

## Overview
Hierarchical Density Adaptation is a sampling strategy for volume rendering that controls rendering density using a hierarchy of sampled data to maintain constant frame rates [S1]. It serves as an alternative to opacity-based adaptive sampling, trading image smoothness for rendering speed by controlling the total number of rendered primitives per frame [S1]. It distinguishes itself from standard hierarchical sampling by generating the coarse density estimate on-the-fly from the dense grid via convolution, rather than relying on a separate, pre-allocated coarse network or grid [S2].

## How it works
The method integrates with a hierarchical sampling framework where a coarse estimate predicts rough density, and a fine estimate predicts precise color using additional samples placed where coarse density was high [Parent]. The core mechanism generates the coarse density estimate on-the-fly to avoid memory overhead and ensure surface regions are not skipped [S2].

**Inputs and Outputs:**
*   **Inputs:** Dense density grid $G_\sigma$, ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, maximum ray distance $D_{\text{max}}$, number of coarse samples $N_c$, number of fine samples $N_f$ [S2].
*   **Outputs:** Pixel color $\hat{C}$ [S2].

**Data Structures:**
*   **Dense Density Grid ($G_\sigma$):** A 3D grid storing volume density $\sigma$ [S2]. In EgoNeRF, this is a balanced spherical feature grid [S2].
*   **Convolution Kernel ($K$):** A non-learnable average pooling kernel used to smooth the dense grid [S2].

**Steps:**

1.  **Obtain Coarse Density Estimate:**
    Apply a non-learnable convolution kernel $K$ (average pooling) to the dense density grid $G_\sigma$ to create a coarser estimate on the fly. This prevents skipping important surface regions that might occur if the dense grid were sampled directly at coarse intervals [S2].
    $$ \sigma(\mathbf{x}_{\text{coarse}}) = \mathcal{T}(K * G_\sigma, \mathbf{x}_{\text{coarse}}) $$
    where $\mathcal{T}$ denotes trilinear interpolation [S2].

2.  **Sample Coarse Points:**
    Sample $N_c$ points along the ray. In standard implementations, these are distributed uniformly [S2]. However, in variants with exponentially increasing grid resolution (e.g., spherical grids), they are distributed exponentially [S2].

3.  **Calculate Importance Sampling Weights:**
    Compute weights for the coarse samples based on their contribution to the ray integral. The weight $w_i$ for the $i$-th coarse sample is:
    $$ w_i = \tau_i (1 - e^{-\sigma_i \delta_i}) $$
    where:
    *   $\sigma_i$ is the coarse density at sample $i$ [S2].
    *   $\delta_i$ is the distance between coarse samples $i$ and $i+1$ (or the step size) [S2].
    *   $\tau_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j}$ is the accumulated transmittance up to sample $i$ [S2].

4.  **Sample Fine Points:**
    Sample $N_f$ fine points from the probability distribution defined by the normalized weights $w_i$. This is importance sampling, concentrating samples in regions of high density [Parent, S2].

5.  **Render:**
    Evaluate the fine network (or feature grid) at all $N_c + N_f$ sample locations to obtain density $\sigma$ and color $c$. Accumulate the color using the volume rendering equation [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def hierarchical_density_adaptation(ray_origin, ray_direction, dense_grid, max_dist, N_c, N_f, kernel_size=3):
    # 1. On-the-fly coarse density via average pooling [G]
    coarse_grid = average_pool_3d(dense_grid, kernel_size)

    # 2. Sample N_c coarse points uniformly along ray [0, max_dist] [S2]
    coarse_ts = [i * max_dist / N_c for i in range(N_c)]
    coarse_points = [(ray_origin[0] + t * ray_direction[0],
                      ray_origin[1] + t * ray_direction[1],
                      ray_origin[2] + t * ray_direction[2]) for t in coarse_ts]

    # 3. Evaluate coarse density at these points [G]
    coarse_densities = [trilinear_interpolate(coarse_grid, p) for p in coarse_points]

    # 4. Calculate weights for importance sampling [S2]
    delta = max_dist / N_c
    transmittances = [0.0] * N_c
    transmittances[0] = 1.0
    for i in range(1, N_c):
        transmittances[i] = transmittances[i-1] * math.exp(-coarse_densities[i-1] * delta)

    weights = [transmittances[i] * (1.0 - math.exp(-coarse_densities[i] * delta)) for i in range(N_c)]

    # Normalize weights to probability distribution
    w_sum = sum(weights)
    if w_sum == 0:
        prob_dist = [1.0 / N_c] * N_c
    else:
        prob_dist = [w / w_sum for w in weights]

    # 5. Sample N_f fine points from the probability distribution (Inverse CDF) [S2]
    cdf = [0.0] * N_c
    cdf[0] = prob_dist[0]
    for i in range(1, N_c):
        cdf[i] = cdf[i-1] + prob_dist[i]

    fine_ts = []
    for _ in range(N_f):
        u = random.uniform(0, 1)
        idx = 0
        for i in range(N_c):
            if u <= cdf[i]:
                idx = i
                break
        t_min = idx * delta
        t_max = (idx + 1) * delta
        fine_ts.append(random.uniform(t_min, t_max))

    # 6. Combine and sort all points by depth [G]
    all_ts = sorted(coarse_ts + fine_ts)
    all_points = [(ray_origin[0] + t * ray_direction[0],
                   ray_origin[1] + t * ray_direction[1],
                   ray_origin[2] + t * ray_direction[2]) for t in all_ts]

    # 7. Evaluate fine density and color at all points [G]
    results = [evaluate_fine(all_points[i], dense_grid) for i in range(len(all_points))]
    sorted_densities = [r[0] for r in results]
    sorted_colors = [r[1] for r in results]

    # 8. Volume rendering accumulation [S2]
    total_transmittance = 1.0
    accumulated_color = [0.0, 0.0, 0.0]
    for i in range(len(all_points)):
        sigma = sorted_densities[i]
        color = sorted_colors[i]
        if i < len(all_points) - 1:
            delta_i = math.sqrt(sum((all_points[i+1][j] - all_points[i][j])**2 for j in range(3)))
        else:
            delta_i = delta
        alpha = 1.0 - math.exp(-sigma * delta_i)
        weight = total_transmittance * alpha
        accumulated_color = [accumulated_color[j] + weight * color[j] for j in range(3)]
        total_transmittance *= (1.0 - alpha)

    return accumulated_color

def average_pool_3d(grid, kernel_size):
    """Applies average pooling to a 3D grid [G]."""
    # Standard downsampling implementation omitted for brevity
    pass

def trilinear_interpolate(grid, point):
    """Performs trilinear interpolation on a 3D grid [G]."""
    # Standard interpolation implementation omitted for brevity
    pass

def evaluate_fine(point, dense_grid):
    """Returns (sigma, color) from the fine representation [G]."""
    # Standard fine network/grid evaluation omitted for brevity
    pass
```

**Parameters:**
*   $N_c$: Number of coarse samples along the ray [S2].
*   $N_f$: Number of fine samples along the ray [S2].

**Edge Cases and Pitfalls:**
*   **Skipping Surface Regions:** Evaluating density directly from the dense grid $G_\sigma$ at coarse points might skip important surface regions if the coarse sampling interval is larger than the feature thickness. This is handled by convolving the grid with an average pooling kernel to create a coarser estimate on the fly [S2].
*   **Precision:** Accumulated transmittance $\tau_i$ can underflow for long rays with high density. Use log-space accumulation if necessary [G].
*   **Boundary Conditions:** Ensure ray clipping is handled correctly when calculating $\delta_i$ for the last sample [G].

## Subtopics

### Environment Map Integration
A variant used for outdoor or large indoor environments where the feature grid is bounded by a maximum radius $R_{\text{max}}$ [S2]. It integrates an equirectangular environment map to represent visibility at almost infinite distance [S2].
*   **What it is:** Extends the rendering equation to include an environment map term for rays that exit the feature grid [S2].
*   **What sets it apart:** Handles infinite background without requiring an infinitely large feature grid [S2].
*   **When to choose it:** For egocentric view synthesis or large indoor environments where distant visibility is needed [S2].
*   **Link:** [Environment Map Integration](4865-hierarchical-sampling.md#environment-map-integration)
*   **Details:**
    *   **Sampling:** Distribute $N_c$ coarse samples exponentially along the ray rather than uniformly, because the feature grid size increases exponentially along the radial direction [S2].
    *   **Rendering Equation:**
        $$ \hat{C} = \sum_{i=1}^{N} \tau_i (1 - e^{-\sigma(\mathbf{x}_i)\delta_i}) c(\mathbf{x}_i, \mathbf{d}) + \tau_{N+1} c_{\text{env}}(\mathbf{d}) $$
        where $N = N_c + N_f$, $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$, and $c_{\text{env}}$ is the environment map color [S2].
    *   **Environment Map Fetch:**
        $$ c_{\text{env}}(\mathbf{d}) = \mathcal{E}(u, v; \mathbf{d}) $$
        where $(u, v)$ are sampling positions dependent only on viewing direction $\mathbf{d}$ [S2].
    *   **Parameter:** $E \in \mathbb{R}^{H \times W \times 3}$: An equirectangular environment map representing visibility at almost infinite distance [S2].

## Operating characteristics
*   **Time Complexity:** $O(N_c + N_f)$ per ray, where $N_c$ and $N_f$ are the number of coarse and fine samples [S2].
*   **Memory Complexity:** $O(V)$, where $V$ is the volume of the dense grid. No additional memory is allocated for a separate coarse grid [S2].
*   **Strengths:** Memory efficient (no extra coarse grid), adaptive to local density, enables constant frame rate rendering [S1, S2].
*   **Limitations:** Requires a dense grid that stores physically meaningful volume density for the convolution to be effective [S2].
*   **When to use:** When avoiding additional memory allocation for a coarse grid is required, and the density grid stores physically meaningful values [S2].
*   **When not to use:** When the density grid stores abstract neural features rather than physical density, as convolution would not yield a meaningful coarse estimate [S2].

## Background
*   **Origin:** The concept of density control for real-time previewing was introduced as "Point Selection Criteria" to control rendering density using a hierarchy of sampled data [S1]. The specific implementation of "on-the-fly coarse sampling" using convolution on a dense grid was introduced in EgoNeRF [S2].
*   **History:** Early volume rendering used opacity control for image smoothness [S1]. Density control was proposed as an alternative for rendering speed [S1]. Neural volume rendering revived hierarchical sampling, and EgoNeRF adapted it for spherical grids with on-the-fly coarse estimation [S2].
*   **Usage:** Used in real-time volume visualization [S1] and egocentric view synthesis [S2].

## Sources
[S1] Real-time previewing for volume visualization, https://doi.org/10.1145/197938.197974, fetched 2026-11-01
[S2] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
