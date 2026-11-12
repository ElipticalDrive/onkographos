# 3D/2D Hybrid Gaussian Rasterization with Fallback Sigma

Also called: Hybrid Sigma Rasterization, Fallback Sigma Method

## Overview
This method renders 3D Gaussians by combining a screen-space 2D Gaussian approximation with a ray-surfel 3D sigma calculation to determine the tighter bounding sigma for alpha attenuation [S1, unofficial]. It solves the problem of inaccurate alpha estimation in standard splatting when Gaussians are viewed at extreme angles or large distances, ensuring robust occlusion and correct opacity accumulation [S1, unofficial]. It differs from standard 3D Gaussian Splatting rasterization by explicitly computing a fallback 2D sigma based on pre-computed screen-space means and selecting the minimum of the 3D and 2D sigmas to define the effective spread [S1, unofficial].

## How it works
**Inputs:**
- `means`: 3D positions of $N$ Gaussians, shape $[N, 3]$.
- `quats`: Rotations (quaternions) of $N$ Gaussians, shape $[N, 4]$.
- `scales`: Scales of $N$ Gaussians, shape $[N, 3]$.
- `opacities`: Opacity values of $N$ Gaussians, shape $[N]$.
- `sh_coeffs`: Spherical Harmonic coefficients, shape $[N, 1, 3]$ (SH degree 0 assumed in source) [S1, unofficial].
- `means2d_np`: Pre-computed 2D screen-space means for each camera, shape $[C, N, 2]$ [S1, unofficial].
- `cameras`: Camera intrinsics and extrinsics for $C$ views.
- `backgrounds`: Optional background colors per camera, shape $[C, 3]$.

**Outputs:**
- `render_colors`: Rendered RGB images, shape $[C, H, W, 3]$.
- `render_normals`: Rendered normal images, shape $[C, H, W, 3]$.
- `render_alphas`: Accumulated alpha images, shape $[C, H, W, 1]$.

**Parameters:**
- `MAX_ALPHA`: Maximum alpha value per Gaussian (default not specified, typically 1.0) [S1, unofficial].
- `ALPHA_THRESHOLD`: Minimum alpha to process (default not specified) [S1, unofficial].
- `TRANSMITTANCE_THRESHOLD`: Minimum transmittance to continue loop (default not specified) [S1, unofficial].
- `EPSILON`: Depth threshold for skipping, $10^{-10}$ [S1, unofficial].

**Data Structures:**
- Gaussians are stored in flat arrays for batch processing.
- Screen-space coordinates are computed per pixel and per Gaussian.

**Steps:**

1. **Initialization**: Initialize `render_colors`, `render_normals`, and `render_alphas` to zero. Initialize transmittance $T = 1.0$ for each pixel.

2. **Pixel Loop**: Iterate over each pixel $(p_y, p_x)$ and camera $c$.

3. **Gaussian Loop**: Iterate over each Gaussian $g_i$ (index $gid$).
   a. **Coordinate Transformation**: Calculate temporary coordinates $tmp[0], tmp[1], tmp[2]$ by projecting the 3D mean of the Gaussian into the camera space. The exact projection formula depends on the camera model (pinhole assumed in source context) [S1, unofficial].
   b. **Depth Check**: If $|tmp[2]| < 10^{-10}$, skip this Gaussian [S1, unofficial].
   c. **Screen-Space Coordinates**: Calculate screen-space coordinates:
      $$u_s = \frac{tmp[0]}{tmp[2]}$$
      $$v_s = \frac{tmp[1]}{tmp[2]}$$
      [S1, unofficial]
   d. **3D Sigma Calculation**: Calculate the 3D sigma based on the ray-surfel intersection distance:
      $$\sigma_{3d} = u_s^2 + v_s^2$$
      [S1, unofficial]
   e. **2D Fallback Sigma Calculation**: Retrieve the pre-computed 2D mean $\mu = \text{means2d\_np}[c, gid]$ [S1, unofficial]. Calculate differences:
      $$dx = p_x - \mu[0]$$
      $$dy = p_y - \mu[1]$$
      Calculate the 2D fallback sigma:
      $$\sigma_{2d} = 2.0 \times (dx^2 + dy^2)$$
      [S1, unofficial]
   f. **Hybrid Sigma Selection**: Select the tighter bound:
      $$\sigma = 0.5 \times \min(\sigma_{3d}, \sigma_{2d})$$
      [S1, unofficial]
   g. **Sigma Validation**: If $\sigma < 0.0$, skip the Gaussian [S1, unofficial].
   h. **Alpha Calculation**: Calculate the alpha value:
      $$\alpha = \min(\text{opacities}[gid] \times \exp(-\sigma), \text{MAX\_ALPHA})$$
      [S1, unofficial]
   i. **Alpha Threshold**: If $\alpha < \text{ALPHA\_THRESHOLD}$, skip the Gaussian [S1, unofficial].
   j. **Transmittance Check**: If current transmittance $T < \text{TRANSMITTANCE\_THRESHOLD}$, break the Gaussian loop for this pixel [S1, unofficial].
   k. **Accumulation**: Calculate weight:
      $$w = T \times \alpha$$
      Add to color and normal buffers:
      $$\text{render\_colors}[c, p_y, p_x] += w \times \text{colors}[gid]$$
      $$\text{render\_normals}[c, p_y, p_x] += w \times \text{normals}[gid]$$
      Update transmittance:
      $$T \leftarrow T \times (1.0 - \alpha)$$
      [S1, unofficial]

4. **Alpha Finalization**: After the Gaussian loop, set:
   $$\text{render\_alphas}[c, p_y, p_x, 0] = 1.0 - T$$
   [S1, unofficial]

5. **Background Compositing**: If `backgrounds` is not None:
   a. Convert `backgrounds` to numpy array `bg_np` [S1, unofficial].
   b. For each camera $c$, add background contribution:
      $$\text{render\_colors}[c] += (1.0 - \text{render\_alphas}[c]) \times \text{bg\_np}[c]$$
      [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_gaussian_rasterization(means, quats, scales, opacities, sh_coeffs, means2d_np, cameras, backgrounds, MAX_ALPHA=1.0, ALPHA_THRESHOLD=1e-4, TRANSMITTANCE_THRESHOLD=1e-4):
    """
    Renders Gaussians using hybrid 3D/2D sigma fallback.
    Assumes SH degree 0 for simplicity as per source [S1, unofficial].
    """
    C, N = cameras.shape[0], means.shape[0]
    H, W = cameras[0].height, cameras[0].width
    
    render_colors = np.zeros((C, H, W, 3))
    render_normals = np.zeros((C, H, W, 3))
    render_alphas = np.zeros((C, H, W, 1))
    
    # Pre-compute colors from SH coefficients (SH degree 0)
    # colors shape: [N, 3]
    colors = sh_coeffs[:, 0, :] 
    
    # Pre-compute normals (assumed to be provided or derived from quats/scales)
    # For this implementation, we assume normals are passed or derived.
    # Source mentions normals_np[c, gid], implying per-camera normals might be needed or are constant.
    # We assume normals are pre-computed per Gaussian for simplicity in this stub.
    normals = np.zeros((N, 3)) # Placeholder for normal computation
    
    for c in range(C):
        T = np.ones((H, W)) # Transmittance per pixel
        
        # Get camera projection matrices
        K = cameras[c].K
        viewmat = cameras[c].viewmat
        
        # Project all Gaussians to screen space for this camera
        # This step is optimized in GPU implementations, here shown conceptually
        projected_means = project_3d_to_2d(means, viewmat, K) # Shape [N, 2]
        
        for py in range(H):
            for px in range(W):
                if T[py, px] < TRANSMITTANCE_THRESHOLD:
                    break
                
                for gid in range(N):
                    # tmp coordinates in camera space
                    tmp = transform_point(means[gid], viewmat)
                    
                    if abs(tmp[2]) < 1e-10:
                        continue
                    
                    us = tmp[0] / tmp[2]
                    vs = tmp[1] / tmp[2]
                    
                    # 3D Sigma
                    sigma_3d = us * us + vs * vs
                    
                    # 2D Fallback Sigma
                    mu = means2d_np[c, gid]
                    dx = px - mu[0]
                    dy = py - mu[1]
                    sigma_2d = 2.0 * (dx * dx + dy * dy)
                    
                    # Hybrid Sigma
                    sigma = 0.5 * min(sigma_3d, sigma_2d)
                    
                    if sigma < 0.0:
                        continue
                    
                    alpha = min(opacities[gid] * np.exp(-sigma), MAX_ALPHA)
                    
                    if alpha < ALPHA_THRESHOLD:
                        continue
                    
                    weight = T[py, px] * alpha
                    
                    render_colors[c, py, px] += weight * colors[gid]
                    render_normals[c, py, px] += weight * normals[gid]
                    
                    T[py, px] *= (1.0 - alpha)
                
                render_alphas[c, py, px, 0] = 1.0 - T[py, px]
        
        if backgrounds is not None:
            bg_np = backgrounds[c]
            render_colors[c] += (1.0 - render_alphas[c]) * bg_np
            
    return render_colors, render_normals, render_alphas

def project_3d_to_2d(means, viewmat, K):
    """Project 3D means to 2D screen coordinates."""
    # Homogeneous coordinates
    ones = np.ones((means.shape[0], 1))
    means_homo = np.hstack([means, ones])
    
    # View transform
    means_view = (viewmat @ means_homo.T).T[:, :3]
    
    # Projection
    means_proj = (K @ means_view.T).T
    us = means_proj[:, 0] / means_proj[:, 2]
    vs = means_proj[:, 1] / means_proj[:, 2]
    
    return np.column_stack([us, vs])

def transform_point(point, viewmat):
    """Transform a single 3D point to camera space."""
    point_homo = np.append(point, 1.0)
    transformed = viewmat @ point_homo
    return transformed[:3]
```

**Edge Cases and Pitfalls:**
- **Precision**: Division by zero in $tmp[2]$ is handled by the $10^{-10}$ threshold [S1, unofficial].
- **Boundaries**: Screen-space coordinates $u_s, v_s$ can be large for distant Gaussians, leading to large $\sigma_{3d}$. The fallback $\sigma_{2d}$ helps mitigate this by using pre-computed means [S1, unofficial].
- **Degenerate Input**: If $\sigma < 0.0$, the Gaussian is skipped, preventing invalid alpha calculations [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(C \cdot H \cdot W \cdot N)$ in the naive implementation, where $C$ is cameras, $H, W$ are resolution, $N$ is number of Gaussians. Optimizations like spatial hashing or tile-based rasterization are not detailed in the source but are common in production implementations [S1, unofficial].
- **Memory Complexity**: $O(C \cdot H \cdot W)$ for output buffers, plus $O(N)$ for Gaussian parameters. Pre-computed `means2d_np` adds $O(C \cdot N)$ memory [S1, unofficial].
- **Performance**: The hybrid sigma calculation adds overhead compared to pure 2D splatting but improves accuracy. The use of pre-computed 2D means reduces per-pixel computation for the fallback sigma [S1, unofficial].
- **Strengths**: Robust alpha estimation, handles extreme viewing angles better than pure 3D or 2D methods [S1, unofficial].
- **Limitations**: Requires pre-computation of 2D means, which adds a preprocessing step. The naive loop structure is inefficient for large $N$ [S1, unofficial].

## Background
This method is part of the Gaussian Splatting rendering pipeline, specifically designed to improve the differentiability and accuracy of the rasterization step [S1, unofficial]. It is used in the `gsplat-mlx` library, an unofficial implementation for the MLX framework [S1, unofficial]. The approach addresses limitations in standard 3D Gaussian Splatting where alpha attenuation might not accurately reflect the true overlap of Gaussians in screen space [S1, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
