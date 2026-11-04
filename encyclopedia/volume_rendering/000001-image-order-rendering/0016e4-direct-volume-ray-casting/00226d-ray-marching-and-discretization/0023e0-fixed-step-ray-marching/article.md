# Fixed Step Ray Marching


## Overview
Fixed Step Ray Marching is a direct volume rendering technique that visualizes 3D scalar fields (such as fog or smoke) by casting rays from the camera and accumulating color and opacity along a fixed-distance path [S1, unofficial]. It solves the problem of rendering heterogeneous, translucent media without requiring explicit mesh extraction or adaptive traversal logic [S1, unofficial]. Unlike sibling methods that may use adaptive stepping or discrete integral approximations, this method employs a uniform step size with front-to-back compositing, often integrating gradient-based shading and transfer functions for material appearance [S4, S7, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D density grid (e.g., VDB or sparse voxel octree), a camera ray for each pixel, a directional light vector, and rendering parameters (`step_size`, `absorption`, `max_steps`, `vol_color`).
*   **Output:** A 2D image where each pixel contains the accumulated RGB color and opacity.

**Data Structures:**
*   **Grid:** A sparse 3D structure storing density values. Access is performed via `sample(x, y, z)` which returns the density at world coordinates.
*   **Scene:** Contains bounding box bounds (`bmin`, `bmax`), light direction (`light_dir`), and volume color (`vol_color`).

**Parameters:**
*   `step_size`: Distance between samples. Default: 0.15 [S1, unofficial].
*   `absorption`: Absorption coefficient multiplier. Default: 12.0 [S1, unofficial].
*   `max_steps`: Maximum number of steps per ray. Default: 800 [S1, unofficial].
*   `vol_color`: Base volume color (RGB). Default: (1.0, 0.95, 0.88) [S1, unofficial].

**Algorithm Steps:**

1.  **Setup:**
    *   Compute the active bounding box (`bbox`) from the grid. If the grid is empty, return the background color [S1, unofficial].
    *   Compute world bounds `bmin` and `bmax` by padding the `bbox` with `step_size * 2.0` to ensure rays entering/exiting the volume are handled correctly [S1, unofficial].
    *   Normalize the light direction vector `light_dir` [S1, unofficial].

2.  **Ray Generation:**
    *   For each pixel $(x, y)$, compute the ray origin and direction.
    *   Compute intersection parameters $t_{enter}$ and $t_{exit}$ with the bounding box using a ray-box intersection test [S1, unofficial].
    *   If the ray does not intersect the box (invalid intersection), set the pixel to the background color and continue to the next pixel [S1, unofficial].

3.  **Marching Loop:**
    *   Initialize the ray parameter $t = \max(t_{enter}, 0.001)$ [S1, unofficial].
    *   Initialize transmittance $T = 1.0$ and accumulated color components $C_r = C_g = C_b = 0.0$ [S1, unofficial].
    *   Loop up to `max_steps`:
        *   If $t > t_{exit}$ or $T < 0.005$, break the loop [S1, unofficial].
        *   **Sampling:**
            *   Compute the current 3D position $P = (px, py, pz)$ using the ray equation at $t$ [S1, unofficial].
            *   Sample the density $d = \text{sample}(px, py, pz)$ [S1, unofficial].
            *   If $d > 0.001$:
                *   Compute the optical depth for the step: $ext = d \times \text{absorption} \times \text{step\_size}$ [S1, unofficial].
                *   **Gradient Calculation:**
                    *   Compute central differences for the gradient vector $\nabla d = (g_x, g_y, g_z)$ using a step size $h = \text{step\_size} \times 0.5$ [S1, unofficial]:
                        $$g_x = \text{sample}(px+h, py, pz) - \text{sample}(px-h, py, pz)$$
                        $$g_y = \text{sample}(px, py+h, pz) - \text{sample}(px, py-h, pz)$$
                        $$g_z = \text{sample}(px, py, pz+h) - \text{sample}(px, py, pz-h)$$
                    *   [G] Handle boundary conditions: if $px \pm h$ or similar coordinates fall outside the grid bounds, clamp the sample coordinates to the nearest valid voxel center or return 0 density.
                *   **Shading:**
                    *   Normalize the gradient: $\hat{n} = \frac{\nabla d}{\|\nabla d\|}$ [S1, unofficial].
                    *   Compute the dot product with the light direction: $ndotl = \max(0.0, -(\hat{n}_x \cdot \text{light}_x + \hat{n}_y \cdot \text{light}_y + \hat{n}_z \cdot \text{light}_z))$ [S1, unofficial]. Note the negative sign indicates the gradient points outward, so we negate for standard lighting models or adjust light direction convention.
                    *   Compute the shading factor: $shade = 0.3 + 0.7 \times ndotl$ [S1, unofficial].
                *   **Composition:**
                    *   Compute step transmittance: $tr\_step = \exp(-ext)$ [S1, unofficial].
                    *   Update accumulated color for each channel $c \in \{r, g, b\}$:
                        $$C_c += T \times (1.0 - tr\_step) \times \text{vol\_color}_c \times shade$$
                    *   Update transmittance: $T = T \times tr\_step$ [S1, unofficial].
        *   Increment $t$ by `step_size` [S1, unofficial].

4.  **Final Composition:**
    *   Add the background contribution to the accumulated color: $C_c += T \times \text{bg\_color}_c$ for each channel [S1, unofficial].
    *   Clamp the final RGB values to the range $[0, 1]$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fixed_step_ray_march(ray_origin, ray_direction, scene, grid, params):
    """
    ray_origin: (x, y, z)
    ray_direction: (x, y, z)
    scene: object with bmin, bmax, light_dir, vol_color, bg_color
    grid: object with sample(x, y, z) method
    params: object with step_size, absorption, max_steps
    """
    # 1. Setup
    t_enter, t_exit = ray_box_intersection(ray_origin, ray_direction, scene.bmin, scene.bmax)
    if t_enter is None:
        return scene.bg_color
    
    light_dir = normalize(scene.light_dir)
    step_size = params.step_size
    absorption = params.absorption
    max_steps = params.max_steps
    
    # 2. Initialize
    t = max(t_enter, 0.001)
    transmittance = 1.0
    color_r = 0.0
    color_g = 0.0
    color_b = 0.0
    
    for _ in range(max_steps):
        if t > t_exit or transmittance < 0.005:
            break
            
        # 3. Sampling
        px, py, pz = ray_origin + ray_direction * t
        density = grid.sample(px, py, pz)
        
        if density > 0.001:
            # Optical depth
            ext = density * absorption * step_size
            
            # Gradient Calculation
            h = step_size * 0.5
            # [G] Clamp coordinates to grid bounds before sampling
            gx = grid.sample(px + h, py, pz) - grid.sample(px - h, py, pz)
            gy = grid.sample(px, py + h, pz) - grid.sample(px, py - h, pz)
            gz = grid.sample(px, py, pz + h) - grid.sample(px, py, pz - h)
            
            # Shading
            grad_len = sqrt(gx*gx + gy*gy + gz*gz)
            if grad_len > 1e-6:
                nx, ny, nz = gx/grad_len, gy/grad_len, gz/grad_len
            else:
                nx, ny, nz = 0.0, 0.0, 0.0
                
            ndotl = max(0.0, -(nx * light_dir.x + ny * light_dir.y + nz * light_dir.z))
            shade = 0.3 + 0.7 * ndotl
            
            # Composition
            tr_step = exp(-ext)
            
            color_r += transmittance * (1.0 - tr_step) * scene.vol_color.r * shade
            color_g += transmittance * (1.0 - tr_step) * scene.vol_color.g * shade
            color_b += transmittance * (1.0 - tr_step) * scene.vol_color.b * shade
            
            transmittance *= tr_step
            
        t += step_size
        
    # 4. Final Composition
    color_r += transmittance * scene.bg_color.r
    color_g += transmittance * scene.bg_color.g
    color_b += transmittance * scene.bg_color.b
    
    return clamp(color_r, 0.0, 1.0), clamp(color_g, 0.0, 1.0), clamp(color_b, 0.0, 1.0)
```

## Subtopics

### [Ray Marching Volume Rendering](1735-ray-marching-volume-rendering.md)
A real-time GPU technique that visualizes 3D scalar fields by casting rays and accumulating color/opacity. It distinguishes itself by often integrating gradient-based shading and transfer functions, and is implemented in frameworks like Taichi for high-performance visualization.

### [Ray Evaluation Stepping DVR](3461-ray-evaluation-stepping-dvr.md)
A fixed-step DVR technique that supports configurable density ranges and early-out termination. It distinguishes itself by explicitly supporting both world-space and object-space indexing and using a modular shader define system.

### [Density Marching](3462-density-marching.md)
A direct volume ray casting technique that samples a 3D texture along a ray, accumulating color and opacity. It is distinguished by its reliance on a fixed number of steps (`STEPS`) and specific implementation of procedural noise (Worley and Perlin) combined with boundary fading.

### [Discrete Volume Rendering Integral (VRI)](3552-discrete-volume-rendering-integral-vri.md)
The numerical approximation of the continuous Volume Rendering Integral used in fixed-step ray marching. It solves the problem of integrating light emission and absorption by replacing path integrals with a Riemann sum over discrete sample points, assuming a piecewise-constant density model.

### [Volumetric Light Shafts (URP Raymarching)](3873-volumetric-light-shafts-urp-raymarching.md)
A post-processing technique for rendering volumetric light shafts in Unity URP. It builds on Fixed Step Ray Marching by using a fixed step size and exponential absorption logic, specifically targeting light shafts from a single directional light.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of steps per ray (`max_steps`). The complexity is linear with respect to resolution and step count.
*   **Memory Complexity:** $O(1)$ per ray for the marching loop state, plus the memory required to store the volume grid.
*   **Measured Performance:** In benchmarks using the Lyr.jl library on CUDA hardware, Fixed Step Ray Marching with HDDA (Hierarchical Data-Driven Acceleration) and caching showed significant speedups over linear GPU rendering and CPU baselines. For example, on a "fire.vdb" complex sparse grid, HDDA+cache rendering was significantly faster than linear GPU rendering, with speedup factors varying by grid complexity [S1, unofficial].
*   **Strengths:** Simple implementation, predictable memory usage, easy to parallelize on GPUs, suitable for real-time applications when combined with acceleration structures like HDDA.
*   **Limitations:** Fixed step size can lead to aliasing or slow convergence if the step size is too large, or excessive computation if too small. Does not handle complex scattering or participating media with high optical depth efficiently without many steps.
*   **When to use:** Real-time fog, smoke, or simple volumetric effects where performance is critical and high-fidelity scattering is not required.
*   **When not to use:** Scenes requiring high-fidelity light transport, complex scattering, or highly heterogeneous volumes where adaptive stepping would be more efficient.

## Background
Fixed Step Ray Marching is a foundational technique in direct volume rendering, originating from the need to efficiently visualize 3D scalar fields such as medical imaging data or atmospheric effects. It is implemented in various graphics libraries and frameworks, including Lyr.jl, which provides a volume ray marcher for fog volumes using fixed step size, exponential absorption, and gradient-based shading [S1, unofficial]. The method is widely used in real-time graphics engines and scientific visualization tools due to its simplicity and parallelizability.

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
