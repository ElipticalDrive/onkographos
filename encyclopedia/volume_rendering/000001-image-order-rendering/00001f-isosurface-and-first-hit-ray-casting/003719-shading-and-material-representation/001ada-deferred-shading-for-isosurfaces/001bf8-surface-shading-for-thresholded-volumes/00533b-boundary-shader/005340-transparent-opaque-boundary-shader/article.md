# Transparent-opaque Boundary Shader

Also called: Dual-threshold boundary shader

## Overview
The transparent-opaque boundary shader combines the logic of the boundary shader and the transparent boundary shader into a single shader, allowing different thresholds for opaque and transparent boundaries [S1]. It solves the problem of defining distinct visual boundaries for different material densities within a single rendering pass, enabling the user to control the visibility of internal structures versus outer surfaces simultaneously. It differs from the standard boundary shader by supporting two independent threshold values rather than one [S1].

## How it works
**Inputs and Outputs:**
*   Input: Volume data, ray origin, ray direction, opaque threshold ($T_{opaque}$), transparent threshold ($T_{transparent}$).
*   Output: Pixel color and opacity.

**Data Structures:**
*   Volume data: 3D grid of scalar values.
*   Thresholds: Two scalar values, $T_{opaque}$ and $T_{transparent}$.

**Steps:**
1.  Trace the ray through the volume data.
2.  Check the volume value at the current sample point against $T_{opaque}$ and $T_{transparent}$.
3.  If the value exceeds $T_{opaque}$, treat this as the opaque boundary. Interpolate the exact position where the value equals $T_{opaque}$ using the last lower measured value [S1].
4.  If the value exceeds $T_{transparent}$ but not $T_{opaque}$, treat this as the transparent boundary. Interpolate the exact position where the value equals $T_{transparent}$ using the last lower measured value [S1].
5.  Illuminate the pixel based on the interpolated position, the precomputed volume normal at that point, and lighting settings [S1].
6.  Combine the results from the opaque and transparent boundary detections into a single pixel color [S1].

**Parameters:**
*   Opaque threshold ($T_{opaque}$): Threshold for opaque boundary detection [S1].
*   Transparent threshold ($T_{transparent}$): Threshold for transparent boundary detection [S1].

**Edge Cases and Pitfalls:**
*   If $T_{opaque} \le T_{transparent}$, the behavior may be undefined or result in no transparent boundary being detected if the opaque boundary is always hit first. The sources do not specify the required relationship between the two thresholds [S1].
*   Precision issues may arise if the volume values are close to the thresholds, requiring careful interpolation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transparent_opaque_boundary_shader(ray_origin, ray_direction, volume_data, T_opaque, T_transparent):
    # Trace ray through volume
    t = 0.0
    step_size = get_step_size(volume_data)
    prev_value_opaque = 0.0
    prev_value_transparent = 0.0
    hit_opaque = False
    hit_transparent = False
    hit_pos_opaque = None
    hit_pos_transparent = None
    
    while t < max_distance:
        pos = ray_origin + ray_direction * t
        value = sample_volume(volume_data, pos)
        
        # Check for opaque boundary
        if not hit_opaque and value >= T_opaque:
            # Interpolate exact position
            hit_pos_opaque = interpolate_position(t, prev_value_opaque, value, T_opaque)
            hit_opaque = True
            
        # Check for transparent boundary
        if not hit_transparent and value >= T_transparent:
            # Interpolate exact position
            hit_pos_transparent = interpolate_position(t, prev_value_transparent, value, T_transparent)
            hit_transparent = True
            
        # Update previous values for interpolation
        prev_value_opaque = value
        prev_value_transparent = value
        
        t += step_size
        
    # Compute normals at hit positions
    normal_opaque = compute_normal(volume_data, hit_pos_opaque) if hit_opaque else None
    normal_transparent = compute_normal(volume_data, hit_pos_transparent) if hit_transparent else None
    
    # Illuminate and combine
    color_opaque = 0.0
    color_transparent = 0.0
    
    if hit_opaque:
        color_opaque = apply_lighting(hit_pos_opaque, normal_opaque, lighting_settings)
        
    if hit_transparent:
        color_transparent = apply_lighting(hit_pos_transparent, normal_transparent, lighting_settings)
        
    # Combine results (specific combination method not detailed in sources)
    final_color = combine_colors(color_opaque, color_transparent)
    
    return final_color
```

## Subtopics

### Shaded Surface Display (SSD)
Shaded Surface Display (SSD) is a surface rendering technique adapted for the volume rendering pipeline by setting low opacity only at steep gradients, creating a hard, transparent boundary representation of bone or skin [S2, unofficial]. It is chosen when a hard, transparent boundary is desired for specific structures like bone or skin, leveraging the gradient information to control opacity [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, similar to the boundary shader [S1].
*   **Memory Complexity:** Constant, requiring storage for two threshold values and intermediate hit positions [S1].
*   **Strengths:** Allows simultaneous visualization of opaque and transparent boundaries with different thresholds [S1].
*   **Limitations:** The specific method for combining the opaque and transparent boundary results is not detailed in the sources [S1]. The relationship between the two thresholds is not specified [S1].
*   **When to use:** When distinct visual boundaries for different material densities are needed in a single pass [S1].
*   **When not to use:** When a single threshold is sufficient or when the combination of opaque and transparent boundaries is not required [S1].

## Background
The transparent-opaque boundary shader was introduced in the context of volume rendering in architecture, where overlapping and combining 3D voxel volume data with 3D building models is relevant [S1]. It is part of a family of shaders that includes the boundary shader, transparent boundary shader, cross-section shader, maximum shader, and cloud shader [S1]. The boundary shader, which it extends, traces the ray until its value exceeds a user-defined threshold, interpolates the exact threshold position, and illuminates the pixel based on position, precomputed volume normal, and lighting settings [S1].

## Sources
[S1] Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models, https://doi.org/10.52842/conf.ecaade.2012.1.719, fetched 2026-10-26
[S2] The Principles and Practice of Computed Tomography Volume Rendering, https://www.flyriver.com/g/ct-volume-rendering, fetched 2026-10-28 (unofficial)
