# Per-Fragment Phong Lighting with Gradient Normals

Also called: Gradient-based Phong Shading, Phong Lighting with Gradient Normals, Phong Shading with Gradient Normals

## Overview
Per-fragment Phong Lighting with Gradient Normals is a volume rendering shading technique that enhances the visual perception of internal structures by applying Phong or Gouraud shading models to voxels based on normals derived from the scalar field's gradient [S1, S2]. It distinguishes itself from standard threshold-based shading by providing a "realistic" 3D view through dynamic lighting calculations that interact with the volume's internal density variations, rather than relying solely on surface boundaries [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (density values), a transfer function defining opacity/color, a light vector, and rendering parameters (threshold, interpolation mode, highlight toggle).
*   **Output:** A 2D image where each fragment's color is modulated by the Phong lighting model.

**Data Structures:**
*   **Scalar Field:** A 3D grid of density values. The sources indicate 12-bit precision (range 0–4095) [S2].
*   **Light Vector:** A 3D vector $(L_x, L_y, L_z)$ that moves with the view vector [S2].
*   **Normal Vector:** A 3D vector $(N_x, N_y, N_z)$ calculated per fragment.

**Steps:**
1.  **Ray Marching:** Perform standard volume ray marching to sample the scalar field along the ray [S2].
2.  **Gradient Calculation:** For each sample point (fragment) along the ray, calculate the normal vector $\mathbf{N}$ by computing the gradient of the scalar field $\phi$ at that point [S1, S2].
    *   The gradient is computed using finite differences. The specific interpolation method (nearest neighbor or trilinear) affects the accuracy of the density values used in this calculation [S2].
    *   Formula: $\mathbf{N} = \nabla \phi = \left( \frac{\partial \phi}{\partial x}, \frac{\partial \phi}{\partial y}, \frac{\partial \phi}{\partial z} \right)$ [S2].
    *   Normalize $\mathbf{N}$ to unit length.
3.  **Threshold Check:** Determine if the current sample's density value $\phi$ is above the user-defined threshold value $T$ [S2].
    *   If $\phi < T$, shading is not applied (or the fragment is treated as background/transparent depending on the transfer function's opacity at that density) [S2].
    *   If $\phi \ge T$, proceed to lighting calculation.
4.  **Lighting Calculation:** Calculate the shading color using the Phong or Gouraud model [S2].
    *   Let $\mathbf{V}$ be the view vector (from fragment to camera).
    *   Let $\mathbf{L}$ be the light vector.
    *   Let $\mathbf{R}$ be the reflection vector: $\mathbf{R} = 2(\mathbf{N} \cdot \mathbf{L})\mathbf{N} - \mathbf{L}$ [G] (standard Phong reflection).
    *   The diffuse component is proportional to $\max(0, \mathbf{N} \cdot \mathbf{L})$ [S2].
    *   The specular component is proportional to $\max(0, \mathbf{R} \cdot \mathbf{V})^{\text{shininess}}$ [S2].
    *   The final color is a combination of ambient, diffuse, and specular terms, modulated by the base color from the transfer function [S2].
5.  **Interpolation:** When calculating the gradient or shading colors, use either nearest neighbor or trilinear interpolation for the density values [S2]. Trilinear interpolation provides smoother gradients but may blur sharp features; nearest neighbor preserves sharpness but may introduce artifacts [S2].
6.  **Highlights:** If the "Shading with Highlights" option is enabled, include the specular term in the calculation [S2].

**Parameters:**
*   **Threshold Value ($T$):** Selects the density region for shading. Range: 0 to 4095 [S2].
    *   Typical values: ~400 for skin, ~1700 for bones, 3200 for teeth [S2].
*   **Light Vector:** Moves with the view vector [S2].
*   **Interpolation Mode:** Nearest neighbor or trilinear [S2].
*   **Highlight Toggle:** Boolean to enable/disable specular highlights [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_fragment_phong_gradient_normal(scalar_field, ray_origin, ray_direction, threshold, interpolation_mode, enable_highlights):
    # Initialize accumulated color and opacity
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    
    # Ray marching loop (simplified)
    for t in range(0, max_distance, step_size):
        pos = ray_origin + t * ray_direction
        density = sample_scalar_field(scalar_field, pos, interpolation_mode)
        
        # Threshold check
        if density < threshold:
            continue
            
        # Calculate normal from gradient
        grad_x = compute_gradient_x(scalar_field, pos, interpolation_mode)
        grad_y = compute_gradient_y(scalar_field, pos, interpolation_mode)
        grad_z = compute_gradient_z(scalar_field, pos, interpolation_mode)
        
        normal = normalize((grad_x, grad_y, grad_z))
        
        # View and Light vectors
        view_vector = normalize(-ray_direction) # Assuming camera at origin looking down -Z or similar
        light_vector = get_light_vector(view_vector) # Light moves with view
        
        # Phong Lighting Model
        # Diffuse
        diff = max(0.0, dot(normal, light_vector))
        
        # Specular (if enabled)
        spec = 0.0
        if enable_highlights:
            reflection = reflect(light_vector, normal)
            spec = pow(max(0.0, dot(reflection, view_vector)), shininess)
            
        # Combine lighting
        shading_factor = ambient + diffuse_scale * diff + specular_scale * spec
        
        # Get base color from transfer function
        base_color = transfer_function_color(density)
        base_opacity = transfer_function_opacity(density)
        
        # Apply shading to color
        shaded_color = base_color * shading_factor
        
        # Alpha compositing (front-to-back)
        alpha = base_opacity * step_size * density_scale # Simplified opacity model
        accumulated_color = accumulated_color + shaded_color * alpha * (1.0 - accumulated_opacity)
        accumulated_opacity = accumulated_opacity + alpha * (1.0 - accumulated_opacity)
        
        if accumulated_opacity > 0.99:
            break
            
    return accumulated_color

def sample_scalar_field(field, pos, mode):
    if mode == 'nearest':
        return field[nearest_voxel(pos)]
    elif mode == 'trilinear':
        return trilinear_interpolate(field, pos)
    else:
        raise ValueError("Unknown interpolation mode")

def compute_gradient_x(field, pos, mode):
    # Finite difference approximation
    # Note: Exact implementation depends on grid spacing and boundary handling
    # [G] Assuming uniform grid spacing of 1.0 for simplicity
    val_right = sample_scalar_field(field, pos + (1.0, 0.0, 0.0), mode)
    val_left = sample_scalar_field(field, pos - (1.0, 0.0, 0.0), mode)
    return (val_right - val_left) / 2.0

def compute_gradient_y(field, pos, mode):
    val_up = sample_scalar_field(field, pos + (0.0, 1.0, 0.0), mode)
    val_down = sample_scalar_field(field, pos - (0.0, 1.0, 0.0), mode)
    return (val_up - val_down) / 2.0

def compute_gradient_z(field, pos, mode):
    val_front = sample_scalar_field(field, pos + (0.0, 0.0, 1.0), mode)
    val_back = sample_scalar_field(field, pos - (0.0, 0.0, 1.0), mode)
    return (val_front - val_back) / 2.0

def reflect(light, normal):
    # R = 2(N.L)N - L
    dot_nl = dot(normal, light)
    return (2.0 * dot_nl * normal) - light
```

## Subtopics

### [Diffuse Lighting for Volume Ray Casting](6760-diffuse-lighting-for-volume-ray-casting.md)
Diffuse Lighting for Volume Ray Casting illuminates a volume by calculating normals for every voxel using finite differences of the scalar field, then applying a diffuse lighting model. It provides geometric cues that allow standard diffuse lighting models to interact with the volume's internal structure.

### [Phong Shading with Boundary Enhancement and Facing Subtraction](15408-phong-shading-with-boundary-enhancement-and-facing-subtracti.md)
This method extends per-fragment Phong lighting by applying two alpha-modulation techniques to enhance the visual perception of volume boundaries: boundary enhancement via absorption approximation and facing subtraction for silhouette definition. It distinguishes itself from standard gradient-based Phong shading by explicitly modifying the fragment alpha based on the gradient magnitude and the view direction relative to the surface normal.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples along all rays. The gradient calculation adds a constant factor (typically 6-8 extra samples for central differences) per fragment [S2].
*   **Memory Complexity:** $O(V)$ where $V$ is the volume of the scalar field, required to store the density values [S2].
*   **Performance:** The method is interactive when combined with efficient ray marching and transfer function pre-integration, as noted in the evaluation of the dual-domain interface [S1]. The choice of interpolation (nearest vs. trilinear) affects both quality and speed [S2].
*   **Strengths:** Provides a realistic 3D impression by leveraging internal density gradients, not just surfaces [S2]. Allows selective shading of specific tissue types via thresholding [S2].
*   **Limitations:** Can only visualize one tissue type at a time if using a single threshold, unlike full volume rendering which assigns opacity to all tissues [S2]. Gradient noise can lead to noisy normals if the scalar field is not smooth [G].
*   **When to use:** When a realistic, shaded appearance of specific internal structures (e.g., bones, skin) is desired, and the user wants to control which density ranges are illuminated [S2].
*   **When not to use:** When simultaneous visualization of multiple tissue types with varying opacities is required (use full Volume Rendering instead) [S2].

## Background
This method originates from early volume visualization applications, specifically the "VOLUME RENDERING Application" by Thomas Lidy (2001-2002) [S2]. It was designed to provide a "realistic" view of medical datasets (human skin, bones, teeth) by applying Phong/Gouraud shading to density gradients [S2]. It was later referenced in the context of flexible transfer function design tools, where it served as a shading option for volume exploration [S1]. The technique relies on the principle that the gradient of a scalar field approximates the surface normal, even for implicit surfaces defined by the field [S1, S2].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
