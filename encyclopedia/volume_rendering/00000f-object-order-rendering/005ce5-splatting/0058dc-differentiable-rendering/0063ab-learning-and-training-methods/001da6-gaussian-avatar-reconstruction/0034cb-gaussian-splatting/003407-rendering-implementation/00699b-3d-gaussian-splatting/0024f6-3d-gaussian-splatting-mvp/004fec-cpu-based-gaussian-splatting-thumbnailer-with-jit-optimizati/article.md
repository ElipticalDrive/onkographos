# CPU-based Gaussian Splatting Thumbnailer with JIT Optimization

Also called: JIT Python Gaussian Splatting, Sphere Splatting

## Overview
This method provides a lightweight, CPU-only pipeline for generating rapid visual thumbnails from 3D Gaussian Splatting datasets, prioritizing responsiveness and portability over high-fidelity rendering [S1]. It distinguishes itself from standard GPU-accelerated splatting by omitting rotation parameters, approximating Gaussians as spheres, and leveraging Python's Numba JIT compiler to achieve near-real-time performance on constrained hardware [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of 3D Gaussians defined by position ($\mu$), scale ($\sigma$), and color ($\alpha$). Rotation data is explicitly omitted [S1].
*   **Output:** A 2D image file (thumbnail) representing the projected scene [S1].

**Data Structures**
*   **Gaussian Primitive:** A structure containing:
    *   `position`: 3D vector $(x, y, z)$ [S1].
    *   `scale`: Scalar value $\sigma$ representing the radius [S1].
    *   `color`: Color value (e.g., RGB) [S1].
*   **Buffers:**
    *   `color_buffer`: 2D array of shape $(H, W, 3)$ initialized to zero [S1].
    *   `depth_buffer`: 2D array of shape $(H, W)$ initialized to infinity [S1].

**Parameters**
*   **Splat Size ($\sigma$):** The radius of the spherical approximation. Values between 0.4 and 0.5 balance detail and speed [S1].
*   **Image Resolution:** 720p (1280x720) is the standard target resolution [S1].
*   **Initial Radius:** Set to the largest Gaussian scale in the dataset [S1].

**Algorithm Steps**
1.  **Initialization:** Initialize `color_buffer` and `depth_buffer` for the target resolution [S1].
2.  **Sphere Approximation:** Treat each Gaussian as a sphere. The rotation component is ignored, and the Gaussian is defined solely by its center $\mu$ and radius $\sigma$ [S1].
3.  **Projection:** Project each Gaussian's 3D center $\mu$ onto the 2D image plane to obtain screen-space coordinates $(x, y)$ [S1].
4.  **Weight Calculation:** Compute the Gaussian weight $\rho(x)$ for pixels within the splat's radius using the formula:
    $$ \rho(x) = \alpha \cdot \exp\left(-0.5 \cdot (x - \mu)^T \Sigma^{-1} (x - \mu)\right) $$
    In the spherical approximation, $\Sigma$ is treated as a diagonal matrix derived from the scalar scale $\sigma$ [S1].
5.  **Blending:** Blend the projected Gaussians into the `color_buffer` based on their calculated weights $\rho(x)$ using a fast approximate blending method [S1].
6.  **Buffer Update:** Update the `color_buffer` with the blended colors and the `depth_buffer` with the corresponding depth values for each pixel [S1].
7.  **Output:** Save the final `color_buffer` as an image file [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from numba import njit

@njit
def compute_gaussian_weight(x, y, mu_x, mu_y, sigma):
    """
    Compute the weight of a spherical Gaussian at pixel (x, y).
    mu_x, mu_y: Center of the Gaussian in screen space.
    sigma: Radius of the sphere.
    alpha: Opacity/Color intensity (scalar for simplicity).
    """
    # Distance squared from center
    dist_sq = (x - mu_x)**2 + (y - mu_y)**2
    
    # Exponential term: exp(-0.5 * (dist^2 / sigma^2))
    # Note: For a sphere, the covariance matrix inverse is (1/sigma^2) * I
    exponent = -0.5 * (dist_sq / (sigma**2))
    
    # Clamp exponent to avoid underflow issues if necessary, though exp handles it
    return np.exp(exponent)

@njit
def render_thumbnail(gaussians, width, height, sigma_scale_factor=0.45):
    """
    Render a thumbnail using CPU-based sphere splatting.
    
    gaussians: List of tuples (x, y, z, sigma, r, g, b)
    width: Image width
    height: Image height
    sigma_scale_factor: Multiplier for the Gaussian radius (0.4-0.5 recommended)
    """
    # Initialize buffers
    color_buffer = np.zeros((height, width, 3), dtype=np.float32)
    depth_buffer = np.full((height, width), np.inf, dtype=np.float32)
    
    # Sort Gaussians by depth (far to near) for correct alpha blending
    # [G] Standard alpha blending requires back-to-front sorting.
    sorted_gaussians = sorted(gaussians, key=lambda g: g[2], reverse=True)
    
    for g in sorted_gaussians:
        gx, gy, gz, sigma, r, g_val, b = g
        
        # Project to screen space (simplified orthographic/perspective projection)
        # [G] Assuming a standard pinhole camera model where x_screen = fx * X/Z + cx
        # This is a placeholder for the specific projection matrix used in the source.
        # The source implies projection happens but does not give the matrix.
        # We assume gx, gy are already in normalized device coordinates or screen pixels.
        # For implementation, one must apply the camera projection matrix here.
        
        # Determine bounding box for the splat
        radius = sigma * sigma_scale_factor
        min_x = max(0, int(gx - radius))
        max_x = min(width - 1, int(gx + radius))
        min_y = max(0, int(gy - radius))
        max_y = min(height - 1, int(gy + radius))
        
        # Precompute inverse depth for blending
        inv_depth = 1.0 / gz if gz > 0 else 0.0
        
        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                # Calculate weight
                weight = compute_gaussian_weight(x, y, gx, gy, radius)
                
                # Fast approximate blending
                # [G] Standard alpha blending: C_out = C_in * (1 - alpha) + C_new * alpha
                # The source mentions "fast approximate blending" [S1].
                # We use standard alpha blending here as the approximation.
                
                current_depth = depth_buffer[y, x]
                
                # Only update if this Gaussian is closer than the current pixel depth
                if gz < current_depth:
                    # Update depth
                    depth_buffer[y, x] = gz
                    
                    # Update color
                    # [G] Assuming weight acts as alpha for the splat contribution
                    alpha = weight
                    color_buffer[y, x, 0] = color_buffer[y, x, 0] * (1 - alpha) + r * alpha
                    color_buffer[y, x, 1] = color_buffer[y, x, 1] * (1 - alpha) + g_val * alpha
                    color_buffer[y, x, 2] = color_buffer[y, x, 2] * (1 - alpha) + b * alpha

    return color_buffer
```

**Edge Cases and Pitfalls**
*   **Rotation Omission:** By ignoring rotation, the method loses orientation fidelity, which may cause visual artifacts for elongated or rotated structures [S1].
*   **Sphere Approximation:** Treating Gaussians as spheres simplifies the covariance matrix inversion but reduces accuracy for non-spherical primitives [S1].
*   **Python Performance:** Without JIT, pure Python loops are prohibitively slow (336 s vs 2.33 s with JIT) [S1].
*   **Parallel Overhead:** Parallel processing was deliberately avoided to prevent caching and management overhead on systems with limited core counts [S1].

## Operating characteristics
*   **Performance:** JIT compilation reduces execution time by 156× compared to pure Python (from 336 s to 2.33 s) [S1]. A 720p thumbnail generation takes approximately 1.8 s with optimized parameters [S1].
*   **Memory:** The method is designed to be lightweight, operating entirely in CPU memory without GPU dependency [S1].
*   **Strengths:** High portability across devices, including resource-constrained environments like field operations or criminal investigations [S1]. No GPU required [S1].
*   **Limitations:** Lower visual fidelity compared to high-fidelity Gaussian Splatting due to omitted rotation and spherical approximation [S1]. Limited by Python’s inherent performance ceiling even with JIT [S1].
*   **When to Use:** Rapid previewing, field investigation, or batch visualization in constrained environments where GPU access is unavailable [S1].
*   **When Not to Use:** High-fidelity rendering or applications requiring precise orientation and shape representation [S1].

## Background
*   **Origin:** The method was introduced by E. Genchev et al. in 2025 [S1].
*   **History:** The concept of "Sphere Splatting" is inspired by "Particle Splatting" explored by Adams and Lenaert [10], which proposed using spheres to efficiently visualize particle-based simulation data [S1].
*   **Context:** It addresses the computational demands of 3D Gaussian Splatting on CPU-based systems, aiming to make the technique viable for rapid visualization without dedicated graphics acceleration [S1].

## Sources
[S1] Utilizing JIT Python runtime and parameter optimization for CPU-based Gaussian Splatting thumbnailer, https://zenodo.org/records/19676358, fetched 2026-10-26
