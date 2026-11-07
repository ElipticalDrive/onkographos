# 3D/2D Hybrid Gaussian Rasterization with Fallback Sigma

Also called: Hybrid Sigma Rasterization, 2D/3D Sigma Fallback

## Overview
This method optimizes the rasterization of 3D Gaussians by dynamically selecting the tighter bounding sigma between a 3D ray-surfel intersection estimate and a pre-computed 2D screen-space fallback. It solves the problem of inefficient culling and inaccurate bounding boxes for Gaussians that are far from the camera or highly anisotropic, ensuring that only relevant splats are processed per pixel. It differs from standard 3D Gaussian Splatting rasterization [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by introducing a hybrid sigma calculation that combines 3D geometric bounds with 2D pre-computed means to improve culling accuracy.

## How it works
**Inputs:**
- `means`: 3D Gaussian means, shape $(N, 3)$.
- `quats`: 3D Gaussian rotations (quaternions), shape $(N, 4)$.
- `scales`: 3D Gaussian scales, shape $(N, 3)$.
- `opacities`: 3D Gaussian opacities, shape $(N)$.
- `sh_coeffs`: Spherical Harmonics coefficients, shape $(N, 1, 3)$ for degree 0.
- `means2d_np`: Pre-computed 2D screen-space means for each camera and Gaussian, shape $(C, N, 2)$.
- `colors_np`: Pre-computed colors for each camera and Gaussian, shape $(C, N, 3)$.
- `normals_np`: Pre-computed normals for each camera and Gaussian, shape $(C, N, 3)$.
- `K`: Camera intrinsic matrix, shape $(3, 3)$.
- `viewmat`: Camera view matrix, shape $(4, 4)$.
- `width`, `height`: Image dimensions.
- `MAX_ALPHA`: Maximum alpha value per splat (e.g., 0.99).
- `ALPHA_THRESHOLD`: Minimum alpha to process (e.g., 0.01).
- `TRANSMITTANCE_THRESHOLD`: Minimum transmittance to continue compositing (e.g., 0.01).
- `backgrounds`: Optional background color per camera, shape $(C, 3)$.

**Outputs:**
- `render_colors`: Rendered RGB image, shape $(C, H, W, 3)$.
- `render_normals`: Rendered normal image, shape $(C, H, W, 3)$.
- `render_alphas`: Rendered alpha image, shape $(C, H, W, 1)$.

**Data Structures:**
- Gaussians are stored in flat arrays for batch processing.
- `means2d_np` is pre-computed by projecting `means` using `K` and `viewmat` and dividing by depth.

**Steps:**
1. Initialize `render_colors`, `render_normals`, and `render_alphas` to zero.
2. Initialize `T` (transmittance) to 1.0 for each pixel.
3. For each camera $c$:
    a. For each pixel $(py, px)$:
        i. Initialize `render_colors[c, py, px] = [0, 0, 0]`, `render_normals[c, py, px] = [0, 0, 0]`.
        ii. For each Gaussian $gid$:
            1. Calculate temporary coordinates:
               $$tmp[0] = \text{projected\_x}$$
               $$tmp[1] = \text{projected\_y}$$
               $$tmp[2] = \text{projected\_z}$$
               (Note: The source uses `tmp` variables directly from projection logic. In implementation, these are the screen-space coordinates before normalization or intermediate projection values. Specifically, `tmp[2]` is the depth.)
            2. If $|tmp[2]| < 10^{-10}$, skip this Gaussian.
            3. Calculate screen-space coordinates:
               $$u_s = tmp[0] / tmp[2]$$
               $$v_s = tmp[1] / tmp[2]$$
            4. Calculate 3D sigma:
               $$\sigma_{3d} = u_s^2 + v_s^2$$
            5. Calculate 2D fallback sigma:
               $$dx = px_f - \mu[0]$$
               $$dy = py_f - \mu[1]$$
               $$\sigma_{2d} = 2.0 \times (dx^2 + dy^2)$$
               Where $\mu = \text{means2d\_np}[c, gid]$.
            6. Calculate hybrid sigma:
               $$\sigma = 0.5 \times \min(\sigma_{3d}, \sigma_{2d})$$
            7. If $\sigma < 0.0$, skip this Gaussian.
            8. Calculate alpha:
               $$\alpha = \min(\text{opacities}[gid] \times \exp(-\sigma), \text{MAX\_ALPHA})$$
            9. If $\alpha < \text{ALPHA\_THRESHOLD}$, skip this Gaussian.
            10. If $T < \text{TRANSMITTANCE\_THRESHOLD}$, break the Gaussian loop for this pixel.
            11. Calculate weight:
                $$w = T \times \alpha$$
            12. Update color and normal:
                $$\text{render\_colors}[c, py, px] += w \times \text{colors\_np}[c, gid]$$
                $$\text{render\_normals}[c, py, px] += w \times \text{normals\_np}[c, gid]$$
            13. Update transmittance:
                $$T = T \times (1.0 - \alpha)$$
        iii. Set `render_alphas[c, py, px, 0] = 1.0 - T`.
    b. If `backgrounds` is not None:
        i. Convert `backgrounds` to numpy array `bg_np`.
        ii. For each camera $c$:
            $$\text{render\_colors}[c] += (1.0 - \text{render\_alphas}[c]) \times \text{bg\_np}[c]$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_gaussian_rasterization(
    means, quats, scales, opacities, sh_coeffs,
    means2d_np, colors_np, normals_np,
    K, viewmat, width, height,
    MAX_ALPHA=0.99, ALPHA_THRESHOLD=0.01, TRANSMITTANCE_THRESHOLD=0.01,
    backgrounds=None
):
    C = means2d_np.shape[0]
    N = means.shape[0]
    render_colors = np.zeros((C, height, width, 3))
    render_normals = np.zeros((C, height, width, 3))
    render_alphas = np.zeros((C, height, width, 1))

    for c in range(C):
        for py in range(height):
            for px in range(width):
                T = 1.0
                current_color = np.array([0.0, 0.0, 0.0])
                current_normal = np.array([0.0, 0.0, 0.0])

                for gid in range(N):
                    # Projection logic implied by tmp variables
                    # tmp[0], tmp[1], tmp[2] are derived from means, quats, scales, K, viewmat
                    # For implementation, these must be calculated from 3D Gaussian parameters
                    # to screen space. The source assumes tmp is available.
                    # We assume a helper function `project_gaussian` exists or inline it.
                    # However, the source notes explicitly use tmp[0], tmp[1], tmp[2].
                    # Let's assume these are the projected coordinates in a specific space.
                    # Based on standard 3DGS, tmp[2] is depth.
                    
                    # Placeholder for projection calculation as source does not give full projection code
                    # but uses tmp[0], tmp[1], tmp[2].
                    # In a real implementation, this would involve:
                    # 1. Transform mean to camera space.
                    # 2. Compute covariance in camera space.
                    # 3. Project to 2D.
                    # The source notes skip this and jump to tmp.
                    # We will assume tmp is calculated as per standard 3DGS projection.
                    
                    # Note: The source code snippet shows `tmp` being used directly.
                    # We must define tmp.
                    # Let's assume `tmp` is the result of projecting the Gaussian center.
                    # tmp[0] = x_screen, tmp[1] = y_screen, tmp[2] = z_depth
                    # This is a simplification. The actual 3DGS projection is more complex.
                    # However, the note says "Calculate temporary coordinates tmp[0], tmp[1], tmp[2]".
                    # It does not specify how. We will assume standard projection.
                    
                    # Since the source does not provide the projection formula, we mark it as [G].
                    # Standard 3DGS projection:
                    # mean_cam = viewmat @ [mean, 1]
                    # mean_proj = K @ mean_cam
                    # tmp[0] = mean_proj[0]
                    # tmp[1] = mean_proj[1]
                    # tmp[2] = mean_proj[2]
                    
                    # This is a significant gap. The source notes assume tmp is known.
                    # We will write the projection step as [G] to fill the gap.
                    
                    # [G] Standard 3D Gaussian Splatting projection:
                    # mean_cam = viewmat @ np.append(means[gid], 1.0)
                    # mean_proj = K @ mean_cam[:3]
                    # tmp = np.array([mean_proj[0], mean_proj[1], mean_proj[2]])
                    
                    # Re-reading source: "Calculate temporary coordinates tmp[0], tmp[1], tmp[2]."
                    # It does not say how. It just uses them.
                    # We will assume the caller provides `tmp` or we calculate it.
                    # To be self-contained, we must calculate it.
                    
                    # Let's assume the source implies the standard 3DGS projection.
                    # We will write the projection code.
                    
                    mean_cam = viewmat @ np.append(means[gid], 1.0)
                    mean_proj = K @ mean_cam[:3]
                    tmp = np.array([mean_proj[0], mean_proj[1], mean_proj[2]])

                    if abs(tmp[2]) < 1e-10:
                        continue

                    us = tmp[0] / tmp[2]
                    vs = tmp[1] / tmp[2]

                    sigma_3d = us * us + vs * vs

                    mu = means2d_np[c, gid]
                    dx = px - mu[0]
                    dy = py - mu[1]
                    sigma_2d = 2.0 * (dx * dx + dy * dy)

                    sigma = 0.5 * min(sigma_3d, sigma_2d)

                    if sigma < 0.0:
                        continue

                    alpha = min(opacities[gid] * np.exp(-sigma), MAX_ALPHA)

                    if alpha < ALPHA_THRESHOLD:
                        continue

                    if T < TRANSMITTANCE_THRESHOLD:
                        break

                    weight = T * alpha
                    current_color += weight * colors_np[c, gid]
                    current_normal += weight * normals_np[c, gid]
                    T *= (1.0 - alpha)

                render_colors[c, py, px] = current_color
                render_normals[c, py, px] = current_normal
                render_alphas[c, py, px, 0] = 1.0 - T

        if backgrounds is not None:
            bg_np = backgrounds[c]
            render_colors[c] += (1.0 - render_alphas[c, :, :, 0])[:, :, np.newaxis] * bg_np

    return render_colors, render_normals, render_alphas
```

**Parameters:**
- `MAX_ALPHA`: Typically 0.99.
- `ALPHA_THRESHOLD`: Typically 0.01.
- `TRANSMITTANCE_THRESHOLD`: Typically 0.01.

**Edge Cases:**
- If $|tmp[2]| < 10^{-10}$, the Gaussian is skipped to avoid division by zero.
- If $\sigma < 0.0$, the Gaussian is skipped.
- If $T < \text{TRANSMITTANCE\_THRESHOLD}$, the loop breaks, assuming further Gaussians will have negligible contribution.

## Subtopics

### [3D Gaussian Splatting Differentiable Rasterizer > Backward Pass](8352-3d-gaussian-splatting-differentiable-rasterizer-backward-pas.md)
Extends the rasterizer to support Time-of-Flight (ToF) phase data and depth, returning gradients for color, phasor, depth, normal, and accumulated opacity channels.

### [Image-Based Gaussian Splatting (IBGS)](8764-image-based-gaussian-splatting-ibgs.md)
Augments 3DGS by predicting a color residual from neighboring training images to capture high-frequency details, using a lightweight neural network.

## Operating characteristics
- **Time Complexity:** $O(C \times H \times W \times N)$ in the worst case, where $C$ is the number of cameras, $H \times W$ is the image resolution, and $N$ is the number of Gaussians. The hybrid sigma check allows early culling, reducing the effective $N$ per pixel.
- **Memory Complexity:** $O(C \times H \times W + C \times N)$ for storing render buffers and pre-computed 2D means/colors/normals.
- **Strengths:** Improved culling accuracy for distant or anisotropic Gaussians, leading to fewer unnecessary alpha compositing operations.
- **Limitations:** Requires pre-computation of 2D means, colors, and normals for all cameras and Gaussians, increasing memory usage.
- **When to use:** When rendering large scenes with many Gaussians where standard 3D culling is insufficient.
- **When not to use:** When memory is constrained, as pre-computed 2D data increases memory footprint.

## Background
This method is part of the `gsplat-mlx` library [S1, unofficial], which provides a Metal-based implementation of 3D Gaussian Splatting for Apple Silicon. It was developed to optimize rasterization performance on mobile and embedded devices.

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
