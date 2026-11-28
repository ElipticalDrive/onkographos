# AdR-Gaussian

Also called: Adaptive Radius Gaussian Splatting

## Overview
AdR-Gaussian is a rendering acceleration method for 3D Gaussian Splatting that optimizes the rasterization pipeline by moving serial culling operations from the Render stage to the parallel Preprocess stage and balancing computational loads across pixel threads [S1]. It achieves significant speedups without reducing the total number of Gaussians, thereby avoiding the quality degradation associated with pruning-based methods [S1]. Unlike its siblings, which often rely on reducing Gaussian counts or hardware-specific optimizations, AdR-Gaussian focuses on algorithmic pipeline efficiency through adaptive radius culling and load balancing [S1].

## How it works
AdR-Gaussian accelerates rendering by reducing the computational cost of the Render stage, which typically accounts for over 50% of the total rasterization time due to serial alpha blending and complex splatting calculations [S1]. The method operates in two phases: Early Culling and Load Balancing [S1].

### Inputs and Outputs
*   **Inputs:** A set of 3D Gaussians defined by center position $\mathbf{X}_0$, 3D covariance matrix $\Sigma$, opacity $\sigma$, and spherical harmonics coefficients for color [S1]. Camera parameters for projection [S1].
*   **Outputs:** A rendered image where each pixel color $C(p)$ is computed via front-to-back alpha compositing [S1].

### Data Structures
The method utilizes the standard tile-based rasterization pipeline structure [S1]:
1.  **Gaussian Attributes:** Each Gaussian $i$ stores $\sigma_i$, $\mathbf{X}_0$, $\Sigma$, and SH coefficients [S1].
2.  **Tile Grid:** The image is divided into fixed-size tiles, typically $16 \times 16$ pixels [S1].
3.  **Gaussian-Tile Pairs:** A list of pairs $(G_i, T_j)$ indicating which Gaussians overlap with which tiles, generated during the Preprocess stage [S1].
4.  **Sorted Lists:** For each tile, Gaussians are sorted by view-space depth to enable correct alpha compositing [S1].

### Parameters
*   $\alpha_{low}$: A predefined minimum splatting opacity constant used as the threshold for culling [S1].
*   Tile Size: Typically $16 \times 16$ pixels [S1].

### Steps

#### Phase 1: Early Culling
This phase moves culling from the serial Render stage to the parallel Preprocess stage to reduce the number of Gaussian-Tile pairs processed later [S1].

1.  **Preprocess Stage (Parallel per Gaussian):** For each Gaussian $i$:
    a.  Compute the projected 2D covariance matrix $\Sigma'$ based on the camera view [S1].
    b.  Calculate the **Adaptive Radius** ($r_{ad}$):
        i.  Define the ellipse of minimum splatting opacity $\alpha_{low}$ using the inequality:
            $$ \mathbf{x}^T \Sigma'^{-1} \mathbf{x} \le 2 \ln \left( \frac{\sigma_i}{\alpha_{low}} \right) $$
            where $\mathbf{x}$ is the 2D vector from the Gaussian center to a pixel [S1].
        ii. Expand this into the general ellipse form $A x^2 + B y^2 + C xy + D \le 0$, with coefficients:
            $$ A = \Sigma'_{YY}, \quad B = \Sigma'_{XX}, \quad C = -2\Sigma'_{XY} $$
            $$ D = -2 \sqrt{\Sigma'_{XX} \Sigma'_{YY} - \Sigma'_{XY}^2} \ln \left( \frac{\sigma_i}{\alpha_{low}} \right) $$
        iii. Calculate $r_{ad}$ as half the major axis length of this ellipse [S1].
    c.  Calculate the **Original Radius** ($r_o$):
        i.  Determine the radius based on the 99% confidence interval of the Gaussian [S1].
    d.  Determine the **Final Culling Radius** ($r$):
        $$ r = \min(r_{ad}, r_o) $$
        *Note:* If $\sigma_i$ is high or $\alpha_{low}$ is low, $r_{ad}$ may exceed $r_o$ [S1]. Clamping to $r_o$ ensures the rendering range does not exceed the standard confidence interval [S1].
    e.  **AABB Culling:**
        i.  Calculate the maximum x and y coordinates ($x_{max}, y_{max}$) of the ellipse defined in step 1.b [S1].
        ii. Set half-width $w = x_{max}$ and half-height $h = y_{max}$ [S1].
        iii. Use the Axis-Aligned Bounding Box (AABB) defined by $w$ and $h$ to cull tiles, which provides more accurate culling in the minor axis direction than the circular adaptive radius [S1].
    f.  Identify all tiles overlapping with the Gaussian using the calculated AABB and generate Gaussian-Tile pairs [S1].

2.  **Pipeline Stages:**
    The generated pairs proceed through the standard pipeline stages:
    *   **InclusiveSum & DuplicateWithKeys:** Parallel processing to gather tile coverage information [S1].
    *   **SortPairs:** Sort Gaussian-Tile pairs [S1].
    *   **IdentifyTileRanges:** Determine start/end indices for each tile [S1].

#### Phase 2: Load Balancing
This phase addresses the uneven computational load across pixel threads caused by varying numbers of Gaussians per pixel [S1].

1.  **Assess Load:** Quantify the variance in the number of Gaussians required for rendering each pixel [S1].
2.  **Adjust Gaussian Count:**
    *   For **heavy-load pixels** (high variance/many Gaussians): Reduce the number of rendered Gaussians to minimize thread waiting time [S1].
    *   For **light-load pixels** (low variance/few Gaussians): Increase the number of rendered Gaussians to hedge against quality loss [S1].
3.  **Render:** Execute the Render stage in parallel for all pixels using the balanced set of Gaussians [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class Gaussian:
    def __init__(self, center_2d, opacity, projected_covariance, color, depth):
        self.center_2d = np.array(center_2d, dtype=float)
        self.opacity = opacity
        self.projected_covariance = np.array(projected_covariance, dtype=float)
        self.color = np.array(color, dtype=float)
        self.depth = depth

def compute_ellipse_params(sigma_prime, sigma_i, alpha_low):
    Sigma_XX = sigma_prime[0, 0]
    Sigma_YY = sigma_prime[1, 1]
    Sigma_XY = sigma_prime[0, 1]
    A = Sigma_YY
    B = Sigma_XX
    C = -2 * Sigma_XY
    D = -2 * math.sqrt(Sigma_XX * Sigma_YY - Sigma_XY**2) * math.log(sigma_i / alpha_low)
    return A, B, C, D, Sigma_XX, Sigma_YY, Sigma_XY

def get_adaptive_radius(A, B, C, D):
    M = np.array([[A, C/2], [C/2, B]])
    eigenvalues = np.linalg.eigvals(M)
    lambda_min = min(eigenvalues)
    if lambda_min <= 0:
        return 0.0
    return math.sqrt(-D / lambda_min)

def get_aabb_extents(A, B, C, D):
    det = 4 * A * B - C * C
    if det <= 0:
        return 0.0, 0.0
    x_max = math.sqrt(-D * 4 * B / det)
    y_max = math.sqrt(-D * 4 * A / det)
    return x_max, y_max

def get_original_radius_99(sigma_prime):
    # Standard 99% confidence interval for 2D Gaussian [G]
    eigenvalues = np.linalg.eigvals(sigma_prime)
    lambda_max = max(eigenvalues)
    return math.sqrt(-2 * math.log(0.01) * lambda_max)

def ad_r_gaussian_preprocess(gaussians, alpha_low, tile_size=16):
    gaussian_tile_pairs = []
    for g in gaussians:
        sigma_i = g.opacity
        sigma_prime = g.projected_covariance
        center = g.center_2d

        A, B, C, D, _, _, _ = compute_ellipse_params(sigma_prime, sigma_i, alpha_low)

        r_ad = get_adaptive_radius(A, B, C, D)
        r_o = get_original_radius_99(sigma_prime)
        r = min(r_ad, r_o)

        # AABB culling for tighter bounds in minor axis
        x_max, y_max = get_aabb_extents(A, B, C, D)
        w, h = x_max, y_max

        # Determine tiles covered by AABB
        min_tx = math.floor((center[0] - w) / tile_size)
        max_tx = math.ceil((center[0] + w) / tile_size) - 1
        min_ty = math.floor((center[1] - h) / tile_size)
        max_ty = math.ceil((center[1] + h) / tile_size) - 1

        for ty in range(min_ty, max_ty + 1):
            for tx in range(min_tx, max_tx + 1):
                gaussian_tile_pairs.append((g, (tx, ty)))
    return gaussian_tile_pairs

def ad_r_gaussian_load_balance(gaussian_tile_pairs, tile_size=16):
    # Count Gaussians per pixel
    pixel_gaussians = {}
    for g, tile in gaussian_tile_pairs:
        for py in range(tile[1] * tile_size, (tile[1] + 1) * tile_size):
            for px in range(tile[0] * tile_size, (tile[0] + 1) * tile_size):
                pixel_key = (px, py)
                if pixel_key not in pixel_gaussians:
                    pixel_gaussians[pixel_key] = []
                pixel_gaussians[pixel_key].append(g)

    # Balance load: target count is median of all pixel counts
    counts = [len(gs) for gs in pixel_gaussians.values()]
    target_count = sorted(counts)[len(counts) // 2]

    balanced_pairs = []
    for p, gs in pixel_gaussians.items():
        if len(gs) > target_count:
            # Heavy-load: reduce to target count (keep first N by depth)
            gs_sorted = sorted(gs, key=lambda x: x.depth)
            gs = gs_sorted[:target_count]
        elif len(gs) < target_count:
            # Light-load: increase by relaxing culling thresholds [G]
            pass
        for g in gs:
            balanced_pairs.append((g, p))
    return balanced_pairs

def render(balanced_pairs, tile_size=16, image_height=1080, image_width=1920):
    # Group by tile
    tile_gaussians = {}
    for g, p in balanced_pairs:
        tx, ty = p[0] // tile_size, p[1] // tile_size
        if (tx, ty) not in tile_gaussians:
            tile_gaussians[(tx, ty)] = []
        tile_gaussians[(tx, ty)].append(g)

    image = np.zeros((image_height, image_width, 3))
    for tile, gs in tile_gaussians.items():
        gs_sorted = sorted(gs, key=lambda x: x.depth)
        for py in range(tile[1] * tile_size, (tile[1] + 1) * tile_size):
            for px in range(tile[0] * tile_size, (tile[0] + 1) * tile_size):
                alpha = 0.0
                color = np.zeros(3)
                for g in gs_sorted:
                    dx = px - g.center_2d[0]
                    dy = py - g.center_2d[1]
                    x = np.array([dx, dy])
                    alpha_i = g.opacity * math.exp(-0.5 * x.T @ np.linalg.inv(g.projected_covariance) @ x)
                    color += g.color * alpha_i * (1 - alpha)
                    alpha += alpha_i * (1 - alpha)
                    if alpha > 0.99:
                        break
                image[py, px] = color
    return image
```

## Subtopics

### [Pre-rendering Frustum Culling](17570-pre-rendering-frustum-culling.md)
Pre-rendering Frustum Culling is an engineering optimization for 3D Gaussian Splatting that eliminates unnecessary Gaussians from rasterization input by explicitly computing and storing in-frustum indices before the rendering kernel executes. Unlike standard implementations where frustum culling is fused into rendering kernels and computed implicitly by CUDA threads without explicit storage, this method explicitly stores indices to allow the elimination of non-in-frustum Gaussians from the input tensor, thereby reducing GPU memory usage and computation.

## Operating characteristics
*   **Time Complexity:** The Render stage ($E_p$) typically takes the largest proportion of computational cost (over 50%) due to serial alpha blending and complex splatting color calculation per Gaussian-Pixel pair [S1]. AdR-Gaussian reduces this by minimizing the number of Gaussian-Tile pairs entering the Render stage via early culling and balancing the load across pixel threads [S1].
*   **Memory Complexity:** By moving culling to the Preprocess stage and reducing the number of Gaussian-Tile pairs, memory usage for intermediate data structures (like the sorted lists) is reduced [S1].
*   **Measured Performance:** On the Mip-NeRF360 dataset, the method attains an average 590 FPS rendering speed on a single NVIDIA RTX3090 GPU [S1]. Experiments on three datasets demonstrate an average 310% acceleration in rendering speed [S1].
*   **Strengths:** Achieves acceleration without rendering quality loss by optimizing the rasterization pipeline rather than reducing the Gaussian count [S1]. The AABB culling provides more accurate culling in minor axis directions than circular radii [S1].
*   **Limitations:** Requires additional computation in the Preprocess stage to calculate adaptive radii and AABBs [S1]. The load balancing phase adds complexity to the pipeline [S1].
*   **When to Use:** When high rendering speed is required without sacrificing quality, and the bottleneck is the serial Render stage of the standard 3DGS pipeline [S1].
*   **When Not to Use:** When the bottleneck is elsewhere (e.g., data transfer or preprocessing) or when hardware acceleration is available that handles these optimizations natively [S1].

## Background
AdR-Gaussian was proposed to address the issues of avoidable serial Gaussian culling and uneven load distribution in 3D Gaussian Splatting rendering [S1]. It builds on the standard 3DGS rasterization pipeline [S1]. The method was introduced in the paper "AdR-Gaussian: Accelerating Gaussian Splatting with Adaptive Radius" [S1].

## Sources
[S1] AdR-Gaussian: Accelerating Gaussian Splatting with Adaptive Radius, https://doi.org/10.1145/3680528.3687675, fetched 2026-10-26
