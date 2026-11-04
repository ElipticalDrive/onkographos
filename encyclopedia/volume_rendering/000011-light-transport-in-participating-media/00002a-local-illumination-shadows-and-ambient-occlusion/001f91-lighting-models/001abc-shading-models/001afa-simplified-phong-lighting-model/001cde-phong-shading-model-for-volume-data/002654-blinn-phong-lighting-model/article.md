# Blinn-Phong Lighting Model

Also called: Blinn-Phong Lighting on Isosurface, Blinn-Phong Shading, Blinn-Phong Shading for Volume Data, Isosurface Shading, Volumetric Shading

## Overview
The Blinn-Phong Lighting Model applies the Blinn-Phong shading approximation to volume rendering data, typically by computing surface normals from the scalar field gradient to simulate isosurface shading [S1, S2, unofficial]. It enhances the perception of depth and shape structure in volumetric data by combining ambient, diffuse, and specular lighting components, offering a computationally cheap alternative to full ray-traced shadows while providing stronger visual cues than unshaded volume rendering [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume data $V$, light direction vector $\mathbf{l}$, view direction vector $\mathbf{v}$, lighting coefficients ($k_a, k_d, k_s$), and light intensity $I_L$.
*   **Output:** Shaded color $I$ for each sample point or voxel.

**Data Structures:**
*   **Scalar Volume:** 3D texture or array storing scalar values $s(x,y,z)$.
*   **Gradient Data (Optional):** If precomputed, a 3D texture storing the gradient vector $\nabla s$ or its components. If not precomputed, gradients are derived on-the-fly from the scalar volume.

**Steps:**

1.  **Compute Surface Normal:**
    Calculate the surface normal $\mathbf{n}$ at the current sample point by computing the gradient of the scalar field. The sources specify using central differences between neighboring voxels [S2, unofficial]. Let $x$ be the voxel spacing (or distance between sample points). The gradient $\nabla I$ is computed as:
    $$ \nabla I = \left( \frac{I_{right} - I_{left}}{2x}, \frac{I_{top} - I_{bottom}}{2x}, \frac{I_{front} - I_{back}}{2x} \right) $$
    Normalize the result to obtain the unit normal vector $\mathbf{n}$:
    $$ \mathbf{n} = \frac{\nabla I}{\|\nabla I\|} $$
    [S2, unofficial]

2.  **Calculate Diffuse Component:**
    Compute the diffuse lighting contribution based on the angle between the light direction and the surface normal. The light direction $\mathbf{l}$ (or $\mathbf{lightDir}$) must be normalized [S1, unofficial].
    $$ \text{diffuseStrength} = \max(\mathbf{n} \cdot \mathbf{l}, 0.0) $$
    [S1, unofficial]

3.  **Calculate Specular Component:**
    Compute the specular highlight. The sources describe two variations for the reflection vector. One approach uses the reflection of the light vector about the normal [S1, unofficial]:
    $$ \mathbf{r} = \text{reflect}(-\mathbf{l}, \mathbf{n}) $$
    $$ \text{specularStrength} = \max(0.0, \mathbf{v} \cdot \mathbf{r})^{\text{shininess}} $$
    where $\mathbf{v}$ is the normalized view direction (ray origin direction) and the exponent is typically 64.0 [S1, unofficial].
    
    *Note:* The standard Blinn-Phong model uses the half-vector $\mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|}$ and computes $\max(0.0, \mathbf{n} \cdot \mathbf{h})^{\text{shininess}}$. The source [S2, unofficial] presents the general Blinn-Phong formula using $\mathbf{h}$, while [S1, unofficial] implements the reflection vector variant. Both are cited as Blinn-Phong shading in the context of volume rendering.

4.  **Combine Lighting:**
    Composite the shaded colors using the Blinn-Phong illumination model equation [S2, unofficial]:
    $$ I = k_a + I_L k_d (\mathbf{l} \cdot \mathbf{n}) + I_L k_s (\mathbf{h} \cdot \mathbf{n}) $$
    Or, using the specific implementation details from [S1, unofficial] which scales the base color:
    $$ \text{color.rgb} = (\text{diffuseStrength} + \text{specularStrength}) \times \text{val\_color.rgb} $$
    where $\text{val\_color}$ is the base color of the volume at that point, potentially modulated by alpha/transparency [S1, unofficial]. The ambient term $k_a$ is often added as a constant or base color if not included in the scaling factor.

**Parameters:**
*   $k_a, k_d, k_s$: Ambient, diffuse, and specular coefficients. User-set [S2, unofficial].
*   $I_L$: Light intensity [S2, unofficial].
*   Shininess: Exponent for specular term. Source [S1, unofficial] uses 64.0.
*   $x$: Voxel spacing/distance between neighbors for gradient calculation [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Noise:** Computing normals on-the-fly requires 6 texture fetches per calculation (3 pairs of neighbors) [S2, unofficial]. This can be noisy if the volume data is sparse or low-resolution.
*   **Memory Trade-off:** Precalculating normals on the host and storing them in voxel data requires 4x texture memory (3 components for normal + 1 for value, or separate gradient textures) [S2, unofficial].
*   **Boundary Conditions:** Gradient calculation at the volume boundaries requires handling missing neighbors (e.g., clamping or zero-padding) [G].
*   **View Direction:** The view direction $\mathbf{v}$ must be normalized [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blinn_phong_shading(volume, sample_point, light_dir, view_dir, k_a, k_d, k_s, I_L, shininess, voxel_spacing):
    """
    Computes Blinn-Phong shading for a volume sample.
    
    Args:
        volume: 3D array or texture of scalar values.
        sample_point: (x, y, z) coordinates of the current sample.
        light_dir: Normalized light direction vector.
        view_dir: Normalized view direction vector.
        k_a: Ambient coefficient.
        k_d: Diffuse coefficient.
        k_s: Specular coefficient.
        I_L: Light intensity.
        shininess: Specular exponent.
        voxel_spacing: Distance between voxel centers.
        
    Returns:
        Shaded color value.
    """
    # 1. Compute Normal via Central Differences
    # Get neighboring voxel values
    # Note: Boundary checks required in implementation [G]
    val_left = volume.get(sample_point.x - voxel_spacing, sample_point.y, sample_point.z)
    val_right = volume.get(sample_point.x + voxel_spacing, sample_point.y, sample_point.z)
    val_bottom = volume.get(sample_point.x, sample_point.y - voxel_spacing, sample_point.z)
    val_top = volume.get(sample_point.x, sample_point.y + voxel_spacing, sample_point.z)
    val_back = volume.get(sample_point.x, sample_point.y, sample_point.z - voxel_spacing)
    val_front = volume.get(sample_point.x, sample_point.y, sample_point.z + voxel_spacing)
    
    grad_x = (val_right - val_left) / (2 * voxel_spacing)
    grad_y = (val_top - val_bottom) / (2 * voxel_spacing)
    grad_z = (val_front - val_back) / (2 * voxel_spacing)
    
    normal = normalize(vec3(grad_x, grad_y, grad_z))
    
    # 2. Calculate Diffuse Component
    diffuse_strength = max(dot(normal, light_dir), 0.0)
    
    # 3. Calculate Specular Component
    # Using reflection vector as per [S1, unofficial]
    # reflect(-light_dir, normal) computes the reflection of the light vector
    reflect_dir = reflect(-light_dir, normal)
    specular_strength = pow(max(dot(view_dir, reflect_dir), 0.0), shininess)
    
    # 4. Combine Lighting
    # Base color from volume (simplified as 1.0 for scalar, or fetched from color map)
    base_color = vec3(1.0) # Placeholder for actual color mapping [G]
    
    # Ambient term
    ambient = k_a
    
    # Diffuse and Specular terms scaled by light intensity and coefficients
    # Note: [S2, unofficial] formula includes I_L in diffuse and specular terms
    diffuse_term = I_L * k_d * diffuse_strength
    specular_term = I_L * k_s * specular_strength
    
    final_color = ambient + diffuse_term + specular_term
    
    # Apply to base color if needed (as per [S1, unofficial] scaling approach)
    # final_color = (diffuse_strength + specular_strength) * base_color 
    # This depends on specific implementation choice between [S1, unofficial] and [S2, unofficial]
    
    return final_color
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for shading calculation. On-the-fly normal computation adds 6 texture fetches per sample [S2, unofficial].
*   **Memory Complexity:** $O(1)$ extra memory if normals are computed on-the-fly. $O(N)$ extra memory (4x volume size) if normals are precomputed and stored [S2, unofficial].
*   **Performance:** Described as "quite cheap and simple to compute" [S2, unofficial].
*   **Strengths:** Provides strong depth and shape cues; simple to implement; works well with composite ray functions [S2, unofficial].
*   **Limitations:** Approximates surface shading on volumetric data; does not account for volumetric shadows or subsurface scattering unless extended (e.g., via second raymarch) [S2, unofficial].
*   **When to use:** When visual clarity of internal structures is needed and performance is a concern.
*   **When not to use:** When physically accurate light transport (shadows, scattering) is required.

## Background
The Blinn-Phong model is a modification of the Phong reflection model, introduced by Jim Blinn [S2, unofficial]. In the context of volume rendering, it is applied to scalar fields to simulate isosurface lighting, enhancing the visual interpretation of 3D data [S1, S2, unofficial]. It is commonly used in scientific visualization tools and real-time volume rendering engines.

## Sources
[S1] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
