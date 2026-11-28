# Boundary Shader


## Overview
The Boundary Shader traces a ray through volume data until the scalar value exceeds a user-defined threshold, interpolates the exact crossing position, and illuminates the pixel using that position, a precomputed volume normal, and lighting settings [S1]. It differs from full volume raycasting by terminating at the first threshold crossing rather than integrating along the entire ray [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data set (scalar field), user-defined threshold value $T$, lighting parameters, and precomputed volume normals [S1].
*   **Output:** Pixel color representing the illuminated surface at the threshold boundary [S1].

**Data Structures:**
*   **Volume Data:** A 3D array of scalar values $V(x, y, z)$ [G].
*   **Normals:** Precomputed normal vectors $\mathbf{n}(x, y, z)$ associated with the volume data [S1].
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].

**Steps:**
1.  Trace the ray along the volume data [S1].
2.  Check if the current measured value is higher than the user-defined threshold [S1].
3.  If the threshold is exceeded, interpolate the exact position of the threshold using the last lower measured value [S1].
4.  Illuminate the pixel using the interpolated position, the precomputed volume normal at that point, and the lighting settings [S1].

**Formulas:**
Let $v_{prev}$ and $\mathbf{p}_{prev}$ be the value and position of the last sample below the threshold, and $v_{curr}$ and $\mathbf{p}_{curr}$ be the current sample above the threshold [G]. The interpolation factor is $t = \frac{T - v_{prev}}{v_{curr} - v_{prev}}$ [G]. The exact boundary position is $\mathbf{p} = \mathbf{p}_{prev} + t (\mathbf{p}_{curr} - \mathbf{p}_{prev})$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundary_shader(ray_origin, ray_direction, volume_data, threshold, lighting_model):
    step_size = 1.0 / volume_data.resolution  # [G]
    max_steps = 1000  # [G]
    
    current_pos = ray_origin
    prev_val = None
    prev_pos = None
    
    for i in range(max_steps):
        # Trilinear interpolation of scalar value at current_pos [G]
        val = volume_data.interpolate(current_pos)
        
        if val > threshold:
            if prev_val is not None:
                t = (threshold - prev_val) / (val - prev_val)
                boundary_pos = prev_pos + t * (current_pos - prev_pos)
            else:
                boundary_pos = current_pos  # [G]
            
            # Trilinear interpolation of precomputed normal at boundary_pos [G]
            normal = volume_data.normals.interpolate(boundary_pos)
            
            # Compute illumination using position, normal, and lighting settings [S1]
            color = lighting_model.compute(boundary_pos, normal, ray_direction)
            return color
        
        prev_val = val
        prev_pos = current_pos
        current_pos = current_pos + ray_direction * step_size
        
    return (0.0, 0.0, 0.0)  # [G]
```

**Parameters:**
*   **Threshold ($T$):** A user-defined scalar value that determines when the ray stops and illumination occurs [S1].

**Edge Cases and Pitfalls:**
*   **First Sample Exceeds Threshold:** If the ray origin is already inside the thresholded region, the interpolation step requires a previous value; this is handled by using the current position as the boundary [G].
*   **Precision:** Linear interpolation assumes a linear transition between samples, so steep gradients may cause inaccurate boundary positions [G].
*   **Normal Accuracy:** Shading quality depends heavily on the accuracy of the precomputed normals; misaligned normals cause artifacts [G].

## Subtopics

### [Transparent-opaque Boundary Shader](14931-transparent-opaque-boundary-shader.md)
Combines the boundary shader and transparent boundary shader into a single shader, allowing different thresholds for opaque and transparent boundaries. It solves the problem of defining distinct visual boundaries for different material densities within a single rendering pass, enabling the user to control the visibility of internal structures versus outer surfaces simultaneously. It differs from the standard boundary shader by supporting two independent threshold values rather than one.

### Transparent Boundary Shader
Works very much like the boundary shader but does not abort after the first threshold found; instead it searches for further thresholds on the ray and adds up the lighting results for display [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of samples until the threshold is exceeded, or $O(N_{max})$ in the worst case [G].
*   **Memory Complexity:** $O(1)$ per pixel for execution, plus $O(V)$ for precomputed normals [G].
*   **Performance:** Fast for single surfaces due to early termination, but degrades if the threshold requires many samples [G].
*   **Strengths:** Simple implementation and efficient single-surface rendering with smooth shading [G].
*   **Limitations:** Inability to visualize internal structures without the transparent variant and accuracy dependence on sampling rate [G].

## Background
It is described in the context of architectural volume rendering for overlapping and combining 3D voxel volume data with 3D building models [S1]. The method is part of a design tool development suite that allows users to quickly determine whether a ray intersects the front-most surface of a 3D building model [S1]. It was presented in the paper "Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models" at eCAADe 2012 [S1].

## Sources
[S1] Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models, https://doi.org/10.52842/conf.ecaade.2012.1.719, fetched 2026-10-26
