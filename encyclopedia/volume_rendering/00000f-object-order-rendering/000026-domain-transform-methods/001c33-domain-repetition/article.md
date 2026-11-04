# Domain Repetition

Also called: Modular repetition, Tiling via modulo

## Overview
Domain Repetition is a technique for creating infinite, periodic repetitions of a geometric object in space by wrapping raymarching coordinates using modulo arithmetic [S1, unofficial]. It solves the problem of rendering tiled or infinite scenes without increasing memory or geometry complexity, by mapping arbitrary world coordinates into a single repeating unit cell [S1, unofficial]. It sets itself apart from [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md) and [GPU-based 2D FFT](841-gpu-based-2d-fft.md) by operating directly in the spatial domain via coordinate transformation rather than transforming the volume data into the frequency domain [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D point $p = (p_x, p_y, p_z)$ representing the current raymarching position, and a repetition size $s = (s_x, s_z)$ defining the dimensions of the repeating unit in the x and z axes [S1, unofficial].
*   Output: A transformed point $p'$ where the x and z coordinates are wrapped into the interval $[-0.5, 0.5]$ centered at the origin, while $p_y$ remains unchanged [S1, unofficial].

**Data Structures:**
*   No persistent data structures are required. The method is a pure function applied per-sample during raymarching [S1, unofficial].

**Steps:**
1.  Apply the modulo operation to the x and z coordinates of the point $p$ using the repetition size $s_x$ and $s_z$ [S1, unofficial].
2.  Shift the result by $-0.5$ (or $-s_x/2$ and $-s_z/2$) to center the repeated domain around the origin [S1, unofficial].
3.  Evaluate the base distance field function (e.g., `cube`) on the transformed point $p'$ [S1, unofficial].

**Formula:**
The transformation for the x and z coordinates is defined as [S1, unofficial]:
$$ p'_x = \text{mod}(p_x, s_x) - \frac{s_x}{2} $$
$$ p'_z = \text{mod}(p_z, s_z) - \frac{s_z}{2} $$
The y coordinate remains unchanged:
$$ p'_y = p_y $$

**Parameters:**
*   $s_x, s_z$: The size of the repeating tile. In the source example, $s_x = 1$ and $s_z = 1$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Artifacts:** Distorting the distance field via modulo can cause "nasty artifacts" in raymarching, such as incorrect normals or missed collisions, because the distance field is no longer smooth across tile boundaries [S1, unofficial].
*   **Mitigation:** A suggested solution is to step shorter in the raymarching function than the distance field returns, effectively adding a safety margin to prevent tunneling through tile boundaries [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def repeat(p: vec3, gx: float, gz: float) -> vec3:
    """
    Repeats the domain in x and z by size gx and gz.
    Centers the result in [-0.5, 0.5].
    """
    # Apply modulo and shift to center
    p.x = mod(p.x, gx) - gx * 0.5
    p.z = mod(p.z, gz) - gz * 0.5
    return p

def repeated_cubes(p: vec3) -> float:
    """
    Evaluates a cube SDF on a tiled domain.
    """
    p = repeat(p, 1.0, 1.0)
    return cube(p)
```

## Subtopics

### [Space Repetition](1677-space-repetition.md)
A technique to repeat objects in space by modifying the position p before querying the SDF. The text notes this is also 'cheating' and allows repeating objects 'for free'.

### [Polar Modulo](3358-polar-modulo.md)
Polar Modulo repeats a 2D pattern around the origin by a fixed angle determined by the number of repetitions. It builds on [Domain Repetition](1218-domain-repetition.md) by applying modulo arithmetic to the angular coordinate rather than Cartesian coordinates, enabling radial symmetry. Unlike [Space Repetition](1677-space-repetition.md) which tiles in linear space, Polar Modulo creates concentric or angular sectors.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per raymarching step, as it involves only basic arithmetic operations [S1, unofficial].
*   **Memory Complexity:** $O(1)$, as no additional data structures are stored [S1, unofficial].
*   **Strengths:** Extremely efficient for creating infinite tiled scenes; no increase in geometry or memory usage [S1, unofficial].
*   **Limitations:** Can introduce artifacts in raymarching due to discontinuities in the distance field at tile boundaries [S1, unofficial].
*   **When to use:** When rendering infinite grids, tiled floors, or periodic patterns in raymarched scenes [S1, unofficial].
*   **When not to use:** When high precision is required at tile boundaries or when the distance field is complex and prone to artifacts [S1, unofficial].

## Background
Domain Repetition is a technique popularized in the context of raymarching signed distance fields (SDFs) for real-time graphics [S1, unofficial]. It was described in the blog post "Raymarching Distance Fields" by Inigo Quilez in 2013 [S1, unofficial]. The method leverages the periodicity of the modulo operator to create infinite repetitions of simple geometric primitives [S1, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
