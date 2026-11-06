# WebGPU Voxel Ray Tracing Renderer

Also called: DDA voxel grid traversal, WebGPU-based ray tracing renderer

## Overview
A WebGPU compute shader-based ray tracer for voxel models that replaces a rasterizer [S1, unofficial]. It casts rays per pixel, traverses a 3D voxel grid using DDA, applies directional lighting with diffuse shading, and generates hard shadows via shadow rays [S1, unofficial]. It sets itself apart from its parent [Voxel Shadow Raycasting with DDA](2410-voxel-shadow-raycasting-with-dda.md) by implementing the entire rendering pipeline (primary rays, shading, and shadow rays) within a WebGPU compute shader, rather than relying on a GLSL rasterizer with DDA for shadow calculation only [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid (dense or sparse), a project palette of block colors, lighting parameters (sun direction, intensity, ambient light), and camera state.
*   **Output:** A 2D image buffer rendered to the canvas [S1, unofficial].

**Data Structures:**
*   **Uniform Buffer:** Expanded to 128 bytes to store lighting parameters [S1, unofficial].
*   **Storage Buffer:** Contains the project palette for retrieving block colors [S1, unofficial].
*   **Voxel Data:** Gathered from the voxel engine into a flat buffer [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** The renderer checks for WebGPU support. If unavailable, it displays a fallback background [S1, unofficial].
2.  **Primary Ray Casting:** For each pixel, a primary ray is cast into the scene [S1, unofficial].
3.  **Voxel Traversal:** The 3D voxel grid is traversed using the Digital Differential Analyzer (DDA) algorithm to find the first intersecting voxel [S1, unofficial].
4.  **Diffuse Shading:** If a voxel is hit, diffuse shading is calculated using the dot product of the surface normal ($N$) and the light direction ($L$) [S1, unofficial].
5.  **Shadow Raycasting:** Shadow rays are cast by performing DDA traversal from the hit point toward the sun direction to determine occlusion [S1, unofficial].
6.  **Color Retrieval:** Block colors are retrieved from the storage buffer containing the project palette [S1, unofficial].
7.  **Ground Plane:** A ground plane with grid lines is rendered, which receives shadows [S1, unofficial].
8.  **Background:** A subtle sky gradient background is rendered [S1, unofficial].
9.  **Blitting:** The compute shader output is blitted to the canvas using a fullscreen triangle render pass [S1, unofficial].

**Formulas:**
*   **Diffuse Shading:** $I_{diffuse} = \max(0, N \cdot L)$ [S1, unofficial].
*   **Sun Direction:** Calculated from Sun Azimuth ($\theta$) and Sun Elevation ($\phi$) using spherical-to-Cartesian conversion: $L = (\cos\phi \sin\theta, \sin\phi, \cos\phi \cos\theta)$ [G].

**Parameters:**
*   **Sun Azimuth:** Horizontal angle of the sun, range 0 to 360 degrees [S1, unofficial].
*   **Sun Elevation:** Vertical angle of the sun, range 5 to 90 degrees [S1, unofficial].
*   **Sun Intensity:** Brightness of the directional light source [S1, unofficial].
*   **Ambient Light Intensity:** Intensity of the ambient light component [S1, unofficial].
*   **Shadow Strength:** Intensity/opacity of the shadows [S1, unofficial].
*   **Sun Color:** Presets include Warm, White, Cool, Gold, Pink [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **WebGPU Availability:** WebGPU is not available in headless CI environments, so the canvas shows a fallback background instead of rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_pixel(pixel_x, pixel_y, camera, voxel_grid, palette, lighting):
    ray_origin, ray_direction = camera.get_ray(pixel_x, pixel_y)
    hit_voxel, t = dda_traverse(ray_origin, ray_direction, voxel_grid)

    if hit_voxel is not None:
        base_color = palette[hit_voxel[0]][hit_voxel[1]][hit_voxel[2]]
        hit_point = [ray_origin[i] + ray_direction[i] * t for i in range(3)]
        hit_normal = compute_normal(hit_point, hit_voxel)

        diffuse_factor = max(0.0, dot(hit_normal, lighting.sun_direction))
        is_shadowed = dda_traverse_shadow(hit_point, lighting.sun_direction, voxel_grid)
        if is_shadowed:
            diffuse_factor = 0.0

        final_color = base_color * (lighting.ambient_intensity + diffuse_factor * lighting.sun_intensity)
    else:
        final_color = get_sky_gradient(ray_direction)
    return final_color

def dda_traverse(ray_origin, ray_direction, voxel_grid):
    voxel = [math.floor(ray_origin[i]) for i in range(3)]
    step = [1 if ray_direction[i] > 0 else -1 if ray_direction[i] < 0 else 0 for i in range(3)]
    t_delta = [1.0 / abs(ray_direction[i]) if ray_direction[i] != 0 else float('inf') for i in range(3)]

    t_max = [0.0, 0.0, 0.0]
    for i in range(3):
        if step[i] > 0:
            t_max[i] = (voxel[i] + 1.0 - ray_origin[i]) * t_delta[i]
        elif step[i] < 0:
            t_max[i] = (voxel[i] - ray_origin[i]) * t_delta[i]
        else:
            t_max[i] = float('inf')

    bounds = voxel_grid.bounds
    t = 0.0
    while 0 <= voxel[0] < bounds[0] and 0 <= voxel[1] < bounds[1] and 0 <= voxel[2] < bounds[2]:
        if voxel_grid.is_opaque(voxel):
            return voxel, t
        axis = t_max.index(min(t_max))
        t += t_delta[axis]
        t_max[axis] += t_delta[axis]
        voxel[axis] += step[axis]
    return None, 0.0

def dda_traverse_shadow(origin, direction, voxel_grid):
    epsilon = 1e-4
    shifted_origin = [origin[i] + direction[i] * epsilon for i in range(3)]
    hit_voxel, _ = dda_traverse(shifted_origin, direction, voxel_grid)
    return hit_voxel is not None

def dot(a, b):
    return sum(a[i] * b[i] for i in range(3))

def compute_normal(point, voxel):
    # Returns the outward face normal of the voxel hit
    dx = point[0] - voxel[0]
    dy = point[1] - voxel[1]
    dz = point[2] - voxel[2]
    if abs(dx - 0.5) < 1e-3: return [1.0, 0.0, 0.0]
    if abs(dy - 0.5) < 1e-3: return [0.0, 1.0, 0.0]
    if abs(dz - 0.5) < 1e-3: return [0.0, 0.0, 1.0]
    return [0.0, 1.0, 0.0]

def get_sky_gradient(direction):
    # Returns a subtle sky gradient based on ray direction
    return [0.5, 0.7, 1.0]
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Performance depends on the hardware's WebGPU support and the complexity of the voxel scene [S1, unofficial].
*   **Strengths:** Real-time lighting and shadows in a voxel environment; interactive controls for lighting parameters [S1, unofficial].
*   **Limitations:** Requires WebGPU support; not available in headless environments [S1, unofficial].
*   **When to use:** When real-time voxel rendering with dynamic lighting and shadows is required in a web browser [S1, unofficial].
*   **When not to use:** In headless CI environments or browsers without WebGPU support [S1, unofficial].

## Background
The WebGPU Voxel Ray Tracing Renderer was developed as part of the LunaVoxel project [S1, unofficial]. It was introduced as an alternative to the existing Three.js rasterizer, accessible via a dedicated Render tab in the header navigation [S1, unofficial]. The implementation includes directional lighting, shadow rays, and interactive controls [S1, unofficial]. It was authored by tfritzy and merged in February 2026 [S1, unofficial].

## Sources
[S1] Add WebGPU ray tracing renderer with lighting and render tab, https://github.com/tfritzy/LunaVoxel/pull/177, fetched 2026-10-25 (unofficial)
