# Sun-aligned Relighting TensoRF

Also called: Fast Sun-aligned Outdoor Scene Relighting based on TensoRF, SR-TensoRF

## Overview
Sun-aligned Relighting TensoRF (SR-TensoRF) is a method for outdoor scene relighting and novel view synthesis that eliminates the need for environment maps by using the sun direction as a direct input for shadow generation [S1]. It builds on [Local illumination, shadows and ambient occlusion](34-local-illumination-shadows-and-ambient-occlusion.md) by decomposing the synthesis process into albedo, shadow, and tone components, leveraging TensoRF's efficiency [S1]. Unlike NeRF-OSR, which relies on environment maps, SR-TensoRF uses sun direction input, simplifying inference and allowing dynamic shadow generation based on time and location [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, sun direction $\mathbf{d}_{sun}$, and a latent vector $\mathbf{l}$ for tone [S1].
*   **Outputs:** Final pixel color $\hat{\mathbf{C}}(\mathbf{r})$ [S1].

**Data Structures:**
SR-TensoRF comprises three tensors [S1]:
1.  **Density Tensor ($G_\sigma$):** Learns density features [S1].
2.  **Appearance Tensor ($G_a$):** A 4D tensor concatenating albedo and shadow features: $G_a(\mathbf{x}) = G_{albedo}(\mathbf{x}) \oplus G_{shadow}(\mathbf{x})$ [S1].
3.  **Cubemap Tensor ($G_c$):** Six 3D tensors representing the faces of a cube, each with 128x128 elements [S1].

**Numbered Steps:**
1.  **Sun Direction Computation:** Compute $\mathbf{d}_{sun}$ from the date, time, and location coordinates of captured images using PyEphem [S1]. Align $\mathbf{d}_{sun}$ (initially in horizontal coordinates) with the world coordinates of the training images by rotating it along the y-axis based on the camera pose convention where the first training image serves as the world coordinate [S1].
2.  **Feature Extraction:** Compute density $\sigma = G_\sigma(\mathbf{x})$ [S1]. Compute albedo color $\mathbf{c} = M_{albedo}(G_{albedo}(\mathbf{x}), \mathbf{l}, \mathbf{d})$ via a shallow MLP [S1]. Compute shadow strength $s = M_{shadow}(G_{shadow}(\mathbf{x}), \mathbf{d}_{sun})$ via a shallow MLP [S1].
3.  **Volumetric Rendering of Albedo and Shadow:** Render albedo $A(\mathbf{r})$ using $\sigma$ and $\mathbf{c}$, and render shadow $S(\mathbf{r}) \in [0, 1]$ using $\sigma$ and $s$ with the standard volume rendering equation [S1].
4.  **Cubemap Background Computation:** Interpolate cubemap feature $G_c(\mathbf{r})$ bilinearly from the six 3D cubemap tensors where the ray intersects [S1]. Compute cubemap background $\mathbf{b}$ by passing $G_c(\mathbf{r})$ and $\mathbf{d}_{sun}$ through a shallow MLP $M_{cubemap}$: $\mathbf{b} = M_{cubemap}(G_c(\mathbf{r}), \mathbf{d}_{sun})$ [S1].
5.  **Final Output:** Compute the final color by combining the volume rendered albedo and shadow with the cubemap background:
    $$ \hat{\mathbf{C}}(\mathbf{r}) = A(\mathbf{r}) \cdot S(\mathbf{r}) + \left(1 - \sum_{i=1}^{N} w_i\right) \mathbf{b} $$
    where $w_i = T_i(1-\exp(-\sigma_i \delta_i))$ and $T_i=\exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].

**Parameters:**
*   **Cubemap tensor size:** 128x128 elements per facet [S1].
*   **Adam optimizer learning rate:** 0.01 for tensor parameters, 0.001 for MLPs [S1].
*   **Cubemap integration delay:** The cubemap is withheld from integration for the initial 5,000 iterations to prevent overfitting [S1].
*   **Loss function:** Mean Squared Error (MSE), L1 loss for density tensor, Total Variation (TV) loss for all tensors, and a regularization loss to constrain shadow values from approaching 1 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_box_intersection(origin, direction, box_min, box_max):
    """Compute intersection of ray with axis-aligned bounding box."""
    t_min = (box_min - origin) / direction
    t_max = (box_max - origin) / direction
    t_enter = max(min(t_min), min(t_max))
    t_exit = min(max(t_min), max(t_max))
    if t_enter > t_exit:
        return None
    return origin + direction * t_enter

def bilinear_interpolate(grid, point):
    """Bilinear interpolation on a 2D grid."""
    x, y = point
    x0, y0 = int(x), int(y)
    x1, y1 = x0 + 1, y0 + 1
    dx, dy = x - x0, y - y0
    g00 = grid[y0, x0]
    g01 = grid[y0, x1]
    g10 = grid[y1, x0]
    g11 = grid[y1, x1]
    top = g00 * (1 - dx) + g01 * dx
    bottom = g10 * (1 - dx) + g11 * dx
    return top * (1 - dy) + bottom * dy

def volume_render_ray(origin, direction, num_samples, latent_vector, sun_direction, 
                      G_sigma, G_albedo, G_shadow, G_cubemap, 
                      M_albedo, M_shadow, M_cubemap):
    """Render a single ray using SR-TensoRF."""
    # Ray marching parameters
    t_near, t_far = 0.0, 1.0
    points = origin + direction * (t_near + (t_far - t_near) * torch.linspace(0, 1, num_samples))
    deltas = torch.diff(torch.cat([torch.tensor([t_near]), torch.linspace(t_near, t_far, num_samples)]))
    
    accumulated_transmittance = 1.0
    accumulated_color = torch.zeros(3)
    
    for i in range(num_samples):
        x = points[i]
        # Tensor lookup (simplified grid interpolation for TensoRF features)
        sigma_feat = G_sigma[x]
        albedo_feat = G_albedo[x]
        shadow_feat = G_shadow[x]
        
        # Compute per-sample properties
        sigma = sigma_feat
        c = M_albedo(albedo_feat, latent_vector, direction)
        s = torch.sigmoid(M_shadow(shadow_feat, sun_direction))
        
        # Volume rendering weights
        alpha = 1.0 - torch.exp(-sigma * deltas[i])
        w_i = accumulated_transmittance * alpha
        accumulated_color += w_i * c
        accumulated_transmittance *= (1.0 - alpha)
        
    # Cubemap background blending
    cubemap_point = ray_box_intersection(origin, direction, torch.tensor([-1.0]), torch.tensor([1.0]))
    if cubemap_point is not None:
        G_c = bilinear_interpolate(G_cubemap, cubemap_point)
        b = M_cubemap(G_c, sun_direction)
        final_color = accumulated_color + (1.0 - accumulated_transmittance) * b
    else:
        final_color = accumulated_color
        
    return final_color
```

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than NeRF-OSR (e.g., 1h 11m vs 32h for Site 1) due to the TensoRF framework [S1].
*   **Memory Complexity:** Uses three tensors ($G_\sigma, G_a, G_c$) and shallow MLPs [S1]. The cubemap tensor size is fixed at 128x128 per facet [S1].
*   **Strengths:** Eliminates environment maps, simplifies inference, allows dynamic shadow generation, and accelerates training and rendering by approximately tenfold compared to NeRF-OSR [S1].
*   **Limitations:** Requires accurate sun direction computation and alignment [S1]. Alpha masks in TensoRF can cause elongated training time, extraneous density learning, artifacts, or cracks in background regions if not handled correctly [S1].
*   **When to use:** Daytime outdoor scene relighting and novel view synthesis where sun direction is known or can be computed [S1].
*   **When not to use:** Indoor scenes or scenes with multiple dominant light sources where sun direction is not the primary illumination source [S1].

## Background
SR-TensoRF was introduced by Jung Yi and Nojun Kwak from Yonsei University in November 2023 [S1]. It is designed for outdoor scene relighting, addressing the challenges of unbounded scenes and varying white balances in training images [S1]. The method builds on TensoRF [8] and integrates concepts from MipNeRF-360 [3] for ray contraction [S1].

## Sources
[S1] Fast Sun-aligned Outdoor Scene Relighting based on TensoRF, https://doi.org/10.1109/wacv57701.2024.00359, fetched 2026-10-26
