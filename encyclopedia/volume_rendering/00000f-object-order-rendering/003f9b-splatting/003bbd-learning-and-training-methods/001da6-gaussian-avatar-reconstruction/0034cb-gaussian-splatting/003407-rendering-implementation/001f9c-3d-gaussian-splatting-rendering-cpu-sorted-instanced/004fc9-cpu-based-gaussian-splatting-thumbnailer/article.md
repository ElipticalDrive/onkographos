# CPU-based Gaussian Splatting Thumbnailer

Also called: JIT-accelerated CPU Gaussian Thumbnailer, Rotation-less Gaussian Splatting Thumbnailer

## Overview
This method generates low-resolution preview images (thumbnails) from trained 3D Gaussian Splatting scenes using a pure CPU pipeline optimized for speed via Just-In-Time (JIT) compilation. It solves the problem of slow interactive preview generation by simplifying the rendering model—omitting Gaussian rotation and complex shading—and leveraging Numba to accelerate Python-based numerical loops. It differs from its parent, [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md), by explicitly excluding rotation data and using a simplified, approximate blending function rather than full alpha compositing, prioritizing execution time over visual fidelity.

## How it works

### Inputs and Outputs
**Inputs:**
*   **Scene Data:** A binary file containing 3D Gaussian parameters. Parsed into arrays for `position` (3 floats), `scale` (3 floats), and `color` (3 floats). Rotation data is explicitly excluded from the input structure [S1].
*   **Camera Parameters:** Fixed position, target, and Field of View (FOV) used to compute View ($V$) and Projection ($P$) matrices [S1].
*   **Parameters:**
    *   $\sigma$: Gaussian splat size scalar. Tested values: $\{0.1, 0.2, 0.4, 0.5, 0.7, 1.0\}$ [S1].
    *   Resolution: Output dimensions. Tested values: 480p ($640 \times 480$), 720p ($960 \times 720$), 1080p ($1440 \times 1080$) [S1].

**Outputs:**
*   A PNG image file representing the rendered thumbnail [S1].

### Data Structures
The scene data is loaded using `numpy.fromfile()` and stored in a dictionary-like structure or parallel arrays [S1].
*   `positions`: Array of shape $(N, 3)$, type `float32` or `float64`.
*   `scales`: Array of shape $(N, 3)$, type `float32` or `float64`.
*   `colors`: Array of shape $(N, 3)$, type `float32` or `float64`.
*   `visible_indices`: Array of indices identifying Gaussians within the camera frustum [S1].
*   `sorted_indices`: Array of indices sorted by camera-space depth [S1].
*   `color_buffer`: 2D array of shape $(H, W, 3)$, initialized to black.
*   `depth_buffer`: 2D array of shape $(H, W)$, initialized to infinity.

### Steps

1.  **Camera Setup:** Compute the View matrix $V$ and Projection matrix $P$ from the fixed camera position, target, and FOV [S1].
2.  **Visible Splat Extraction:** Transform Gaussian positions into camera space. Identify Gaussians falling within the camera’s frustum. Store their indices in `visible_indices` [S1].
3.  **Splat Sorting:** Sort the visible Gaussians by their camera-space $z$-depth in back-to-front order (farthest first). This ensures that closer splats are blended on top of farther ones [S1]. Store indices in `sorted_indices`.
4.  **Projection and Rasterization Loop:** Iterate through `sorted_indices`. For each Gaussian $i$:
    a.  **Project Center:** Transform the Gaussian center $c_i$ to screen space $(u_i, v_i)$ using $V$ and $P$ [S1].
    b.  **Calculate Splat Radius:** Compute the radius $r_i$ of the splat in pixels. The formula uses the largest component of the 3D scale vector, the depth $z_i$, the image width $W$, and the scalar $\sigma$ [S1]:
        $$ r_i = \sigma \cdot \max(s_{i,x}, s_{i,y}, s_{i,z}) \cdot \frac{z_i}{W} $$
        *Note: The source implies a minimum size of 1 pixel is enforced [S1].*
    c.  **Determine Bounding Box:** Calculate the pixel coordinates $(u_{min}, v_{min})$ to $(u_{max}, v_{max})$ covering the splat circle. Clip to image bounds $[0, W-1] \times [0, H-1]$ [S1].
    d.  **Rasterize and Blend:** For each pixel $(x, y)$ in the bounding box:
        i.  Calculate distance $d$ from pixel center to splat center $(u_i, v_i)$ [S1].
        ii. Compute weight $w$ based on distance and depth. The source describes a "fast approximate blending method" and a "simplified weighting function considering distance from splat center and current depth values" [S1]. It explicitly avoids complex lighting/shading [S1]. The exact mathematical formula for the weight function $w(d, z)$ is not provided in the text, only that it depends on $d$ and $z$ [S1].
        iii. Update the `color_buffer` and `depth_buffer` at $(x, y)$ using the computed weight and the Gaussian's color [S1]. The update allows quick composition without expensive memory operations [S1].
5.  **Image Output:** Save the `color_buffer` as a PNG file [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from numba import njit

@njit
def compute_splat_radius(scale_z, depth, width, sigma):
    """
    Computes the radius of a Gaussian splat in pixels.
    scale_z: max component of the 3D scale vector
    depth: z-distance from camera
    width: image width in pixels
    sigma: global scaling factor
    """
    # Formula derived from Eq. 2 in [S1]
    # Ensures minimum size of 1 pixel is maintained implicitly or via clamp
    radius = sigma * scale_z * (depth / width)
    return radius

@njit
def render_thumbnail(positions, scales, colors, visible_indices, 
                     view_matrix, proj_matrix, width, height, sigma):
    """
    Renders a thumbnail using CPU-sorted, rotation-less Gaussian splats.
    """
    # Initialize buffers
    color_buffer = np.zeros((height, width, 3), dtype=np.float32)
    depth_buffer = np.full((height, width), np.inf, dtype=np.float32)
    
    # Sort visible splats by depth (back-to-front)
    # Note: In a real implementation, this requires extracting depths and sorting indices
    # Here we assume sorted_indices is passed or computed externally
    # For the sake of self-contained pseudocode, we assume 'visible_indices' 
    # are already sorted by depth.
    
    for idx in visible_indices:
        pos = positions[idx]
        scale = scales[idx]
        color = colors[idx]
        
        # 1. Project center to camera space
        # Homogeneous coordinates
        pos_h = np.array([pos[0], pos[1], pos[2], 1.0])
        cam_pos = view_matrix @ pos_h
        cam_pos_h = proj_matrix @ cam_pos
        
        # Perspective divide
        if cam_pos_h[3] == 0:
            continue
            
        ndc_x = cam_pos_h[0] / cam_pos_h[3]
        ndc_y = cam_pos_h[1] / cam_pos_h[3]
        depth = cam_pos_h[2] # Camera-space Z
        
        # Convert NDC to screen coordinates
        screen_x = (ndc_x + 1.0) * 0.5 * width
        screen_y = (1.0 - (ndc_y + 1.0) * 0.5) * height # Flip Y for image coords
        
        # 2. Calculate radius
        # Scale is rotation-less, so use max component
        max_scale = max(scale[0], scale[1], scale[2])
        radius = compute_splat_radius(max_scale, depth, width, sigma)
        
        # Ensure minimum radius
        if radius < 1.0:
            radius = 1.0
            
        # 3. Determine bounding box
        x_min = int(max(0, screen_x - radius))
        x_max = int(min(width - 1, screen_x + radius))
        y_min = int(max(0, screen_y - radius))
        y_max = int(min(height - 1, screen_y + radius))
        
        # 4. Rasterize
        for y in range(y_min, y_max + 1):
            for x in range(x_min, x_max + 1):
                # Distance from pixel center to splat center
                dx = x - screen_x
                dy = y - screen_y
                dist_sq = dx*dx + dy*dy
                
                # Weight calculation
                # Source [S1] states: "Color contribution is computed based on a 
                # simplified weighting function considering distance from splat 
                # center and current depth values."
                # Exact formula not provided. Standard Gaussian kernel is assumed 
                # for implementation gap [G], but strictly following source, 
                # this is a placeholder for the specific "fast approximate" method.
                # However, to be implementable, we use a standard Gaussian falloff 
                # as the "simplified weighting function" implied by "Gaussian Splatting".
                # Note: The source explicitly omits rotation, so the exponent is 
                # just -dist_sq / (2 * sigma_eff^2).
                
                # Since exact formula is missing, we define the weight w 
                # based on the description: function of distance and depth.
                # A common approximation for splatting weight is exp(-dist^2 / (2*sigma^2)).
                # We will use a standard Gaussian weight for implementability [G].
                # If the source meant a specific linear or other approximation, 
                # it is not detailed.
                
                # Using standard Gaussian weight for implementability [G]
                # sigma_splat is derived from radius. 
                # radius ~ 2*sigma_splat or similar. 
                # Let's assume weight = exp(-dist_sq / (2 * (radius/2)^2)) 
                # or similar standard splatting weight.
                # To be safe and minimal:
                weight = np.exp(-dist_sq / (2.0 * (radius * radius)))
                
                # Update buffers
                # Source [S1]: "Buffer Update: The color and depth buffers are updated 
                # with the blended colors and depth values... quick composition"
                # Standard alpha blending: C_out = C_in * (1-a) + C_new * a
                # Depth update: if new_depth < current_depth, update.
                
                current_depth = depth_buffer[y, x]
                
                if depth < current_depth:
                    # Simple alpha blending based on weight
                    # Note: Source does not specify alpha value, only "blended colors"
                    # Assuming weight acts as alpha or contributes to it [G]
                    alpha = weight
                    
                    # Blend
                    color_buffer[y, x, 0] = color_buffer[y, x, 0] * (1.0 - alpha) + color[0] * alpha
                    color_buffer[y, x, 1] = color_buffer[y, x, 1] * (1.0 - alpha) + color[1] * alpha
                    color_buffer[y, x, 2] = color_buffer[y, x, 2] * (1.0 - alpha) + color[2] * alpha
                    
                    depth_buffer[y, x] = depth
                    
    return color_buffer

def main():
    # 1. Data Ingestion
    # Load from binary file using numpy.fromfile()
    # Parse into positions, scales, colors arrays [S1]
    # Rotation data is excluded [S1]
    
    # 2. Camera Setup
    # Fixed position, target, FOV [S1]
    # Compute View and Projection matrices [S1]
    
    # 3. Visible Splat Extraction
    # Identify Gaussians in frustum [S1]
    
    # 4. Splat Sorting
    # Sort by depth back-to-front [S1]
    
    # 5. Render
    # Call render_thumbnail with parameters sigma, width, height [S1]
    
    # 6. Image Output
    # Save color buffer as PNG [S1]
    pass
```

### Parameters and Typical Values
*   **$\sigma$ (Splat Size):** Scalar controlling spatial extent. Values: $0.1, 0.2, 0.4, 0.5, 0.7, 1.0$. Smaller values preserve detail but cause fragmentation; larger values increase coverage but cause blurring [S1].
*   **Resolution:** Discrete variable. Values: 480p ($640 \times 480$), 720p ($960 \times 720$), 1080p ($1440 \times 1080$) [S1].
*   **Rotation:** Omitted. Gaussians are treated as rotationally invariant spheres [S1].

### Edge Cases and Pitfalls
*   **Python Loop Bottleneck:** Standard Python interpretation is too slow for the rasterization loop. Mitigated by using Numba for JIT compilation [S1].
*   **Parallel Processing:** Deliberately avoided due to caching and computation management overhead on systems with limited core counts [S1].
*   **Visual Eligibility:** Omitting rotation data reduces computational cost but may impact visual quality. However, evaluations showed negligible impact on thumbnail eligibility [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** Dominated by the rasterization loop over visible splats. Complexity is roughly $O(N_{visible} \cdot A_{splat})$, where $N_{visible}$ is the number of visible Gaussians and $A_{splat}$ is the average area of a splat in pixels. JIT compilation reduces the constant factor significantly [S1].
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes and $O(W \cdot H)$ for the color and depth buffers [S1].

### Measured Performance
*   **Hardware:** AMD Ryzen Threadripper PRO 5995WX, 16 GB RAM, Arch Linux Kernel 6.9.1 [S1].
*   **Software:** Python 3.13.7, NumPy 2.1.3, Numba 0.61.0 [S1].
*   **JIT vs Non-JIT:** JIT acceleration provides a ~156x speed-up (2.15s vs 336s for $\sigma=0.7$, fixed resolution) [S1].
*   **Execution Times (JIT Enabled):**
    *   $\sigma=0.1$: 0.9s (480p), 0.94s (720p), 1.29s (1080p)
    *   $\sigma=0.2$: 0.97s (480p), 1.11s (720p), 1.87s (1080p)
    *   $\sigma=0.4$: 1.03s (480p), 1.42s (720p), 2.99s (1080p)
    *   $\sigma=0.5$: 1.29s (480p), 1.87s (720p), 4.20s (1080p)
    *   $\sigma=0.7$: 2.15s (480p), 3.65s (720p), 7.11s (1080p)
    *   $\sigma=1.0$: 3.66s (480p), 6.42s (720p), 13.31s (1080p)
    [S1]

### Strengths and Limitations
*   **Strengths:** Extremely fast execution on CPU due to JIT compilation and simplified rendering model (no rotation, approximate blending). Suitable for generating quick previews [S1].
*   **Limitations:** Lower visual fidelity compared to full GPU splatting or rotation-aware CPU rendering. Fragmentation at small $\sigma$, blurring at large $\sigma$. Not suitable for high-quality final frames [S1].

### When to Use
*   Use when rapid thumbnail generation is required on CPU-only systems or when GPU resources are unavailable.
*   Use when visual fidelity is secondary to speed (e.g., scene browsing, preview generation) [S1].

## Background

### Origin and History
The method was developed by E. Genchev et al. and published in *Array* (Volume 28, 2025, 100611) [S1]. It was designed to address the computational intensity of 3D Gaussian Splatting rendering by creating a lightweight, CPU-based thumbnailer. The work focuses on parameter optimization ($\sigma$, resolution) and the use of JIT compilation to mitigate Python's performance limitations [S1].

### Usage
The method was evaluated by colleagues with backgrounds in computer graphics. The most preferred configurations were $\sigma=0.4$ and $\sigma=0.5$ at 720p resolution, balancing visual clarity and rendering performance [S1].

## Sources

[S1] Utilizing JIT Python runtime and parameter optimization for CPU-based Gaussian Splatting thumbnailer, https://zenodo.org/records/19676358, fetched 2026-10-26
