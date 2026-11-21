# Phong Shading with Boundary Enhancement and Facing Subtraction

Also called: Boundary-enhanced volume shading, Gradient-based Phong shading

## Overview
This method extends per-fragment Phong lighting by applying two alpha-modulation techniques to enhance the visual perception of volume boundaries: boundary enhancement via absorption approximation and facing subtraction for silhouette definition. It distinguishes itself from standard gradient-based Phong shading by explicitly modifying the fragment alpha based on the gradient magnitude and the view direction relative to the surface normal.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (stored as a 3D texture `tex3D`) containing precomputed gradient components ($g_x, g_y, g_z$) and magnitude ($||g||$). A 1D transfer function texture (`tex1D`) maps scalar values to color and alpha.
*   **Output:** A fragment color and alpha value.

**Data Structures:**
The source code references specific uniform structures for lighting and rendering parameters [S1, unofficial]:
*   `light_t`: Contains coefficients `kd` (diffuse), `ks` (specular), `ka` (ambient).
*   `rendering_t` (or `myRender`): Contains integer flags `boundary` and `facing`. If `>0`, the respective effect is enabled.

**Numbered Steps:**
1.  **Gradient Extraction:** Sample the 3D texture at the current fragment's position to retrieve the gradient vector components $(g_x, g_y, g_z)$ and the precomputed magnitude $||g||$.
2.  **Normal Computation:** Normalize the gradient vector to obtain the surface normal $\mathbf{n}$:
    $$ \mathbf{n} = \frac{(g_x, g_y, g_z)}{||(g_x, g_y, g_z)||} $$
3.  **Coordinate Transformation:** Transform the normal $\mathbf{n}$ into view space using the model-view matrix. The vertex shader computes the eye vector $\mathbf{E}$ and light vector $\mathbf{L}$ in view space and passes them as varyings [S1, unofficial].
4.  **Lighting Calculation:**
    *   Compute the diffuse term $D$ using the dot product of the normal $\mathbf{n}$ and the light direction $\mathbf{L}$:
        $$ D = \max(\mathbf{n} \cdot \mathbf{L}, 0) $$
    *   Compute the reflection vector $\mathbf{r}$:
        $$ \mathbf{r} = \text{reflect}(-\mathbf{L}, \mathbf{n}) $$
    *   Compute the specular term $S$ using a specular exponent of 512:
        $$ S = \max(\mathbf{r} \cdot \mathbf{n}, 0)^{512} $$
    *   Compute the ambient term $A$ by sampling the 1D transfer function at the scalar value:
        $$ A = \text{TF}(t) $$
    *   Combine terms to get the initial color contribution:
        $$ C = k_s S + k_d D + k_a A $$
5.  **Boundary Enhancement:** If the `boundary` flag is enabled, modify the fragment alpha $\alpha$ using an absorption approximation based on the gradient magnitude:
    $$ \alpha' = \alpha (1 - e^{-||g||}) $$
    This step enhances the visibility of boundaries where the gradient magnitude is high [S1, unofficial].
6.  **Facing Subtraction:** If the `facing` flag is enabled, further modify the alpha by subtracting the facing component relative to the view vector $\mathbf{V}$ (or eye vector $\mathbf{E}$). The source code uses `ndotv`, defined as $\mathbf{n} \cdot \mathbf{v}$:
    $$ \alpha'' = \alpha' (1 - |\mathbf{n} \cdot \mathbf{v}|) $$
    This reduces alpha for surfaces facing the viewer, enhancing silhouette definition [S1, unofficial]. Note: The formula in the notes uses absolute value $|\mathbf{n} \cdot \mathbf{v}|$, while the code snippet uses `1-ndotv`. Given the context of "facing subtraction" for silhouettes, the absolute value ensures symmetry for both front and back faces, or it assumes a specific winding. The code snippet `1-ndotv` implies a non-absolute dot product, which would only affect front-facing fragments if $\mathbf{n} \cdot \mathbf{v} > 0$. However, the formula explicitly states $|\mathbf{n} \cdot \mathbf{v}|$. We follow the formula's explicit definition for the general case, but note the code implementation may vary [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader (simplified from GLSL 120)
def vertex_shader(vertex_pos, model_view_matrix):
    # Compute eye vector in view space
    vertex_view_pos = model_view_matrix * vertex_pos
    eye_vector = normalize(-vertex_view_pos.xyz)
    
    # Light direction is fixed in this implementation (0,0,-1)
    light_vector = vec3(0, 0, -1)
    
    return eye_vector, light_vector

# Fragment Shader
def fragment_shader(gradient_tex, tf_tex, scalar_value, normal, eye_vector, light_vector, boundary_flag, facing_flag):
    # 1. Lighting Terms
    # Diffuse
    diffuse_term = max(dot(normal, light_vector), 0.0)
    
    # Specular
    reflection = reflect(-light_vector, normal)
    specular_term = pow(max(dot(reflection, normal), 0.0), 512.0)
    
    # Ambient / Transfer Function
    tf_sample = texture1D(tf_tex, scalar_value)
    ambient_term = tf_sample
    
    # Initial Color and Alpha
    # Assuming uniform coefficients kd, ks, ka are passed
    color = ks * specular_term + kd * diffuse_term + ka * ambient_term
    alpha = tf_sample.a
    
    # 2. Boundary Enhancement
    # Gradient magnitude is assumed to be passed or computed from gradient vector
    # In the source, magnitude is extracted from the 3D texture
    gradient_magnitude = ... # Extracted from tex3D
    
    if boundary_flag > 0:
        alpha = alpha * (1.0 - exp(-gradient_magnitude))
        
    # 3. Facing Subtraction
    # ndotv is n . v
    ndotv = dot(normal, eye_vector)
    
    if facing_flag > 0:
        # Formula specifies absolute value, code snippet does not. 
        # Using absolute value as per formula description for silhouette enhancement.
        alpha = alpha * (1.0 - abs(ndotv))
        
    return color, alpha
```

**Parameters:**
*   `kd`, `ks`, `ka`: Diffuse, specular, and ambient lighting coefficients [S1, unofficial].
*   `boundary`: Integer flag. If `>0`, boundary enhancement is enabled [S1, unofficial].
*   `facing`: Integer flag. If `>0`, facing subtraction is enabled [S1, unofficial].
*   Specular exponent: Fixed at 512.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If the gradient magnitude is zero, the normal is undefined. The source does not specify a fallback, but numerically, normalization of a zero vector results in NaN/Inf. A guard against zero-length gradients is required [G].
*   **Facing Subtraction Sign:** The discrepancy between the formula ($|\mathbf{n} \cdot \mathbf{v}|$) and the code snippet (`1-ndotv`) must be resolved. The formula suggests handling both front and back faces, while the code only affects front-facing fragments (where $\mathbf{n} \cdot \mathbf{v} > 0$). Implementers should choose based on whether back-face silhouettes are desired [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment, assuming constant-time texture lookups and arithmetic operations.
*   **Memory Complexity:** $O(1)$ additional memory per fragment, plus the storage for the 3D gradient texture and 1D transfer function texture.
*   **Strengths:** Enhances the perception of internal structures and boundaries in volume data without requiring explicit surface extraction. The high specular exponent (512) creates sharp highlights, emphasizing surface-like qualities.
*   **Limitations:** Relies on precomputed gradients, increasing memory bandwidth and storage requirements. The facing subtraction can make volumes appear hollow or less dense if over-applied. The fixed specular exponent may not be suitable for all material appearances.
*   **When to use:** When visualizing complex internal structures where boundary definition is critical, and explicit surface extraction is too expensive or unnecessary.
*   **When not to use:** When real-time performance is constrained by memory bandwidth, or when a more physically accurate lighting model is required.

## Background
This method originates from a GitHub repository for scientific visualization, specifically focusing on slicing and volume rendering techniques. It is an implementation detail found in educational or prototype code rather than a peer-reviewed algorithm. The technique combines standard Phong shading with specific alpha modulation strategies to improve the visual clarity of volume data.

## Sources
[S1] extrovert7986/Scientific-visualization-Slicing-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Slicing-volume-rendering, fetched 2026-10-26 (unofficial)
