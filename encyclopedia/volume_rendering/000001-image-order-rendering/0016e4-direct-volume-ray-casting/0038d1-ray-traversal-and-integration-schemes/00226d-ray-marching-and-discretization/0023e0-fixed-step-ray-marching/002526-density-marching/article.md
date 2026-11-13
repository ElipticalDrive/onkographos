# Density Marching

Also called: Front-to-Back Ray Marching, Simple Ray Marching

## Overview
Density Marching is a direct volume rendering technique that samples a 3D density field along rays cast from the camera, accumulating color and opacity using front-to-back blending to produce the final pixel color [S1, S2, unofficial]. It differs from adaptive or hierarchical traversal methods by using a fixed number of sampling steps per ray, making it straightforward to implement [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a 3D volume texture (density field), camera position, ray direction, bounding box intersection distances (`t_min`, `t_max`), and a step count `N` [S1, S2, unofficial]. It outputs an accumulated color and transmittance per pixel [S1, S2, unofficial].

**Data Structures:**
*   **Image Buffer:** A 2D array of shape `[height, width, 3]` with `uint8` data type, storing the final RGB color [S1, unofficial].
*   **Ray Data:** Per-pixel ray origin (camera position) and ray direction (normalized vector) [S1, S2, unofficial].
*   **Accumulators:** Per-pixel variables for `render_color` (RGB float), `T` (transmittance scalar), and optionally `accum` (RGBA) [S1, S2, unofficial].

**Parameters:**
*   `N` (or `num_points`/`STEPS`): The number of sampling steps along each ray. Typical values are 128 [S1, S2, unofficial].
*   `DENSITY_MULTIPLIER`: Scales sampled density to produce alpha in linear approximations. Value: 5.0 [S2, unofficial].
*   `background_color`: Color for non-intersecting rays. Value: `[157., 204., 224.]` [S1, unofficial].
*   `cell_color`: Color of volume cells. Value: `[255, 255, 255]` (white) [S1, unofficial].
*   `t_min`, `t_max`: Entry and exit distances of the ray within the bounding box [S1, S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:** For each pixel, initialize `render_color` to `[0., 0., 0.]` and transmittance `T` to `1.` [S1, unofficial].
2.  **Ray Setup:** Calculate the ray direction by normalizing the vector from the camera position to the pixel's world position [S1, S2, unofficial].
3.  **Bounding Box Intersection:** Intersect the ray with the volume's Axis-Aligned Bounding Box (AABB) using the slab method to determine entry (`tNear`) and exit (`tFar`) distances [S1, S2, unofficial]. If the ray does not intersect, discard the pixel [S2, unofficial].
4.  **Boundary Clamping:** Clamp the entry distance to the camera plane: `t_min = max(t_min, 0.0)` to handle rays originating inside the volume [G].
5.  **Step Size Calculation:** Compute the step size $\delta = \frac{t_{max} - t_{min}}{N}$, where $N$ is the number of steps (`num_points` or `STEPS`) [S1, S2, unofficial].
6.  **Marching Loop:** Iterate $i$ from $0$ to $N-1$:
    a.  **Sample Position:** Calculate the current position along the ray: $p_i = \text{rayOrigin} + (t_{min} + i \cdot \delta) \cdot \text{rayDirection}$ [S1, S2, unofficial].
    b.  **Density Sampling:** Read the density value $\rho_i$ from the 3D texture at $p_i$ [S1, S2, unofficial].
    c.  **Alpha Calculation:** Compute the alpha for the current step: $\alpha_i = 1 - e^{-\rho_i \cdot \delta}$ [S1, unofficial]. Note: Some implementations use a linear approximation $\alpha_i = \rho_i \cdot \text{DENSITY\_MULTIPLIER} \cdot \delta$ [S2, unofficial].
    d.  **Weight Calculation:** Calculate the contribution weight using the transmittance *before* the current step: $w_i = T \cdot \alpha_i$ [S1, unofficial].
    e.  **Color Accumulation:** Add the weighted color: $\text{render\_color} \leftarrow \text{render\_color} + w_i \cdot \text{cell\_color}$ [S1, unofficial].
    f.  **Transmittance Update:** Update the accumulated transmittance: $T \leftarrow T \cdot e^{-\rho_i \cdot \delta}$ [S1, unofficial].
    g.  **Early Exit:** If $T < 0.001$, break the loop [S2, unofficial].
7.  **Final Compositing:** Composite the accumulated color with the background:
    $$ \text{final\_color} = T \cdot \text{background\_color} + \text{render\_color} $$
    [S1, unofficial]. If the accumulated opacity $1 - T$ is below a threshold (e.g., 0.001), discard the fragment [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def density_marching(camera_pos, ray_dir, t_min, t_max, num_points, volume_texture, background_color, cell_color):
    """
    Performs density marching for a single ray.
    """
    # 1. Initialization
    render_color = [0.0, 0.0, 0.0]
    T = 1.0
    
    # 2. Boundary Clamping & Intersection Check
    t_min = max(t_min, 0.0)
    if t_max <= t_min:
        return [0.0, 0.0, 0.0]
        
    # 3. Step Size
    delta = (t_max - t_min) / num_points
    
    # 4. Marching Loop
    for i in range(num_points):
        # Sample Position
        t = t_min + i * delta
        pos = camera_pos + ray_dir * t
        
        # Density Sampling
        density = volume_texture.sample(pos)
        
        # Alpha Calculation (Exponential Absorption)
        alpha = 1.0 - math.exp(-density * delta)
        
        # Weight Calculation (Standard: use T before update)
        weight = T * alpha
        render_color[0] += weight * cell_color[0]
        render_color[1] += weight * cell_color[1]
        render_color[2] += weight * cell_color[2]
        
        # Transmittance Update
        T *= math.exp(-density * delta)
        
        # Early Exit Optimization
        if T < 0.001:
            break
            
    # 5. Final Compositing
    final_color = [
        T * background_color[0] + render_color[0],
        T * background_color[1] + render_color[1],
        T * background_color[2] + render_color[2]
    ]
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Transmittance Order:** Updating transmittance before calculating the weight for the current step leads to incorrect lighting and opacity accumulation [S1, unofficial].
*   **Boundary Artifacts:** Abrupt cutoffs at the volume boundaries can be mitigated by applying fade factors based on distance to the box edges [S3, unofficial].
*   **Ray Origin Inside Volume:** Failing to clamp `t_min` to 0 causes sampling points to fall behind the camera, resulting in zero density reads and black output [G].
*   **Precision:** Using fixed step sizes may lead to aliasing or missing thin structures if the step size is too large relative to the volume features [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
Density Marching is a fundamental technique in direct volume rendering, originating from the early days of computer graphics [S1, S2, unofficial]. It is widely used in scientific visualization and real-time applications due to its simplicity [S1, S2, unofficial].

## Sources
[S1] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
[S2] GitThirteen/vis-magnus-2018, https://github.com/GitThirteen/vis-magnus-2018, fetched 2026-10-26 (unofficial)
[S3] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
