# CPU-based Gaussian Splatting Thumbnailer

Also called: 3D Gaussian Splat CPU Rasterization, 3DGS Thumbnail, CPU Gaussian Splat Rendering, CPU Gaussian Splat Thumbnail Rendering, Gaussian Splat Thumbnail Rendering, Splat Thumbnail Generation

## Overview
This method generates low-resolution preview images (thumbnails) from trained 3D Gaussian Splatting scenes using a pure CPU pipeline optimized for speed via Just-In-Time (JIT) compilation [S1]. It solves the problem of slow interactive preview generation by simplifying the rendering model—omitting Gaussian rotation and complex shading—and leveraging Numba to accelerate Python-based numerical loops [S1]. It differs from its parent, [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md), by explicitly excluding rotation data, using a simplified approximate blending function, and processing a fixed opacity-filtered budget of splats rather than full alpha compositing [S1][S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:** Scene files in formats `.ply`, `.splat`, `.spz`, `.sog`, `.ksplat`, `.lcc`, or `.compressed.ply` [S2, unofficial]. Fixed camera parameters: position, target, and field of view (FOV) [S1]. Parameters: Gaussian splat size scalar $\sigma$, output resolution, opacity threshold, and splat budget [S1][S2, unofficial].
**Outputs:** A PNG image file representing the rendered thumbnail [S1].

### Data Structures
Scene data is parsed into parallel arrays for `position` (3 floats), `scale` (3 floats), and `f_dc` (degree-0 spherical harmonic coefficient, 3 floats) [S1][S2, unofficial]. Rotation data is explicitly omitted from the input structure [S1]. Format routing inspects PLY headers or file extensions to select native `numpy.fromfile()` decoding or CPU conversion via the PlayCanvas `splat-transform` CLI [S2, unofficial].

### Steps
1. **Camera Setup:** Compute fixed view ($V$) and projection ($P$) matrices from the camera position, target, and FOV [S1].
2. **Format Conversion & Coordinate Normalization:** Convert input formats to a standard 3DGS PLY representation; rotate scenes 180° about the Z-axis to convert the original COLMAP Y-down convention to Y-up [S2, unofficial].
3. **Visible Splat Extraction & Filtering:** Identify Gaussians falling within the camera’s frustum [S1]. Compute opacity $O = \text{sigmoid}(opacity)$ and discard any splat where $O < 0.2$ [S2, unofficial].
4. **Budget Sampling:** Sample the remaining splats to a target budget (1 million splats per vCPU, maximum 5 million total) with probability proportional to their opacity $O$ [S2, unofficial].
5. **Splat Sorting:** Sort the visible splats by their camera-space $z$-depth in back-to-front order (farthest first) to ensure proper front-to-back blending [S1].
6. **Projection & Rasterization:** Project each Gaussian center onto the image plane using $V$ and $P$; treat Gaussians as simple, rotation-less spheres [S1]. Compute footprint radius $R = 2 \times \exp(\max(scale_*))$ [S2, unofficial]. Rasterize using a fast approximate blending method where color contribution is computed based on a simplified weighting function considering distance from splat center and current depth values [S1].
7. **Buffer Update & Output:** Update color and depth buffers with blended colors and depth values as each Gaussian is blended; save the color buffer as a PNG image file [S1].

### Formulas
$$C = 0.5 + 0.28209479 \times f_{dc}$$
where $C$ is the output color and $f_{dc}$ is the degree-0 spherical harmonic coefficient [S2, unofficial].

$$O = \text{sigmoid}(opacity)$$
where $O$ is the computed opacity used for filtering and sampling [S2, unofficial].

$$R = 2 \times \exp(\max(scale_*))$$
where $R$ is the footprint radius in pixels and $scale_*$ represents the scale components [S2, unofficial].

### Parameters and Typical Values
*   $\sigma$ (Splat Size): Scalar controlling spatial extent. Tested values: $\{0.1, 0.2, 0.4, 0.5, 0.7, 1.0\}$ [S1]. Smaller values preserve detail but cause fragmentation; larger values increase coverage but cause blurring [S1].
*   Resolution: Discrete variable. Tested values: 480p ($640 \times 480$), 720p ($960 \times 720$), 1080p ($1440 \times 1080$) [S1].
*   Opacity Threshold: Splats with opacity below $0.2$ are dropped [S2, unofficial].
*   Splat Budget: 1 million splats per vCPU, capped at 5 million total [S2, unofficial].
*   Footprint Multiplier: The footprint radius is twice the exponential of the maximum scale component [S2, unofficial].

### Edge Cases and Pitfalls
*   **Python Loop Bottleneck:** Standard Python interpretation is too slow for the rasterization loop; mitigated by using Numba for JIT compilation of performance-critical numerical routines [S1].
*   **Parallel Processing:** Deliberately avoided due to caching and computation management overhead on systems with limited core counts [S1].
*   **Visual Eligibility:** Omitting rotation data reduces computational cost but may impact visual quality; however, evaluations showed negligible impact on thumbnail eligibility [S1].
*   **Security/Isolation:** The `splat-transform` CLI is executed with strict constraints: only an allowlisted local file inside the work directory is accessible, no shell is used, and a timeout is applied [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from numba import njit

@njit
def compute_footprint_radius(scale):
    """R = 2 * exp(max(scale_*)) [S2, unofficial]"""
    return 2.0 * np.exp(np.max(scale))

@njit
def approximate_weight(dist_sq, footprint_radius):
    """Simplified Gaussian weighting function considering distance [S1]"""
    return np.exp(-dist_sq / (2.0 * footprint_radius * footprint_radius))

@njit
def render_thumbnail(positions, scales, f_dc, depths, width, height, sigma):
    """
    Renders a thumbnail using CPU-sorted, rotation-less Gaussian splats.
    positions: (N, 3) array of camera-space centers
    scales: (N, 3) array of scale components
    f_dc: (N, 3) array of degree-0 spherical harmonic coefficients
    depths: (N,) array of camera-space z-depths (sorted back-to-front)
    """
    color_buffer = np.zeros((height, width, 3), dtype=np.float32)
    depth_buffer = np.full((height, width), np.inf, dtype=np.float32)

    for i in range(len(positions)):
        pos = positions[i]
        scale = scales[i]
        color = 0.5 + 0.28209479 * f_dc[i]
        depth = depths[i]

        # Project center to screen space [G]
        ndc_x = (pos[0] * 1.0)
        ndc_y = (pos[1] * 1.0)
        screen_x = (ndc_x + 1.0) * 0.5 * width
        screen_y = (1.0 - (ndc_y + 1.0) * 0.5) * height

        radius = compute_footprint_radius(scale)
        if radius < 1.0:
            radius = 1.0

        # Determine bounding box covering the splat circle
        x_min = int(max(0, screen_x - radius))
        x_max = int(min(width - 1, screen_x + radius))
        y_min = int(max(0, screen_y - radius))
        y_max = int(min(height - 1, screen_y + radius))

        for y in range(y_min, y_max + 1):
            for x in range(x_min, x_max + 1):
                dx = x - screen_x
                dy = y - screen_y
                dist_sq = dx * dx + dy * dy
                w = approximate_weight(dist_sq, radius)

                # Buffer Update: quick composition without expensive memory operations [S1]
                if depth < depth_buffer[y, x]:
                    alpha = w
                    color_buffer[y, x, 0] = color_buffer[y, x, 0] * (1.0 - alpha) + color[0] * alpha
                    color_buffer[y, x, 1] = color_buffer[y, x, 1] * (1.0 - alpha) + color[1] * alpha
                    color_buffer[y, x, 2] = color_buffer[y, x, 2] * (1.0 - alpha) + color[2] * alpha
                    depth_buffer[y, x] = depth

    return color_buffer
```

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** Dominated by the rasterization loop over visible splats. Complexity is roughly $O(N_{visible} \cdot A_{splat})$, where $N_{visible}$ is the number of visible Gaussians and $A_{splat}$ is the average area of a splat in pixels [S1]. JIT compilation reduces the constant factor significantly [S1].
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes and $O(W \cdot H)$ for the color and depth buffers [S1].

### Measured Performance
*   **Hardware:** AMD Ryzen Threadripper PRO 5995WX, 16 GB RAM, Arch Linux Kernel 6.9.1 [S1].
*   **Software:** Python 3.13.7, NumPy 2.1.3, Numba 0.61.0 [S1].
*   **JIT vs Non-JIT:** JIT acceleration provides a ~156× speed-up (2.15s vs 336s for $\sigma=0.7$, fixed resolution) [S1].
*   **Execution Times (JIT Enabled):**
    *   $\sigma=0.1$: 0.9s (480p), 0.94s (720p), 1.29s (1080p)
    *   $\sigma=0.2$: 0.97s (480p), 1.11s (720p), 1.87s (1080p)
    *   $\sigma=0.4$: 1.03s (480p), 1.42s (720p), 2.99s (1080p)
    *   $\sigma=0.5$: 1.29s (480p), 1.87s (720p), 4.20s (1080p)
    *   $\sigma=0.7$: 2.15s (480p), 3.65s (720p), 7.11s (1080p)
    *   $\sigma=1.0$: 3.66s (480p), 6.42s (720p), 13.31s (1080p)
    [S1]

### Strengths and Limitations
*   **Strengths:** Extremely fast execution on CPU due to JIT compilation and simplified rendering model (no rotation, approximate blending) [S1].
*   **Limitations:** Lower visual fidelity compared to full GPU splatting or rotation-aware CPU rendering; fragmentation at small $\sigma$, blurring at large $\sigma$ [S1].

### When to Use
*   Use when rapid thumbnail generation is required on CPU-only systems or when GPU resources are unavailable [S1].
*   Use when visual fidelity is secondary to speed (e.g., scene browsing, preview generation) [S1].

## Background

### Origin and History
The method was developed by E. Genchev et al. and published in *Array* (Volume 28, 2025, 100611) [S1]. It was designed to address the computational intensity of 3D Gaussian Splatting rendering by creating a lightweight, CPU-based thumbnailer [S1]. The work focuses on parameter optimization ($\sigma$, resolution) and the use of JIT compilation to mitigate Python's performance limitations [S1].

### Usage
The method was evaluated by colleagues with backgrounds in computer graphics. The most preferred configurations were $\sigma=0.4$ and $\sigma=0.5$ at 720p resolution, balancing visual clarity and rendering performance [S1].

## Sources

[S1] Utilizing JIT Python runtime and parameter optimization for CPU-based Gaussian Splatting thumbnailer, https://zenodo.org/records/19676358, fetched 2026-10-26
[S2] Render Gaussian splat thumbnails on the CPU ( #469 ), https://github.com/awslabs/visual-asset-management-system/commit/770720200d6f1a903fa230d9b28080f93b3573b3, fetched 2026-10-29 (unofficial)
