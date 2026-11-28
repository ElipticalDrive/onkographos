# Hybrid Radiance Fields (HyRF)

## Overview
Hybrid Radiance Fields (HyRF) is a scene representation method that decomposes a scene into a compact set of explicit 3D Gaussians storing critical high-frequency parameters and grid-based neural fields that predict the remaining properties [S1]. It addresses the memory overhead and limited representational capacity of standard 3D Gaussian Splatting by decoupling geometry and appearance into separate neural fields, enabling a 20× reduction in model size while maintaining real-time rendering speeds and state-of-the-art quality [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of calibrated images with known camera poses; initial sparse point cloud or structure-from-motion data.
*   **Outputs:** A rendered image for a target camera pose; optimized scene representation consisting of explicit Gaussian parameters and neural field weights.

**Data Structures**
1.  **Explicit Gaussians:** A list of $N$ Gaussians. Each Gaussian stores exactly 8 parameters:
    *   Position $p_e \in \mathbb{R}^3$
    *   Diffuse color $c_e \in \mathbb{R}^3$
    *   Isotropic scale $s_e \in \mathbb{R}$
    *   Opacity $\alpha_e \in \mathbb{R}$
    *   *Note:* Rotation is not stored explicitly in the compact set but is predicted by the neural field [S1].
2.  **Neural Fields:** Two separate neural networks, $\Theta_{geo}$ and $\Theta_{rad}$, utilizing multi-resolution hash encoding [S1].
    *   $\Theta_{geo}$: Predicts geometry properties (scale, opacity, rotation).
    *   $\Theta_{rad}$: Predicts appearance properties (view-dependent color).
    *   *Encoding:* Multi-resolution hash encoding is used for coordinate compression [S1].

**Coordinate Preprocessing**
Before querying the neural fields, 3D points are normalized and contracted to the range $(0, 1)$ to handle unbounded scenes [S1].
1.  Define the axis-aligned bounding box (AABB) $B_0$ as the minimum and maximum camera positions of the training set.
2.  Normalize the point $p_i$ using $B_0$.
3.  Contract the normalized point $p_i$ to $p'_i$ using the following formula [S1]:

$$
\text{contract}(p_i) = 
\begin{cases} 
0.25 \cdot p_i + 1 & \text{if } \|p_i\| \le 1 \\
\frac{p_i}{0.25 \cdot (2 - \|p_i\|) + 1} & \text{otherwise}
\end{cases}
$$

**Hybrid Rendering Pipeline**
The rendering process operates in three stages [S1]:

1.  **Visibility Pre-culling:** Eliminate explicit Gaussians that lie outside the current view frustum. This reduces the computational overhead of querying the neural fields for irrelevant regions [S1].
2.  **Neural Property Prediction & Combination:** For the remaining visible Gaussians, query their positions through the neural fields to predict the missing properties. These predicted properties are combined with the stored explicit parameters to recover high-frequency details [S1].
    *   Query $\Theta_{geo}$ with the contracted position to predict scale, opacity, and rotation.
    *   Query $\Theta_{rad}$ with the contracted position and view direction to predict view-dependent color.
    *   Combine predicted geometry with explicit geometry, and predicted color with explicit diffuse color.
3.  **Background Compositing:** The neural field generates a background map projected onto a background sphere. This background map is composited with the foreground Gaussian rendering through alpha blending to address insufficient background modeling for distant objects [S1].

**Alpha Blending**
The final pixel color $C$ is computed using standard point-based alpha blending [S1]:

$$
C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

Where:
*   $C$ is the final predicted pixel color.
*   $N$ is the set of sorted Gaussians projected onto the pixel.
*   $c_i$ is the color of the $i$-th Gaussian.
*   $\alpha_i$ is the opacity of the $i$-th Gaussian.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hyrf(camera, explicit_gaussians, theta_geo, theta_rad, hash_table):
    """
    Renders a frame using Hybrid Radiance Fields.
    
    Args:
        camera: Camera object with projection matrix and view direction.
        explicit_gaussians: List of Gaussian objects with fields:
            - pos: (3,) float32
            - color_diffuse: (3,) float32
            - scale: (1,) float32
            - opacity: (1,) float32
        theta_geo: Neural field for geometry (scale, opacity, rotation).
        theta_rad: Neural field for appearance (view-dependent color).
        hash_table: Multi-resolution hash encoding table.
        
    Returns:
        rendered_image: (H, W, 3) float32 array.
    """
    
    # Stage 1: Visibility Pre-culling
    # Filter Gaussians outside the view frustum
    visible_gaussians = []
    for g in explicit_gaussians:
        if is_in_frustum(g.pos, camera.projection_matrix):
            visible_gaussians.append(g)
            
    # Stage 2: Neural Property Prediction & Combination
    processed_gaussians = []
    for g in visible_gaussians:
        # Normalize and contract position
        p_norm = normalize_with_aabb(g.pos, camera.aabb)
        p_contracted = contract(p_norm)
        
        # Query Geometry Field
        # theta_geo returns: predicted_scale, predicted_opacity, predicted_rotation
        pred_geo = theta_geo.query(p_contracted, hash_table)
        
        # Combine explicit and predicted geometry
        # Explicit stores scale and opacity, but rotation is predicted
        # Note: Source states explicit stores scale/opacity, neural predicts scale/opacity/rotation
        # We combine them. Typically, explicit provides the base, neural refines.
        # For implementation, we assume the neural field predicts the *residual* or 
        # the full property which overrides/adjusts the explicit one. 
        # Based on "combined with stored explicit parameters to recover high-frequency details",
        # we treat explicit as the base and neural as the detail.
        
        final_scale = combine_parameters(g.scale, pred_geo.scale)
        final_opacity = combine_parameters(g.opacity, pred_geo.opacity)
        final_rotation = pred_geo.rotation # Rotation is not in explicit, so use predicted
        
        # Query Appearance Field
        # theta_rad returns: view-dependent color
        view_dir = get_view_direction(g.pos, camera.position)
        pred_color = theta_rad.query(p_contracted, view_dir, hash_table)
        
        # Combine explicit diffuse color with predicted view-dependent color
        # Standard 3DGS SH expansion is implicit in the view-dependent query
        final_color = combine_color(g.color_diffuse, pred_color)
        
        processed_gaussians.append({
            'pos': g.pos,
            'color': final_color,
            'scale': final_scale,
            'opacity': final_opacity,
            'rotation': final_rotation
        })
        
    # Stage 3: Background Compositing
    # Render foreground Gaussians
    foreground_image = alpha_blend_gaussians(processed_gaussians, camera)
    
    # Generate background map from neural field
    # Project background sphere points through neural field
    background_image = render_background_sphere(theta_rad, hash_table, camera)
    
    # Composite foreground and background
    # Assuming background is rendered with its own alpha or as a solid backdrop
    # If background is a map, it might be blended or simply added if alpha is 0
    # The source says "composited ... through alpha blending"
    final_image = alpha_blend(foreground_image, background_image)
    
    return final_image

def contract(p):
    """
    Contracts normalized point p to range (0, 1).
    """
    norm = np.linalg.norm(p)
    if norm <= 1.0:
        return 0.25 * p + 1.0
    else:
        return p / (0.25 * (2.0 - norm) + 1.0)
```

**Parameters**
*   **Neural Field Encoding:** Multi-resolution hash encoding [S1].
*   **Scene Bounding Box:** Defined by the minimum and maximum camera positions of the training set [S1].

**Edge Cases and Pitfalls**
*   **Weak Correlation:** Predicting all Gaussian properties through a single neural field fails to achieve satisfactory performance due to the weak correlation between geometry and appearance attributes [S1]. This necessitates the decoupled architecture.
*   **Distant Background:** Standard Gaussian representations have insufficient background modeling for distant objects. The background sphere compositing addresses this limitation [S1].
*   **Frustum Culling:** Failing to pre-cull Gaussians leads to significant computational overhead when querying neural fields for invisible regions [S1].

## Operating characteristics
*   **Memory Efficiency:** Reduces model size by over 20× compared to standard 3D Gaussian Splatting (3DGS) [S1].
*   **Performance:** Maintains real-time rendering speeds [S1].
*   **Quality:** Achieves state-of-the-art rendering quality [S1].
*   **Strengths:** High-frequency detail preservation via explicit Gaussians; efficient memory usage via neural fields; robust background modeling.
*   **Limitations:** Requires training two separate neural fields; relies on multi-resolution hash encoding which may have memory overhead depending on resolution settings (not specified in sources).

## Background
Hybrid Radiance Fields (HyRF) was introduced to address the limitations of explicit Gaussian representations in terms of memory overhead and representational capacity for fine details [S1]. It builds on the principles of 3D Gaussian Splatting and integrates them with neural radiance fields (NeRF) concepts, specifically multi-resolution hash encoding [S1]. The method was developed to provide a memory-efficient and high-quality novel view synthesis solution [S1].

## Sources
[S1] HyRF: Hybrid Radiance Fields for Memory-efficient and High-quality Novel View Synthesis, https://doi.org/10.52202/085713-0617, fetched 2026-10-30
