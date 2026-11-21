# Boundary Shader

Also called: Thresholded Boundary Shader, First-Hit Boundary Shader

## Overview
The Boundary Shader is an image-order rendering method for isosurfaces that identifies the first intersection of a ray with a volume data set exceeding a user-defined threshold, interpolates the exact entry position, and computes illumination based on precomputed normals [S1]. It differs from standard volume raycasting by rendering only the surface boundary rather than integrating along the entire ray, and from surface extraction methods by performing the intersection test and shading in a single pass without generating an intermediate mesh [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data set (scalar field), user-defined threshold value $T$, lighting parameters (light position, color, material properties), precomputed volume normals at voxel centers or interpolated positions.
*   **Output:** Pixel color (RGB) representing the illuminated surface at the threshold boundary.

**Data Structures:**
*   **Volume Data:** A 3D array of scalar values $V(x, y, z)$.
*   **Normals:** Precomputed normal vectors $\mathbf{n}(x, y, z)$ associated with the volume data, typically derived from gradients of the scalar field.
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.

**Steps:**
1.  **Ray Tracing:** Trace the ray through the volume data from the camera origin along the viewing direction.
2.  **Threshold Check:** For each sampled point along the ray, retrieve the scalar value $v$ from the volume data.
3.  **Boundary Detection:** Check if the current value $v$ exceeds the user-defined threshold $T$.
    *   If $v \le T$, continue to the next sample.
    *   If $v > T$, a boundary has been crossed.
4.  **Interpolation:** Calculate the exact position $\mathbf{p}$ of the threshold crossing. This is done by linearly interpolating between the last sample position $\mathbf{p}_{prev}$ (where value $v_{prev} \le T$) and the current sample position $\mathbf{p}_{curr}$ (where value $v_{curr} > T$).
    *   Let $t$ be the interpolation factor:
        $$ t = \frac{T - v_{prev}}{v_{curr} - v_{prev}} $$
    *   The exact position is:
        $$ \mathbf{p} = \mathbf{p}_{prev} + t (\mathbf{p}_{curr} - \mathbf{p}_{prev}) $$
5.  **Normal Retrieval:** Retrieve or interpolate the precomputed volume normal $\mathbf{n}$ at position $\mathbf{p}$.
6.  **Illumination:** Compute the pixel color using the position $\mathbf{p}$, normal $\mathbf{n}$, and lighting settings (e.g., Phong shading model).
7.  **Termination:** Abort the ray tracing for this pixel, as the first hit is rendered.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundary_shader(ray_origin, ray_direction, volume_data, threshold, lighting_model):
    # Assume volume data is sampled at regular intervals along the ray
    step_size = get_step_size(volume_data)
    max_steps = get_max_steps(volume_data)
    
    current_pos = ray_origin
    prev_val = None
    prev_pos = None
    
    for i in range(max_steps):
        # Sample volume value at current position
        val = sample_volume(volume_data, current_pos)
        
        # Check if threshold is exceeded
        if val > threshold:
            # Interpolate exact boundary position
            # prev_val is guaranteed to be <= threshold from previous iteration
            if prev_val is not None:
                t = (threshold - prev_val) / (val - prev_val)
                boundary_pos = prev_pos + t * (current_pos - prev_pos)
                
                # Get precomputed normal at boundary position
                normal = get_normal(volume_data, boundary_pos)
                
                # Compute illumination
                color = lighting_model.compute(boundary_pos, normal, ray_direction)
                return color
            else:
                # Edge case: first sample exceeds threshold
                boundary_pos = current_pos
                normal = get_normal(volume_data, boundary_pos)
                color = lighting_model.compute(boundary_pos, normal, ray_direction)
                return color
        
        # Store previous state for interpolation
        prev_val = val
        prev_pos = current_pos
        
        # Move to next sample
        current_pos = current_pos + ray_direction * step_size
        
    # No boundary found
    return background_color
```

**Parameters:**
*   **Threshold ($T$):** A user-defined scalar value that determines the isovalue for the surface. Typical values depend on the data range (e.g., 0.5 for normalized data) [S1].

**Edge Cases and Pitfalls:**
*   **First Sample Exceeds Threshold:** If the ray origin is already inside the thresholded region, the interpolation step requires a "previous" value. This can be handled by assuming a value below the threshold at the ray origin or by using the first sample as the boundary position [G].
*   **Precision:** Linear interpolation assumes a linear transition between samples. If the threshold crossing occurs between samples with steep gradients, the interpolated position may be inaccurate.
*   **Normal Accuracy:** The quality of the shading depends heavily on the accuracy of the precomputed normals. If normals are not aligned with the true surface gradient, shading artifacts (e.g., incorrect specular highlights) will occur.

## Subtopics

### Transparent-opaque Boundary Shader
A variant that combines the boundary shader and the transparent boundary shader into a single shader, allowing different thresholds for opaque and transparent boundaries [S1]. It uses an opaque threshold for detecting the primary surface and a transparent threshold for detecting secondary or translucent boundaries, enabling more complex material representations [S1].

### Transparent Boundary Shader
A variant that does not abort after the first threshold is found. Instead, it continues to trace the ray, searching for further thresholds. The lighting results for all found thresholds are accumulated and added up for display [S1]. This allows for the visualization of multiple internal structures or layered boundaries within the volume.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of samples along the ray until the threshold is exceeded. In the worst case (no threshold exceeded or threshold at the end), it is $O(N_{max})$.
*   **Memory Complexity:** $O(1)$ per pixel for the shader execution, assuming volume data is accessed on-demand or cached. Precomputed normals require additional memory proportional to the volume size.
*   **Performance:** Fast for rendering single surfaces as it terminates early. Performance degrades if the threshold is set high, requiring more samples to find the boundary.
*   **Strengths:** Simple implementation, efficient for single-surface rendering, provides smooth shading via interpolated positions and normals.
*   **Limitations:** Only renders the first hit; cannot visualize internal structures without the transparent variant. Accuracy depends on sampling rate and interpolation.

## Background
The Boundary Shader was introduced in the context of architectural volume rendering, specifically for overlapping and combining 3D voxel volume data with 3D building models [S1]. It is part of a suite of shaders designed for a design tool development environment, allowing users to quickly determine intersections with the front-most surface of a 3D building model [S1]. The method is described in the paper "Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models" presented at eCAADe 2012 [S1].

## Sources
[S1] Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models, https://doi.org/10.52842/conf.ecaade.2012.1.719, fetched 2026-10-26
