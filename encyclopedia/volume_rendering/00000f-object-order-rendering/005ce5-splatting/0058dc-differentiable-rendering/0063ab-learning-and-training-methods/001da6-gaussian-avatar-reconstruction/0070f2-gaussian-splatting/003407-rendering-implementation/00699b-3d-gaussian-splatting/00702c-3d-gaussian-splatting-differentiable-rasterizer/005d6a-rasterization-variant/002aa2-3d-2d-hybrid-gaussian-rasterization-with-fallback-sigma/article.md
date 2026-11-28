# 3D/2D Hybrid Gaussian Rasterization with Fallback Sigma

Also called: Hybrid Sigma Rasterization, Fallback Sigma Gaussian Rasterization

## Overview
This method renders 3D Gaussians by combining a 3D geometric sigma calculation with a 2D screen-space fallback sigma to determine the effective spread of each Gaussian splat. It solves the problem of inaccurate alpha culling and rendering artifacts when Gaussians are far from the camera or when the 3D covariance projection is numerically unstable. It sets itself apart from standard rasterization by introducing a hybrid sigma bound that ensures tighter alpha thresholds, and from [Masked-Rasterization](4366-masked-rasterization.md) by using a continuous sigma value rather than a binary mask.

## How it works
**Inputs:**
*   `means`: 3D Gaussian means, shape $[N, 3]$.
*   `opacities`: Per-Gaussian opacity, shape $[N]$.
*   `colors`: Per-Gaussian color, shape $[N, 3]$.
*   `normals`: Per-Gaussian normal, shape $[N, 3]$.
*   `means2d`: Pre-computed 2D screen-space means, shape $[C, N, 2]$, where $C$ is the number of cameras.
*   `cameras`: Camera parameters (intrinsic $K$, extrinsic $V$).
*   `background`: Optional background color, shape $[C, 3]$.

**Outputs:**
*   `render_colors`: Rendered RGB images, shape $[C, H, W, 3]$.
*   `render_normals`: Rendered normal images, shape $[C, H, W, 3]$.
*   `render_alphas`: Rendered alpha images, shape $[C, H, W, 1]$.

**Parameters:**
*   `MAX_ALPHA`: Maximum alpha value per splat (e.g., 1.0).
*   `ALPHA_THRESHOLD`: Minimum alpha to process (e.g., 0.01).
*   `TRANSMITTANCE_THRESHOLD`: Minimum transmittance to continue loop (e.g., 0.0001).
*   `EPSILON`: Small constant for depth check (e.g., $10^{-10}$).

**Data Structures:**
*   `render_colors`: Accumulator for color.
*   `render_normals`: Accumulator for normal.
*   `T`: Current transmittance, initialized to 1.0 per pixel.

**Steps:**
1.  **Initialization:** Initialize `render_colors`, `render_normals` to 0. Initialize `render_alphas` to 0. Initialize `T` (transmittance) to 1.0 for all pixels.
2.  **Pixel Loop:** Iterate over each pixel $(p_y, p_x)$ and each camera $c$.
3.  **Gaussian Loop:** Iterate through all Gaussians $i$ (index $gid$).
    a.  **Projection Check:** Calculate temporary coordinates `tmp` by projecting the 3D mean. If `abs(tmp[2]) < EPSILON`, skip this Gaussian [S1, unofficial].
    b.  **Screen Coordinates:** Calculate screen-space coordinates:
        $$ u_s = \frac{tmp[0]}{tmp[2]} $$
        $$ v_s = \frac{tmp[1]}{tmp[2]} $$
    c.  **3D Sigma Calculation:** Calculate the 3D sigma based on the distance from the projected center:
        $$ \sigma_{3d} = u_s^2 + v_s^2 $$
    d.  **2D Fallback Sigma Calculation:** Use the pre-computed 2D mean $\boldsymbol{\mu}_{2d}$ for camera $c$ and Gaussian $gid$. Calculate differences:
        $$ dx = p_x - \mu_{2d}[0] $$
        $$ dy = p_y - \mu_{2d}[1] $$
        $$ \sigma_{2d} = 2.0 \times (dx^2 + dy^2) $$
    e.  **Hybrid Sigma:** Take the tighter bound:
        $$ \sigma = 0.5 \times \min(\sigma_{3d}, \sigma_{2d}) $$
    f.  **Alpha Calculation:** Calculate alpha:
        $$ \alpha = \min(\text{opacities}[gid] \times \exp(-\sigma), \text{MAX\_ALPHA}) $$
    g.  **Culling:** If $\alpha < \text{ALPHA\_THRESHOLD}$, skip this Gaussian [S1, unofficial].
    h.  **Transmittance Check:** If current transmittance $T < \text{TRANSMITTANCE\_THRESHOLD}$, break the Gaussian loop for this pixel [S1, unofficial].
    i.  **Compositing:** Calculate weight $w = T \times \alpha$.
        $$ \text{render\_colors}[c, p_y, p_x] \leftarrow \text{render\_colors}[c, p_y, p_x] + w \times \text{colors}[gid] $$
        $$ \text{render\_normals}[c, p_y, p_x] \leftarrow \text{render\_normals}[c, p_y, p_x] + w \times \text{normals}[gid] $$
        $$ T \leftarrow T \times (1.0 - \alpha) $$
4.  **Alpha Finalization:** After the Gaussian loop, set:
    $$ \text{render\_alphas}[c, p_y, p_x, 0] = 1.0 - T $$
5.  **Background Addition:** If `background` is provided, for each camera $c$:
    $$ \text{render\_colors}[c] \leftarrow \text{render\_colors}[c] + (1.0 - \text{render\_alphas}[c]) \times \text{background}[c] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_rasterization_with_fallback_sigma(
    means: Array,
    opacities: Array,
    colors: Array,
    normals: Array,
    means2d: Array,  # Shape [C, N, 2]
    cameras: List[Camera],
    background: Optional[Array] = None,
    max_alpha: float = 1.0,
    alpha_threshold: float = 0.01,
    transmittance_threshold: float = 0.0001,
    epsilon: float = 1e-10
) -> Tuple[Array, Array, Array]:
    """
    Renders 3D Gaussians using hybrid 3D/2D sigma fallback.
    
    Args:
        means: [N, 3] 3D means.
        opacities: [N] opacities.
        colors: [N, 3] colors.
        normals: [N, 3] normals.
        means2d: [C, N, 2] pre-computed 2D means per camera.
        cameras: List of camera objects with intrinsic/extrinsic matrices.
        background: [C, 3] optional background colors.
        max_alpha: Max alpha per splat.
        alpha_threshold: Min alpha to process.
        transmittance_threshold: Min transmittance to continue.
        epsilon: Depth check epsilon.
        
    Returns:
        render_colors: [C, H, W, 3]
        render_normals: [C, H, W, 3]
        render_alphas: [C, H, W, 1]
    """
    C = len(cameras)
    H, W = cameras[0].height, cameras[0].width
    
    # Initialize outputs
    render_colors = mx.zeros((C, H, W, 3))
    render_normals = mx.zeros((C, H, W, 3))
    render_alphas = mx.zeros((C, H, W, 1))
    
    for c in range(C):
        cam = cameras[c]
        # Project all means to screen space for this camera
        # This step is assumed to be done externally or via a helper
        # to get tmp coordinates and depth
        projected_means = project_gaussians(means, cam) # Returns [N, 3] in clip space
        
        for py in range(H):
            for px in range(W):
                T = 1.0
                for gid in range(len(means)):
                    tmp = projected_means[gid]
                    
                    # Check depth
                    if abs(tmp[2]) < epsilon:
                        continue
                    
                    # Screen coordinates
                    us = tmp[0] / tmp[2]
                    vs = tmp[1] / tmp[2]
                    
                    # 3D Sigma
                    sigma_3d = us * us + vs * vs
                    
                    # 2D Fallback Sigma
                    mu_2d = means2d[c, gid]
                    dx = px - mu_2d[0]
                    dy = py - mu_2d[1]
                    sigma_2d = 2.0 * (dx * dx + dy * dy)
                    
                    # Hybrid Sigma
                    sigma = 0.5 * min(sigma_3d, sigma_2d)
                    
                    # Alpha
                    alpha = min(opacities[gid] * mx.exp(-sigma), max_alpha)
                    
                    # Culling
                    if alpha < alpha_threshold:
                        continue
                    
                    # Transmittance check
                    if T < transmittance_threshold:
                        break
                    
                    # Compositing
                    weight = T * alpha
                    render_colors[c, py, px] += weight * colors[gid]
                    render_normals[c, py, px] += weight * normals[gid]
                    T *= (1.0 - alpha)
                
                render_alphas[c, py, px, 0] = 1.0 - T
    
    # Background addition
    if background is not None:
        for c in range(C):
            render_colors[c] += (1.0 - render_alphas[c, :, :, 0]) * background[c]
            
    return render_colors, render_normals, render_alphas
```

**Edge Cases and Pitfalls:**
*   **Depth Zero:** If `tmp[2]` is near zero, division by zero occurs. The check `abs(tmp[2]) < EPSILON` handles this [S1, unofficial].
*   **Negative Sigma:** Although unlikely with squared terms, a check `if sigma < 0.0` is included as a safeguard [S1, unofficial].
*   **Numerical Stability:** The hybrid sigma ensures that even if the 3D projection is inaccurate, the 2D fallback provides a reasonable bound.

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer > Backward Pass](8352-3d-gaussian-splatting-differentiable-rasterizer-backward-pas.md)
Computes gradients for training.

### [Image-Based Gaussian Splatting (IBGS)](8764-image-based-gaussian-splatting-ibgs.md)
Adds color residuals.

### [3D Gaussian Splatting Rendering with Language Features](12439-3d-gaussian-splatting-rendering-with-language-features.md)
Adds language feature channels.

## Operating characteristics
*   **Time Complexity:** $O(C \times H \times W \times N)$ in the worst case, where $C$ is cameras, $H,W$ resolution, $N$ Gaussians. Culling reduces this significantly.
*   **Memory Complexity:** $O(C \times H \times W)$ for output buffers. $O(C \times N)$ for `means2d`.
*   **Strengths:** Robust to projection errors; tighter alpha culling improves performance.
*   **Limitations:** Requires pre-computation of `means2d`; hybrid sigma calculation adds overhead per pixel.
*   **When to Use:** When standard rasterization suffers from artifacts due to inaccurate sigma estimation or when culling efficiency is critical.

## Background
This method is part of the `gsplat-mlx` library, an unofficial implementation of 3D Gaussian Splatting for the MLX framework [S1, unofficial]. It introduces the hybrid sigma approach to improve robustness and culling.

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
