# Checkerboard Progressive Rendering
Also called: Checkerboard sampling
## Overview
Checkerboard Progressive Rendering reduces per-frame computational cost by tracing only half the pixels in a checkerboard pattern each frame, then blending the new results with previous frames to maintain a complete image [S1, unofficial]. It addresses the problem of interactive volume rendering where full-resolution ray tracing per frame is too expensive, trading per-frame noise for temporal stability [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   Input: Previous frame's blended color buffer $C_{prev}$, ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, volume grid $V$ [G].
*   Output: Current frame's blended color buffer $C_{final}$ [G].

**Data Structures:**
*   $C_{prev}$ and $C_{final}$: 2D arrays of color vectors (e.g., `float3`) with dimensions $W \times H$ [G].
*   $V$: 3D array of scalar density values with dimensions $D_x \times D_y \times D_z$ [G].

**Steps:**
1. Initialize an empty buffer $C_{curr}$ of size $W \times H$ [G].
2. For each pixel $(x, y)$, determine if it is active in the current frame $f$ using a checkerboard mask: $M(x, y, f) = ((x + y + f) \bmod 2 == 0)$ [S1, unofficial].
3. If $M(x, y, f)$ is true, trace a ray through the volume to compute $C_{new}(x, y)$ [S1, unofficial]. If false, copy the previous buffer: $C_{curr}(x, y) = C_{prev}(x, y)$ [S1, unofficial].
4. Blend $C_{curr}$ and $C_{prev}$ to produce $C_{final}$ [G].
5. Return $C_{final}$ as the output and the new $C_{prev}$ for the next frame [G].

**Formulas:**
*   Checkerboard activation: $M(x, y, f) = \begin{cases} 1 & \text{if } (x + y + f) \bmod 2 = 0 \\ 0 & \text{otherwise} \end{cases}$ [S1, unofficial].
*   Volume rendering integral (discrete slab march): $C = \sum_{i=0}^{N-1} T_i \cdot (1 - e^{-\sigma_i \Delta t}) \cdot c_i$, where $T_i = e^{-\sum_{j=0}^{i-1} \sigma_j \Delta t}$ is the accumulated transmittance, $\sigma_i$ is the density at step $i$, $\Delta t$ is the step size, and $c_i$ is the emitted color [G].
*   Temporal blend: $C_{final}(x, y) = \alpha \cdot C_{curr}(x, y) + (1 - \alpha) \cdot C_{prev}(x, y)$, where $\alpha$ is the blend weight [G].

**Parameters:**
*   Frame index $f$: Integer, increments by 1 each frame [S1, unofficial].
*   Blend weight $\alpha$: Not given in the sources. [G] Standard implementations use $\alpha = 0.5$ for simple averaging.
*   Step size $\Delta t$: Not given in the sources. [G] Typically set to the grid cell size or a fraction thereof.

**Edge Cases and Pitfalls:**
*   **Zero Direction Components:** Division by zero occurs when computing inverse direction for slab stepping. Handled by clamping or using a large sentinel value [G].
*   **Boundary Crossing:** Rays exiting the volume grid must be clamped to $[0, D_{max}]$ to prevent out-of-bounds memory access [G].
*   **Motion Artifacts:** Rapid camera or object movement breaks temporal coherence, causing ghosting. The method assumes minimal motion between frames [S1, unofficial].
*   **Convergence:** Full resolution is achieved after two frames. The first frame after a scene change or reset will show checkerboard artifacts until blending stabilizes [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def checkerboard_progressive_render(frame_index, width, height, prev_buffer, grid, grid_dims, ray_origin, ray_dir, step_size):
    curr_buffer = [[None] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            is_active = ((x + y + frame_index) % 2 == 0)
            if is_active:
                color = trace_volume(ray_origin, ray_dir, grid, grid_dims, step_size)
                curr_buffer[y][x] = color
            else:
                curr_buffer[y][x] = prev_buffer[y][x]
    
    alpha = 0.5  # [G] Standard simple averaging
    final_buffer = [[None] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            final_buffer[y][x] = alpha * curr_buffer[y][x] + (1 - alpha) * prev_buffer[y][x]
    return final_buffer

def trace_volume(origin, direction, grid, grid_dims, step_size):
    inv_dir = [1.0 / d if d != 0.0 else float('inf') for d in direction]
    t_min = 0.0
    t_max = max(grid_dims) * step_size  # [G] Approximate volume extent
    t = t_min
    color = [0.0, 0.0, 0.0]
    transmittance = 1.0
    
    while t < t_max:
        gx = int((origin[0] + t * direction[0]) / step_size)
        gy = int((origin[1] + t * direction[1]) / step_size)
        gz = int((origin[2] + t * direction[2]) / step_size)
        
        if 0 <= gx < grid_dims[0] and 0 <= gy < grid_dims[1] and 0 <= gz < grid_dims[2]:
            density = grid[gz][gy][gx]
        else:
            density = 0.0
            
        absorption = density * step_size
        color[0] += transmittance * (1.0 - math.exp(-absorption))
        color[1] += transmittance * (1.0 - math.exp(-absorption))
        color[2] += transmittance * (1.0 - math.exp(-absorption))
        transmittance *= math.exp(-absorption)
        
        if transmittance < 1e-4:
            break
        t += step_size
    return color
```
## Subtopics

### [Progressive Ray Tracing](9723-progressive-ray-tracing.md)
Computes only a subset of rays during interaction and progressively fills remaining pixels when interaction stops. Chose when interactive latency is critical and full refinement can occur offline; differs from checkerboard rendering by using spatial subsampling with interpolation rather than temporal blending of half-pixel traces.

## Operating characteristics
**Time Complexity:** $O(W \cdot H)$ per frame, with a constant factor of 0.5 compared to full rendering, as only half the pixels are traced each frame [S1, unofficial].
**Memory Complexity:** $O(W \cdot H)$ to store the previous frame's buffer and the current frame's buffer [G].
**Performance:** Reduces per-frame compute by tracing only half the pixels each frame while blending results over time [S1, unofficial].
**Strengths:** Provides a complete image every frame, reducing latency for interactive applications [G].
**Limitations:** Ghosting artifacts with motion; requires two frames to converge to the full-resolution result [S1, unofficial].
**When to use:** Interactive volume rendering where frame rate is critical and motion is minimal [G].
**When not to use:** High-motion scenes or when high fidelity is required every frame [G].
## Background
Checkerboard Progressive Rendering is a technique used in real-time volume rendering to maintain interactive frame rates by deviating from fixed uniform sampling [S1, unofficial]. It was implemented in a voxel raytracer to reduce per-frame compute by tracing only half the pixels each frame while blending results over time [S1, unofficial].
## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
