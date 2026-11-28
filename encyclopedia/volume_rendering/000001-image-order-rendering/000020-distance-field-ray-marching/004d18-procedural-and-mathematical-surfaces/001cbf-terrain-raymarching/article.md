# Terrain Raymarching

Also called: Height Field, Heightmap Raymarching, 高度场

## Overview
Terrain Raymarching renders a 3D landscape defined by a 2D heightmap by casting rays through each pixel of the image and stepping along them until the ray's vertical position drops below the terrain height at its current horizontal coordinates [S1, unofficial]. It differs from standard Signed Distance Field raymarching by evaluating a height function $h(x,z)$ directly rather than a global distance field, which simplifies the intersection test but requires careful step size management to avoid perspective occlusion artifacts [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel [G]. Heightmap function $h(x, z)$ returning surface height at world coordinates $(x, z)$, provided as a texture or procedural function [S1, unofficial]. Maximum marching steps $N_{max}$ and minimum step size $\epsilon$ to prevent infinite loops or precision errors [G].
*   **Outputs:** Color $C$ for each pixel, determined by the height and shading of the intersection point [S1, unofficial].

**Data Structures:**
*   No complex acceleration structures are required if the heightmap is evaluated directly [G]. If the heightmap is large, a 2D texture array or procedural noise function is used [G].

**Steps:**
1.  Initialize current position $\mathbf{p} = \mathbf{o}$ and step count $i = 0$ [G].
2.  While $i < N_{max}$:
    a.  Extract horizontal coordinates: $x = \mathbf{p}.x$, $z = \mathbf{p}.z$ [G].
    b.  Query terrain height: $h_{terrain} = h(x, z)$ [S1, unofficial].
    c.  Calculate vertical distance to surface: $dist = \mathbf{p}.y - h_{terrain}$ [G].
    d.  If $dist < \epsilon$ (intersection found): set $\mathbf{p}_{hit} = \mathbf{p}$ and break loop [G].
    e.  If $dist < 0$ (ray passed below surface): clamp $\mathbf{p}.y = h_{terrain}$ and break [G].
    f.  Advance ray by fixed distance $\Delta$ along direction: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{d} \cdot \Delta$ [S1, unofficial].
    g.  Increment $i$ [G].
3.  If intersection found, compute color:
    a.  Determine base color based on height $h_{terrain}$ (e.g., white for snow at high $h$, green for grass at low $h$) [S1, unofficial].
    b.  Compute surface normal $\mathbf{n}$ at $\mathbf{p}_{hit}$ using finite differences of $h(x,z)$:
        $$ \mathbf{n} = \text{normalize}(\langle -\frac{\partial h}{\partial x}, 1, -\frac{\partial h}{\partial z} \rangle) $$
    c.  Shade the point by raymarching towards a light source [S1, unofficial].

**Formulas:**
*   Intersection condition: $\mathbf{p}.y - h(\mathbf{p}.x, \mathbf{p}.z) < \epsilon$ [G].
*   Surface normal: $\mathbf{n} = \text{normalize}(\langle -\partial_x h, 1, -\partial_z h \rangle)$ [G].

**Parameters:**
*   `step_size` ($\Delta$): Depends on terrain resolution and camera distance; too large causes tunneling and too small causes performance issues [G].
*   `max_steps` ($N_{max}$): Typically 64–256 for real-time applications [G].

**Edge Cases and Pitfalls:**
*   **Tunneling:** If the terrain has steep slopes, a fixed step size may jump over the surface, which is a known pitfall of perspective relationships in heightfield rendering [S2, unofficial].
*   **Precision:** Floating-point precision errors can cause jittering near the surface [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raymarch_terrain(ray_origin, ray_direction, height_func, max_steps, step_size, epsilon=1e-3):
    p = ray_origin
    for i in range(max_steps):
        h_terrain = height_func(p.x, p.z)
        dist = p.y - h_terrain
        if dist < epsilon:
            return p, h_terrain
        if dist < 0:
            p.y = h_terrain
            return p, h_terrain
        p = p + ray_direction * step_size
    return None, None
```

## Subtopics

### Horizon Mapping
Horizon Mapping is a technique to describe the self-occlusion of a heightfield by calculating visibility [S3, unofficial]. It determines the altitude or horizon angle for a set of azimuth directions for every point in the surface, storing this data in a horizon map [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** The method fits perfectly on the GPU, as each pixel can be calculated independently [S1, unofficial].
*   **Limitations:** It may occlude parts of the region due to perspective relationships [S2, unofficial].
*   **When to use:** Not given in the sources. [G]
*   **When not to use:** Not given in the sources. [G]

## Background
Terrain Raymarching emerged as a practical application of raymarching techniques for rendering landscapes [S1, unofficial]. The technique leverages the simplicity of heightmaps and applies modern GPU-based raymarching to them [S1, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
