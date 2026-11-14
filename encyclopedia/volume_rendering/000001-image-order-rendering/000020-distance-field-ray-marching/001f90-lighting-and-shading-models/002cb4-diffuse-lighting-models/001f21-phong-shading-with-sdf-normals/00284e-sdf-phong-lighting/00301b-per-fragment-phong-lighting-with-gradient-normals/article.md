# Per-Fragment Phong Lighting with Gradient Normals

Also called: Gradient-based Phong Shading, Phong Lighting with Gradient Normals, Phong Shading with Gradient Normals

## Overview
Per-fragment Phong lighting with gradient normals is a volume shading technique that enhances the geometric perception of scalar field data by computing surface normals via finite differences of the scalar field and applying a Phong illumination model [S1, S2]. It distinguishes itself from parent methods by deriving normals directly from the data gradient rather than relying on pre-computed or analytical normals, allowing standard lighting models to interact with the volume's internal structure [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (density values), a threshold value, a light vector, and an interpolation mode.
*   **Output:** A shaded 2D image where voxels above the threshold are illuminated using Phong shading [S2].

**Data Structures:**
*   **Scalar Field:** A 3D grid of density values. The source specifies a range of 0 to 4095, implying 12-bit unsigned integer storage or similar fixed-point representation [S2].
*   **Light Vector:** A 3D vector that moves with the view vector [S2].
*   **Threshold:** A scalar value determining which density regions are shaded [S2].

**Steps:**
1.  **Gradient Calculation:** For each fragment (voxel) in the volume, calculate the normal vector $\mathbf{n}$ by computing the gradient of the scalar field $\phi$. This is typically done using finite differences [S1, S2].
    *   Formula: $\mathbf{n} = \nabla \phi = \left[ \frac{\partial \phi}{\partial x}, \frac{\partial \phi}{\partial y}, \frac{\partial \phi}{\partial z} \right]$ [S2].
    *   The gradient components are derived from neighboring density values. The specific finite difference scheme (central, forward, backward) is not explicitly defined in the sources, but the result is a normal vector representing the local surface orientation [S2].
2.  **Interpolation:** If the fragment position falls between grid points, interpolate the density values using either nearest neighbor or trilinear interpolation before calculating the gradient [S2].
3.  **Thresholding:** Compare the interpolated density value $\phi$ against the threshold value $T$. If $\phi < T$, the fragment is not shaded (or is transparent/unlit) [S2].
4.  **Lighting Calculation:** If $\phi \ge T$, calculate the shading color using the Phong or Gouraud shading model [S2].
    *   The Phong model combines ambient, diffuse, and specular components [S2].
    *   The diffuse component is calculated using the dot product of the normal $\mathbf{n}$ and the light vector $\mathbf{l}$: $C_{diffuse} = k_d (\mathbf{n} \cdot \mathbf{l})$ [S2].
    *   The specular component is enabled if "Shading with Highlights" is switched on [S2].
5.  **Rendering:** Apply the calculated shading color to the fragment [S2].

**Parameters:**
*   **Threshold Value ($T$):** Selects the density region for shading. Range: 0 to 4095 [S2].
    *   Typical values: ~400 for skin, ~1700 for bones, 3200 for teeth [S2].
*   **Light Vector ($\mathbf{l}$):** Moves with the view vector [S2].
*   **Interpolation Mode:** Nearest neighbor or trilinear interpolation for calculating shading colors [S2].
*   **Highlights:** Boolean flag to enable specular highlights [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_fragment_phong_gradient_shading(voxel_pos, scalar_field, threshold, light_vector, use_highlights, interpolation_mode):
    # Step 1: Interpolate density value at voxel_pos
    if interpolation_mode == 'nearest_neighbor':
        density = interpolate_nearest_neighbor(voxel_pos, scalar_field)
    elif interpolation_mode == 'trilinear':
        density = interpolate_trilinear(voxel_pos, scalar_field)
    else:
        density = 0

    # Step 2: Thresholding
    if density < threshold:
        return transparent_color # Or unlit color

    # Step 3: Calculate Normal from Gradient
    # Note: Exact finite difference scheme is not specified in sources [S2]
    # Assuming central differences for interior points
    grad_x = (scalar_field[voxel_pos + (1,0,0)] - scalar_field[voxel_pos + (-1,0,0)]) / 2.0
    grad_y = (scalar_field[voxel_pos + (0,1,0)] - scalar_field[voxel_pos + (0,-1,0)]) / 2.0
    grad_z = (scalar_field[voxel_pos + (0,0,1)] - scalar_field[voxel_pos + (0,0,-1)]) / 2.0
    
    normal = normalize(vec3(grad_x, grad_y, grad_z))

    # Step 4: Phong Lighting Calculation
    # Light vector moves with view vector [S2]
    light_dir = normalize(light_vector)
    
    # Diffuse component
    diffuse_intensity = max(0.0, dot(normal, light_dir))
    diffuse_color = diffuse_intensity * light_color
    
    # Specular component (if highlights enabled)
    specular_color = vec3(0.0, 0.0, 0.0)
    if use_highlights:
        view_dir = normalize(camera_position - voxel_pos)
        reflect_dir = reflect(-light_dir, normal)
        specular_intensity = pow(max(0.0, dot(view_dir, reflect_dir)), shininess)
        specular_color = specular_intensity * light_color

    # Combine components
    shaded_color = ambient_color + diffuse_color + specular_color
    
    return shaded_color
```

## Subtopics

### [Diffuse Lighting for Volume Ray Casting](6760-diffuse-lighting-for-volume-ray-casting.md)
Diffuse Lighting for Volume Ray Casting illuminates a volume by calculating normals for every voxel using finite differences of the scalar field, then applying a diffuse lighting model. It provides geometric cues that allow standard diffuse lighting models to interact with the volume's internal structure.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels processed, as each voxel requires gradient calculation and lighting computation [S2].
*   **Memory Complexity:** Requires storage for the scalar field and potentially temporary buffers for gradient calculation [S2].
*   **Performance:** Interactive performance is maintained by using efficient interpolation and lighting calculations [S1].
*   **Strengths:** Provides realistic geometric cues and enhances the perception of shape and structure within the volume [S2].
*   **Limitations:** Can only visualize one tissue type at a time when using a single threshold, unlike full volume rendering which assigns opacity to multiple tissues [S2].
*   **When to use:** When a realistic 3D view of a specific density region (e.g., skin, bones) is desired with minimal computational overhead [S2].
*   **When not to use:** When multiple tissue types with varying opacities need to be visualized simultaneously [S2].

## Background
This method was described in the context of a volume rendering application by Thomas Lidy in 2001-2002 [S2]. It is also referenced in work on flexible transfer function design for volume visualization, where it is used to enhance the visual quality of scalar field data [S1]. The technique leverages the gradient of the scalar field to estimate surface normals, a common approach in volume shading to provide geometric information that is otherwise lost in volume rendering [S1, S2].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
