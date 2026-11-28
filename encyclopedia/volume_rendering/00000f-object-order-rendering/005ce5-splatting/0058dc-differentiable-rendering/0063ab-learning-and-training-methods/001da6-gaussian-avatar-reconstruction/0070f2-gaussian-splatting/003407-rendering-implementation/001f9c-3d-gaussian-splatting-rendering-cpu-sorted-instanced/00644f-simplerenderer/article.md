# SimpleRenderer

Also called: CPU-based Gaussian Splatting Renderer, Painter's Algorithm Gaussian Renderer

## Overview
SimpleRenderer is a CPU-based renderer for 3D Gaussian Splatting that projects 3D Gaussian distributions into 2D screen-space elliptical splats and composites them using a standard painter's algorithm with alpha blending. It performs all projection, sorting, and rasterization on the CPU using standard array operations, making it suitable for debugging, thumbnail generation, or environments lacking specialized graphics hardware.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians defined by position, color, scale, and opacity; a camera model providing projection matrices and intrinsics ($f_x$); and configuration parameters (`near_plane`, `far_plane`, `alpha_threshold`, `background_color`) [S1, unofficial].
*   **Outputs:** A 2D image of shape $(H, W, 3)$ with `float32` dtype.

**Data Structures:**
*   **Gaussian Attributes:** Arrays for positions $(N, 3)$, colors $(N, 3)$, scales $(N, 3)$, and opacities $(N, 1)$ [G].
*   **Image Buffer:** A 3D array `img` of shape $(H, W, 3)$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Create an image buffer `img` of shape $(H, W, 3)$ initialized to `background_color` [S1, unofficial].
2.  **Camera Transformation:** Transform Gaussian positions to camera coordinates using the camera's world-to-camera transformation. Extract the depth values $z_{vals}$ from the third component of the transformed points [S1, unofficial].
3.  **Filtering:** Filter Gaussians to retain only those where $near\_plane \le z_{vals} \le far\_plane$ and opacity $> alpha\_threshold$ [S1, unofficial].
4.  **Projection:** Project the remaining Gaussian centers to pixel coordinates $(u, v)$ using the camera's camera-to-pixel transformation [S1, unofficial].
5.  **Sorting:** Sort the filtered Gaussians by depth $z_{vals}$ in descending order (far to near) to implement the painter's algorithm [S1, unofficial].
6.  **Rendering Loop:** For each Gaussian $i$ in the sorted list:
    a.  **Radius Calculation:** Calculate the screen-space radius $r$. Let $s_{mean}$ be the mean of the Gaussian's scales. Compute:
        $$ r = \min(15, \max(1, \lfloor s_{mean} \cdot \frac{f_x}{\max(z_{vals}[i], 0.1)} \rfloor)) $$
        The depth is clamped to a minimum of 0.1 to prevent division by zero or excessively large radii [S1, unofficial].
    b.  **Bounding Box:** Define the pixel bounding box for the splat:
        $$ y_{min} = \max(0, \lfloor v \rfloor - r) $$
        $$ y_{max} = \min(H, \lfloor v \rfloor + r + 1) $$
        $$ x_{min} = \max(0, \lfloor u \rfloor - r) $$
        $$ x_{max} = \min(W, \lfloor u \rfloor + r + 1) $$
        If the bounding box is invalid (e.g., $u$ or $v$ is far outside bounds such that the box is empty), skip the Gaussian.
    c.  **Weight Calculation:** For each pixel $(x, y)$ in the bounding box, compute the normalized squared distance from the center:
        $$ dist\_sq = \frac{(x - u)^2 + (y - v)^2}{r^2 + 10^{-8}} $$
        Calculate the Gaussian weight:
        $$ w = \exp(-dist\_sq) \cdot \alpha_i $$
    d.  **Blending:** Update the image buffer using alpha blending for each color channel $c \in \{0, 1, 2\}$:
        $$ img[y, x, c] = img[y, x, c] \cdot (1 - w) + C_i[c] \cdot w $$
        where $C_i[c]$ is the $c$-th component of the Gaussian's color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def simple_renderer(gaussians, camera, config, H, W):
    """
    Render 3D Gaussians using a CPU-based painter's algorithm.
    
    Args:
        gaussians: Dict with keys 'pos' (N,3), 'color' (N,3), 'scale' (N,3), 'opacity' (N,1)
        camera: Object with methods world_to_camera, camera_to_pixel, and attribute fx
        config: Object with attributes near_plane, far_plane, alpha_threshold, background_color
        H: Image height
        W: Image width
        
    Returns:
        img: (H, W, 3) float32 array
    """
    # 1. Initialize image
    img = np.full((H, W, 3), config.background_color, dtype=np.float32)
    
    # 2. Transform to camera space
    cam_points = camera.world_to_camera(gaussians['pos']) # (N, 3)
    z_vals = cam_points[:, 2] # (N,)
    
    # 3. Filter Gaussians
    valid_mask = (
        (z_vals >= config.near_plane) & 
        (z_vals <= config.far_plane) & 
        (gaussians['opacity'].flatten() > config.alpha_threshold)
    )
    
    if not np.any(valid_mask):
        return img
        
    filtered_pos = cam_points[valid_mask]
    filtered_color = gaussians['color'][valid_mask]
    filtered_scale = gaussians['scale'][valid_mask]
    filtered_opacity = gaussians['opacity'][valid_mask].flatten()
    filtered_z = z_vals[valid_mask]
    
    # 4. Project to pixel coordinates
    pixel_coords = camera.camera_to_pixel(filtered_pos) # (M, 2), (u, v)
    u = pixel_coords[:, 0]
    v = pixel_coords[:, 1]
    
    # 5. Sort by depth (far to near)
    sort_indices = np.argsort(filtered_z)[::-1]
    u = u[sort_indices]
    v = v[sort_indices]
    filtered_color = filtered_color[sort_indices]
    filtered_opacity = filtered_opacity[sort_indices]
    filtered_scale = filtered_scale[sort_indices]
    filtered_z = filtered_z[sort_indices]
    
    # 6. Render loop
    fx = camera.fx
    
    for i in range(len(filtered_z)):
        z = filtered_z[i]
        # Clamp z to prevent division by zero
        z_clamped = max(z, 0.1)
        
        # Calculate radius
        s_mean = np.mean(filtered_scale[i])
        radius = int(s_mean * fx / z_clamped)
        radius = max(1, min(radius, 15)) # Cap at 15
        
        # Bounding box
        y_min = max(0, int(v[i]) - radius)
        y_max = min(H, int(v[i]) + radius + 1)
        x_min = max(0, int(u[i]) - radius)
        x_max = min(W, int(u[i]) + radius + 1)
        
        # Skip if out of bounds
        if y_min >= y_max or x_min >= x_max:
            continue
            
        # Precompute weight factor
        alpha = filtered_opacity[i]
        color = filtered_color[i]
        
        # Create meshgrid for bounding box
        ys, xs = np.mgrid[y_min:y_max, x_min:x_max]
        
        # Normalized squared distance
        dist_sq = ((xs - u[i])**2 + (ys - v[i])**2) / (radius**2 + 1e-8)
        
        # Gaussian weight
        w = np.exp(-dist_sq) * alpha
        
        # Blend
        img[y_min:y_max, x_min:x_max, 0] = img[y_min:y_max, x_min:x_max, 0] * (1 - w) + color[0] * w
        img[y_min:y_max, x_min:x_max, 1] = img[y_min:y_max, x_min:x_max, 1] * (1 - w) + color[1] * w
        img[y_min:y_max, x_min:x_max, 2] = img[y_min:y_max, x_min:x_max, 2] * (1 - w) + color[2] * w
        
    return img
```

**Parameters:**
*   `config.near_plane`: Minimum depth for valid Gaussians [S1, unofficial].
*   `config.far_plane`: Maximum depth for valid Gaussians [S1, unofficial].
*   `config.alpha_threshold`: Minimum opacity for a Gaussian to be considered valid [S1, unofficial].
*   `config.background_color`: RGB array of shape (3,) used to initialize the image canvas [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If no Gaussians pass the filtering stage, the function returns the background-initialized image immediately [S1, unofficial].
*   If a Gaussian's projected center $(u, v)$ is outside the image bounds, the bounding box calculation may result in an empty range; the implementation must skip such Gaussians.
*   The radius is capped at 15 to avoid excessive computational overhead for large splats [S1, unofficial].
*   Depth $z_{vals}[i]$ is clamped to a minimum of 0.1 in the radius calculation to prevent division by zero or extremely large radii [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for sorting, plus $O(M \cdot R^2)$ for rasterization, where $N$ is the total number of Gaussians, $M$ is the number of visible Gaussians, and $R$ is the average radius (capped at 15) [G].
*   **Memory Complexity:** $O(N)$ to store Gaussian attributes and intermediate arrays [G].
*   **Performance:** Suitable for low-resolution outputs or small scenes due to CPU-bound processing. Not suitable for real-time rendering or high-resolution video generation without significant optimization [G].
*   **Strengths:** Simple implementation, easy to debug, no GPU dependency [G].
*   **Limitations:** Slow for large scenes, no support for differentiable rendering, limited by CPU cache and memory bandwidth [G].

## Background
SimpleRenderer is an implementation detail found in the `GaussianScan-Recon` repository, an unofficial project for 3D Gaussian Splatting reconstruction [S1, unofficial]. It serves as a basic rendering backend for generating thumbnails or debugging visualizations during the reconstruction process.

## Sources
[S1] Laityperfect7/GaussianScan-Recon, https://github.com/Laityperfect7/GaussianScan-Recon, fetched 2026-10-27 (unofficial)
