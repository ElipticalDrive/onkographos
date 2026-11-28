# Axis-Aligned Clipping

Also called: Boundary clipping, Mesh clipping, Six-Plane Axis-Aligned Clipping

## Overview
Axis-Aligned Clipping hides portions of a 3D scene by defining up to six axis-aligned clipping planes to expose interior geometry [S1]. It differs from general plane clipping by restricting plane orientations to the coordinate axes, simplifying the intersection logic and parameterization [S1]. This method builds on [Axis Threshold Clipping](11577-axis-threshold-clipping.md) by applying the threshold logic specifically to spatial boundaries rather than data values.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D scene (meshes or volumes), and six scalar values defining the clipping boundaries: $x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$ [S1].
*   **Outputs:** A rendered image where geometry outside the defined box is discarded [S1].

**Data Structures:**
The clipping state is defined by six scalar values. An integer parameter `clip_plane_count` indicates how many of the six possible planes are active [S1].

**Steps:**
1.  Define the six axis-aligned clipping planes based on the input values [S1].
2.  For each primitive or sample point in the scene, determine its position relative to the planes [S1].
3.  A point is considered "inside" if its coordinates satisfy:
    $$x_{min} \le x \le x_{max}$$
    $$y_{min} \le y \le y_{max}$$
    $$z_{min} \le z \le z_{max}$$
    [S1]
4.  Render only the geometry that lies on the interior side of all active clipping planes [S1].

**Parameters:**
*   `clip_plane_count`: An integer up to 6, controlling the number of active planes [S1].
*   Boundary coordinates: Specific to the scene scale [S1].

**Pitfalls:**
*   **Discarding in Fragment Shader:** In ray-marching contexts, discarding the pixel in the fragment shader prevents it from being rendered, resulting in black areas instead of showing the inner material [S2, unofficial]. The correct approach is to prevent color and alpha accumulation when the ray position is outside the clipping box [S2, unofficial].

**Pseudocode:**
The following pseudocode illustrates the boundary check logic used in ray-marching implementations to avoid the black-area pitfall [S2, unofficial]. All helpers are defined to ensure self-containment.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_clip(volume, ray_origin, ray_direction, step_size, max_steps, x_min, x_max, y_min, y_max, z_min, z_max):
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    current_pos = ray_origin
    for _ in range(max_steps):
        if x_min <= current_pos.x <= x_max and y_min <= current_pos.y <= y_max and z_min <= current_pos.z <= z_max:
            color_sample, alpha_sample = sample_volume(volume, current_pos)
            accumulated_color += color_sample * alpha_sample * (1.0 - accumulated_alpha)
            accumulated_alpha += alpha_sample * (1.0 - accumulated_alpha)
            if accumulated_alpha >= 1.0:
                break
        current_pos += ray_direction * step_size
    return accumulated_color

def sample_volume(volume, pos):
    # Standard trilinear interpolation or texture lookup on the volume data [G]
    return volume.lookup(pos), volume.density_lookup(pos)
```

## Subtopics

### [Ray-marching boundary clipping](18415-ray-marching-boundary-clipping.md)
A technique to clip a volume during ray marching by preventing color and alpha accumulation when the ray position is outside a defined clipping box, rather than discarding the final pixel in the fragment shader. It differs from standard clipping by handling the exclusion at the sample level, ensuring the interior remains visible without black artifacts. Choose this when implementing custom volume renderers in shaders where standard depth clipping is insufficient or causes visual artifacts.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Only supports axis-aligned cuts; cannot handle arbitrary angled slices [S1].
*   **When to use:** When the user needs to quickly inspect the interior of a scene by cutting away outer layers along the primary axes [S1].
*   **When not to use:** When arbitrary orientation clipping is required [S1].

## Background
This method is prominently featured in Smokeview, a tool for visualizing Fire Dynamics Simulation data [S1]. It allows users to define up to six clipping planes to alleviate the difficulty of viewing the interior of complicated geometries [S1]. The method is implemented via a dialog box that allows users to specify clipping values for each axis [S1].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume I : User’s Guide, https://doi.org/10.6028/nist.sp.1017-1, fetched 2026-10-26
[S2] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
