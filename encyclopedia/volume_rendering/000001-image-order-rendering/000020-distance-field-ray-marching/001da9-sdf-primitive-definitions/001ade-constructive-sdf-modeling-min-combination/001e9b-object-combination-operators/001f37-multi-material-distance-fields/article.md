# Multi-Material Distance Fields

Also called: Color-Coded SDFs, SDF with Material Data

## Overview
Multi-Material Distance Fields associate material properties with geometric primitives by returning a vector containing both the distance value and the material identifier from the scene's `map()` function [S1, unofficial]. This extends standard ray marching to enable distinct visual identification of objects within a combined scene without altering the underlying distance field logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $ro$, ray direction $rd$, maximum march steps, and a Color Ramp texture [S1, unofficial].
*   **Output:** A final pixel color determined by the material ID of the closest hit surface [S1, unofficial].

**Data Structures:**
*   **Multi-Material Point:** A 2D vector where the `x` component is the signed distance value $d$ and the `y` component is the material identifier $m$ (e.g., a normalized color index) [S1, unofficial].

**Algorithm Steps:**
1.  **Define Primitive Mappers:** Each geometric primitive returns a 2D vector where `x` is the standard SDF distance and `y` is a constant material ID [S1, unofficial].
2.  **Combine Fields:** Use the Union operator to merge primitives [S1, unofficial]. The operator compares only the `x` components to determine the closest surface, but returns the entire 2D vector of the winning primitive [S1, unofficial].
3.  **Ray Marching:** Initialize ray distance $t = 0$ [S1, unofficial]. For each step, calculate world position $p = ro + rd \cdot t$, sample the combined scene $d = \text{map}(p)$, check for intersection if $d.x < \epsilon$ (e.g., $0.001$), and advance $t \leftarrow t + d.x$ otherwise [S1, unofficial].
4.  **Shading:** Upon hit, extract the material ID from $d.y$, sample a Color Ramp texture using $d.y$ as the coordinate to retrieve the final color, and compute lighting based on the surface normal and retrieved color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec2:
    def __init__(self, x, y): self.x, self.y = x, y
class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z

def sd_box(p, h):
    q = Vec3(abs(p.x)-h.x, abs(p.y)-h.y, abs(p.z)-h.z)
    return math.sqrt(max(q.x,0)**2 + max(q.y,0)**2 + max(q.z,0)**2) + min(max(q.x, max(q.y, q.z)), 0)

def sd_sphere(p, r):
    return math.sqrt(p.x**2 + p.y**2 + p.z**2) - r

def sd_torus(p, r):
    q = math.sqrt(p.x**2 + p.z**2) - r.x
    return math.sqrt(q**2 + p.y**2) - r.y

def opU(d1, d2):
    return d1 if d1.x < d2.x else d2

def map_torus(p): return Vec2(sd_torus(p, Vec2(1.0, 0.2)), 0.5)
def map_box(p): return Vec2(sd_box(Vec3(p.x+3, p.y, p.z), Vec3(0.75, 0.5, 0.5)), 0.25)
def map_sphere(p): return Vec2(sd_sphere(Vec3(p.x-3, p.y, p.z), 1.0), 0.75)

def map_scene(p):
    ret = opU(map_torus(p), map_box(p))
    return opU(ret, map_sphere(p))

def sample_color_ramp(mat_id):
    # Simplified texture lookup mapping material ID to RGB
    return (mat_id, 1.0 - mat_id, 0.5)

def compute_lighting(p, rd, color):
    # Simplified diffuse lighting calculation
    return (color[0]*0.8, color[1]*0.8, color[2]*0.8)

def raymarch(ro, rd):
    t = 0.0
    max_steps = 64
    hit_mat = -1.0
    for _ in range(max_steps):
        p = Vec3(ro.x + rd.x*t, ro.y + rd.y*t, ro.z + rd.z*t)
        d = map_scene(p)
        if d.x < 0.001:
            hit_mat = d.y
            break
        t += d.x
    if hit_mat >= 0:
        color = sample_color_ramp(hit_mat)
        return compute_lighting(p, rd, color)
    return (0.0, 0.0, 0.0)
```

**Parameters:**
*   **ColorRamp:** A uniform `sampler2D` texture mapping material IDs to colors [S1, unofficial].
*   **Threshold:** A small epsilon value (e.g., $0.001$) for determining intersection [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Material ID Precision:** Material IDs are discrete, causing sharp visual changes at boundaries between materials [G]. Using integer IDs mapped to discrete texture coordinates is often more robust than continuous floats [G].
*   **Normal Calculation:** Normals must be computed carefully to avoid blending across material boundaries if physically based rendering is desired [G].

## Subtopics

### [Material Object Format](1981-material-object-format.md)
A data structure that holds surface shading information, including color, physical properties, and texture descriptors, accessible from both host and device code. It distinguishes itself by explicitly separating geometric displacement (via `heightmapTex`) from shading calculations and by managing texture projection modes and normal deflection scaling.

## Operating characteristics
*   **Time Complexity:** Identical to standard SDF ray marching, $O(N)$ where $N$ is the number of march steps [G]. The overhead of returning and comparing 2D vectors is negligible [G].
*   **Memory Complexity:** Minimal increase; only the material ID needs to be stored per primitive [G].
*   **Strengths:** Enables complex, multi-colored scenes with smooth boolean operations without requiring explicit mesh generation or texture mapping on the geometry itself [G].
*   **Limitations:** Limited to a finite number of materials determined by the precision of the material ID storage and the resolution of the Color Ramp [G]. Cannot represent continuous material variation across a single object's surface unless the SDF itself encodes that variation [G].

## Background
This technique was popularized by Inigo Quilez in his articles on distance functions and raymarching, specifically demonstrating how to extend the `map()` function to return material data alongside distance [S1, unofficial]. It is widely used in GLSL raymarching demos and shader art to create visually rich scenes from simple geometric primitives [S1, unofficial].

## Sources
[S1] Raymarching Distance Fields: Concepts and Implementation, https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
