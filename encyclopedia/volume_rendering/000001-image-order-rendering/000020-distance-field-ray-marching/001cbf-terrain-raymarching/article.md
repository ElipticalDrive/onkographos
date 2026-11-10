# Terrain Raymarching

Also called: Height Field, Heightmap Raymarching, 高度场

## Overview
Terrain Raymarching is a visualization method that renders a 2D heightmap as a pseudo-3D terrain surface by casting rays through the image and stepping along them until the ray position drops below the terrain height defined by the heightmap [S1, S2, unofficial]. It solves the problem of rendering complex, high-detail terrain geometry efficiently on the GPU by treating the heightmap as an implicit surface constraint rather than a mesh [S1, unofficial]. Unlike standard mesh-based rendering which requires tessellation, or full 3D volume rendering which samples a 3D grid, this method operates directly on the 2D scalar field of the heightmap, making it distinct from its sibling [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) and [SDF Primitive Definitions](1578-sdf-primitive-definitions.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A heightmap $H(x, z)$, which can be a sampled texture or a procedural function returning a scalar height value for any $(x, z)$ coordinate. A camera definition (origin $\mathbf{o}$, direction $\mathbf{d}$) and a maximum ray distance $t_{max}$.
*   **Output:** A color value for each pixel, determined by the height and shading of the first point on the ray that intersects the terrain surface.

**Data Structures:**
*   **Heightmap:** A 2D array or texture $H$ of size $W \times H$, storing scalar float values representing elevation.
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).

**Algorithm Steps:**
1.  **Ray Casting:** For each pixel, construct a ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S1, unofficial].
2.  **Initialization:** Initialize the accumulated distance $t = 0$ (or near-plane) [S1, unofficial].
3.  **Marching Loop:**
    *   Calculate the current 3D position: $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S1, unofficial].
    *   Sample the heightmap at the horizontal coordinates of $\mathbf{p}$: $h_{terrain} = H(p_x, p_z)$ [S1, unofficial].
    *   Compare the ray's vertical position $p_y$ with the terrain height $h_{terrain}$.
    *   **Termination:** If $p_y \le h_{terrain}$ (within a small threshold $\epsilon$), the ray has hit the terrain surface. Stop marching [S1, unofficial]. If $t > t_{max}$, the ray has escaped. Stop marching [S2, unofficial].
    *   **Step:** Advance the ray. In standard sphere tracing, the step is $d = f(\mathbf{p})$. For heightfields, the step size is often fixed or adaptive based on distance to maintain resolution, but conceptually it advances until the height constraint is met [S1, unofficial]. Note: Standard SDF sphere tracing assumes a Lipschitz constant of 1. Heightfield raymarching does not strictly use an SDF for the step size in the same way, but rather checks the vertical intersection. A common implementation uses a fixed step size or a step proportional to the distance to the horizon to optimize for LOD [S2, unofficial].
4.  **Shading:** Upon intersection at $\mathbf{p}_{hit}$, determine the color based on height (e.g., white for snow at high altitudes, green for grass at low altitudes) [S1, unofficial]. Shade the point by raymarching towards a light source to compute lighting [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_raymarch(heightmap, camera_origin, camera_direction, max_steps, max_dist):
    t = 0.0
    for step in range(max_steps):
        if t > max_dist:
            return background_color
        
        # Current 3D position
        p = camera_origin + camera_direction * t
        
        # Sample heightmap at current x, z
        # Assuming heightmap is a function or texture sampler
        terrain_height = heightmap.sample(p.x, p.z)
        
        # Check intersection: ray y is below or at terrain height
        # Using a small epsilon for stability
        if p.y <= terrain_height + 0.001:
            # Hit detected
            color = get_color_based_on_height(terrain_height)
            color = shade_point(p, terrain_height, light_source)
            return color
        
        # Step forward
        # Note: Step size can be fixed or adaptive. 
        # A simple approach is a fixed step, but adaptive is better for performance.
        # Here we use a simple fixed step for clarity, though [S1, unofficial] implies "step along the ray by some size".
        step_size = 0.1 # Placeholder for adaptive step logic [G]
        t += step_size
        
    return background_color
```

**Edge Cases and Pitfalls:**
*   **Perspective Occlusion:** The method may occlude parts of the region due to perspective relationships, particularly if the step size is too large relative to the terrain detail [S2, unofficial].
*   **Precision:** Floating-point precision issues can cause the ray to miss the surface or jitter near the intersection. A convergence threshold $\epsilon$ is required [S1, unofficial].

## Subtopics

### Horizon Mapping
Horizon Mapping is a technique to describe the self-occlusion of a heightfield by calculating visibility [S3, unofficial]. It is used to enhance lighting and shadowing accuracy in terrain rendering without the cost of full ray tracing for shadows. Unlike standard raymarching which only finds the primary surface intersection, this method precomputes visibility data. It is chosen when self-shadowing is required for visual fidelity but full global illumination is too expensive. It works by determining the altitude or horizon angle for a set of azimuth directions for every point in the surface, storing this data into a horizon map [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the average number of steps per ray. The step count depends on the terrain complexity and step size strategy.
*   **Memory Complexity:** $O(W \cdot H)$ for the heightmap storage.
*   **Performance:** Highly efficient on GPUs as each pixel is calculated independently [S1, unofficial]. It allows for real-time rendering of high-detail terrain without mesh tessellation overhead.
*   **Strengths:** Simple implementation, easy to integrate with procedural generation, high level of detail control [S1, unofficial].
*   **Limitations:** Limited to heightfield representations (no overhangs or caves), potential occlusion artifacts if step size is not adaptive [S2, unofficial].

## Background
Terrain Raymarching emerged as a practical application of raymarching techniques for rendering complex scenes, specifically leveraging the simplicity of heightmaps. It was popularized in the context of real-time graphics and demoscene programming, where efficient rendering of large terrains was crucial. The technique is discussed in early raymarching tutorials and blogs, such as those by Inigo Quilez and others in the 9bitscience community [S1, unofficial]. It is often contrasted with traditional mesh-based terrain rendering and other volume rendering techniques.

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
