# Filtered Back-Projection for Volume Rendering

Also called: D2VR, Projection-based Volume Rendering

## Overview
Filtered Back-Projection (FBP) for Volume Rendering, also known as D2VR, is a technique that reconstructs density values and gradients at arbitrary sample points along a ray directly from 2D projections, avoiding the intermediate step of creating a 3D rectilinear grid [S1]. It sets itself apart from grid-based Direct Volume Rendering (DVR) by eliminating interpolation errors associated with voxel sampling, thereby producing higher visual quality for thin structures [S1]. It applies the FBP reconstruction formula as a direct resampling scheme for arbitrary ray positions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $K$ filtered 2D projection images $Q_{\theta_i}$, where $i \in \{1, \dots, K\}$, corresponding to projection angles $\theta_i$ [S1]. Camera parameters defining ray origins and directions [G].
*   **Outputs:** A rendered image with color and opacity values for each pixel [S1].

**Data Structures:**
*   **Projections:** A list of 1D arrays representing the filtered projection data $Q_{\theta_i}$ for each angle [S1]. The values are obtained by high-pass filtering the measured projection data [S1].
*   **Ray State:** For each pixel, a ray origin $O$ and direction $D$. Sample points $P$ along the ray are defined by $P(s) = O + s \cdot D$ [G].

**Parameters:**
*   **Number of Projections ($K$):** The total number of available views [S1].
*   **Step Size ($\Delta s$):** The distance between sample points along the ray. It must be less than half the voxel size of the equivalent grid to ensure accuracy [S1].
*   **Finite Difference Step ($\Delta t$):** A small finite value used to approximate the limit in the gradient formula [S1].

**Algorithm Steps:**

1.  **Ray Casting:** For each pixel of the image plane, cast a ray through the volumetric space enclosed by the filtered projections [S1].
2.  **Sampling:** Generate sample points $P_j$ along the ray at intervals of $\Delta s$ [S1].
3.  **Density Reconstruction:** At each resample location, reconstruct the underlying 3D density value using the discrete FBP formula [S1]:
    $$ \tilde{f}(x, y) = \frac{\pi}{K} \sum_{i=1}^K Q_{\theta_i}(t_i) $$
    where $t_i = x \cos \theta_i + y \sin \theta_i$ is the projected coordinate of the point onto the $i$-th projection plane [S1].
4.  **Gradient Estimation:** Compute the partial derivatives $\tilde{f}_x$ and $\tilde{f}_y$ directly from the projections using Newton's difference quotient [S1]:
    $$ \tilde{f}_x = \frac{\pi}{K} \sum_{i=1}^K \lim_{\Delta x \rightarrow 0} \frac{1}{\Delta x} (Q_{\theta_i}(t_i + \Delta x \cos \theta_i) - Q_{\theta_i}(t_i)) $$
    $$ \tilde{f}_y = \frac{\pi}{K} \sum_{i=1}^K \lim_{\Delta y \rightarrow 0} \frac{1}{\Delta y} (Q_{\theta_i}(t_i + \Delta y \sin \theta_i) - Q_{\theta_i}(t_i)) $$
    In practice, $\Delta x$ and $\Delta y$ are replaced by a small finite value $\Delta t$ [S1]. The terms $\cos \theta_i$ and $\sin \theta_i$ scale the derivative of the projection data [S1].
5.  **Shading:** Perform shading by estimating gradients directly on the projections and using them as surface normals [S1].
6.  **Compositing:** Determine the final color and opacity of the pixel using the over-operator in front-to-back order [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0.0:
        return (0.0, 0.0, 1.0)
    return (v[0]/length, v[1]/length, v[2]/length)

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def render_pixel(ray_origin, ray_direction, projections, angles, s_start, s_end, step_size):
    """
    Renders a single pixel using Filtered Back-Projection for Volume Rendering.
    
    Args:
        ray_origin: (x, y, z) tuple
        ray_direction: (dx, dy, dz) tuple, normalized
        projections: List of 1D arrays Q_theta_i
        angles: List of float angles theta_i in radians
        s_start, s_end: Float bounds along the ray
        step_size: Float distance between samples
        
    Returns:
        (color, opacity) tuple
    """
    accumulated_color = [0.0, 0.0, 0.0]
    transmittance = 1.0
    num_proj = len(angles)
    scale = math.pi / num_proj
    delta_t = 1.0  # Finite difference step approximating the limit [S1]
    
    s = s_start
    while s <= s_end:
        x = ray_origin[0] + s * ray_direction[0]
        y = ray_origin[1] + s * ray_direction[1]
        z = ray_origin[2] + s * ray_direction[2]
        
        density = 0.0
        grad_x = 0.0
        grad_y = 0.0
        
        for i in range(num_proj):
            theta = angles[i]
            proj = projections[i]
            width = len(proj)
            
            # Project 3D point to 1D detector coordinate per source formula [S1]
            t_i = x * math.cos(theta) + y * math.sin(theta)
            idx = int(t_i)
            
            if 0 <= idx < width:
                density += proj[idx]
                
                # Approximate limit using finite difference [S1]
                t_plus_x = t_i + delta_t * math.cos(theta)
                t_plus_y = t_i + delta_t * math.sin(theta)
                
                idx_x = int(t_plus_x)
                idx_y = int(t_plus_y)
                
                if 0 <= idx_x < width:
                    grad_x += (proj[idx_x] - proj[idx]) / delta_t * math.cos(theta)
                if 0 <= idx_y < width:
                    grad_y += (proj[idx_y] - proj[idx]) / delta_t * math.sin(theta)
        
        # Scale by pi / K [S1]
        density *= scale
        grad_x *= scale
        grad_y *= scale
        
        # Shading: compute normal from gradients [S1]
        # [G] Assuming gradient vector in the projection plane corresponds to surface normal components.
        normal = normalize((grad_x, grad_y, 0.0))
        
        # [G] Simple diffuse lighting model
        light_dir = normalize((0.0, 0.0, 1.0))
        diffuse = max(dot(normal, light_dir), 0.0)
        shading = 0.2 + 0.8 * diffuse
        
        # [G] Standard transfer function: density -> color, opacity
        opacity_sample = 1.0 - math.exp(-density * 0.1)
        color_sample = [shading * density, shading * density, shading * density]
        
        # Front-to-back compositing [S1]
        alpha = opacity_sample * transmittance
        accumulated_color[0] += color_sample[0] * alpha
        accumulated_color[1] += color_sample[1] * alpha
        accumulated_color[2] += color_sample[2] * alpha
        transmittance *= (1.0 - opacity_sample)
        
        s += step_size
        
    return accumulated_color, 1.0 - transmittance
```

## Subtopics

### Gradient Estimation via Filtered Back-Projection
A method to estimate derivatives of the volumetric density function directly from filtered projections, avoiding the computational cost and interpolation errors of grid-based finite difference methods [S1]. It applies Newton's difference quotient to the filtered projection data $Q_{\theta_i}$ instead of the reconstructed density function [S1]. The partial derivative $\tilde{f}_x$ is calculated as the sum of scaled derivatives of the projection data: $\lim_{\Delta x \rightarrow 0} \frac{1}{\Delta x} (Q_{\theta_i}(t_i + \Delta x \cos \theta_i) - Q_{\theta_i}(t_i))$ [S1]. This approach is superior to grid-based central differences, which require 6 additional interpolated samples for a gradient, accumulating interpolation errors, whereas FBP-based estimation uses density values as accurate as grid points, introducing no interpolation error [S1].

## Operating characteristics
*   **Time Complexity:** $O(K \cdot N \cdot S)$, where $K$ is the number of projections, $N$ is the number of pixels, and $S$ is the number of samples per ray [G]. Each sample requires iterating over all $K$ projections [G].
*   **Memory Complexity:** $O(K \cdot W \cdot H)$ to store the projection images [G]. No 3D volume grid is stored in memory [G].
*   **Measured Performance:** Tested on Marschner & Lobb function (64 projections, 64x64 resolution) and Carp dataset (128 projections, 128x256 resolution) [S1].
*   **Strengths:** Higher visual quality than grid-based DVR using the same amount of pixels on all projections as voxels on the regular grid, particularly reducing holes in thin structures [S1]. Avoids interpolation errors inherent in grid-based gradient estimation [S1].
*   **Limitations:** Computationally intensive per sample due to the summation over all projections [G]. Requires pre-filtered projections [S1].
*   **When to use:** When high-quality rendering of thin structures is required and memory for a high-resolution 3D grid is prohibitive [G].
*   **When not to use:** When real-time performance is critical and the number of projections is large, as the per-sample cost is high [G].

## Background
The method was introduced in the paper "D2VR: High-Quality Volume Rendering of Projection-based Volumetric Data" [S1]. It was developed to address the limitations of grid-based volume rendering, specifically interpolation errors and memory usage [S1]. The technique leverages the mathematical properties of Filtered Back-Projection, traditionally used for tomographic reconstruction, to perform direct volume rendering [S1].

## Sources
[S1] D2VR: High-Quality Volume Rendering of Projection-based Volumetric Data, https://doi.org/10.2312/vissym/eurovis06/211-218, fetched 2026-10-26
