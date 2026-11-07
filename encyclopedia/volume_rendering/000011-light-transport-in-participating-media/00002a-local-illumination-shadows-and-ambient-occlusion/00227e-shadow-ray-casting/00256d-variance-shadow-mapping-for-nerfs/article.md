# Variance Shadow Mapping for NeRFs

Also called: NeRF shadow mapping, VSM for NeRFs

## Overview
Variance Shadow Mapping for NeRFs is a technique for rendering cast shadows in Neural Radiance Fields that avoids the computational cost of cumulative transmittance along shadow rays [S1]. By leveraging depth maps rendered from the light's perspective and applying variance shadow mapping principles, it computes a continuous visibility factor in the range $[0, 1]$ [S1]. Unlike naive shadow mapping which produces hard edges, this method generates soft shadows by transforming discontinuous depth comparisons into continuous probability estimates [S1].

## How it works
The method computes a visibility factor $V(\mathbf{r})$ for a view ray $\mathbf{r}$ by comparing the depth of the surface point along $\mathbf{r}$ against a statistical distribution of depths observed from the light source [S1].

**Inputs:**
*   NeRF model $\mathcal{R}(\mathbf{r})$ and scattering coefficient $S_{SG}(\mathbf{r})$ (as defined in the parent article [Shadow Ray Casting](2796-shadow-ray-casting.md)).
*   Camera parameters (intrinsic and extrinsic) for the view ray $\mathbf{r}$.
*   Light source position (defining the virtual camera view).
*   Depth map $X$ and squared depth map $X^2$ rendered from the light view.
*   Filtered depth map $\mu(X)$ and filtered squared depth map $\mu(X^2)$ from the light view.

**Outputs:**
*   Visibility factor $V(\mathbf{r}) \in [0, 1]$.
*   Final radiance $I(\mathbf{r})$.

**Data Structures:**
*   **Depth Map ($X$):** A 2D array where each element stores the distance $z$ from the light source to the nearest surface intersection along the ray corresponding to that pixel.
*   **Squared Depth Map ($X^2$):** A 2D array where each element stores $z^2$.
*   **Filtered Maps ($\mu(X), \mu(X^2)$):** The result of applying a box filter (mean filter) to $X$ and $X^2$ respectively. The filter size determines the softness of the shadow penumbra.

**Steps:**

1.  **Reconstruct 3D Position:**
    Using the depth map from the *camera* view and the camera parameters, reconstruct the 3D position $\mathbf{p}$ of the surface point intersected by the view ray $\mathbf{r}$. Let $z(\mathbf{r})$ be the distance from the light source to this point $\mathbf{p}$.
    *Note: The source states "coordinates of surface points in 3D space can be obtained" according to the depth map and camera parameters [S1].*

2.  **Map to Light View:**
    Project the 3D point $\mathbf{p}$ onto the 2D plane of the virtual camera located at the light source. Identify the corresponding pixel coordinate $\mathbf{r}_l$ in the light view's depth maps.

3.  **Retrieve Statistical Moments:**
    Retrieve the filtered mean depth $\mu(\mathbf{X})(\mathbf{r}_l)$ and the filtered mean squared depth $\mu(\mathbf{X}^2)(\mathbf{r}_l)$ from the pre-computed maps at pixel $\mathbf{r}_l$.

4.  **Calculate Variance:**
    Compute the variance $\sigma^2(\mathbf{r}_l)$ of the depth distribution at $\mathbf{r}_l$ using the second moment and the square of the first moment:
    $$
    \sigma^2(\mathbf{r}_l) = \mu(\mathbf{X}^2)(\mathbf{r}_l) - \mu^2(\mathbf{X})(\mathbf{r}_l)
    $$
    [S1]

5.  **Compute Visibility:**
    Determine the visibility $V(\mathbf{r})$ based on the comparison between the point's depth $z(\mathbf{r})$ and the filtered mean depth $\mu(\mathbf{X})(\mathbf{r}_l)$:
    $$
    V(\mathbf{r}) = \begin{cases} 
    1, & \text{if } z(\mathbf{r}) \leq \mu(\mathbf{X})(\mathbf{r}_l) \\
    \frac{\sigma^2(\mathbf{r}_l)}{\sigma^2(\mathbf{r}_l) + (z(\mathbf{r}) - \mu(\mathbf{X})(\mathbf{r}_l))^2}, & \text{if } z(\mathbf{r}) > \mu(\mathbf{X})(\mathbf{r}_l)
    \end{cases}
    $$
    [S1]
    *Note: This formula is derived from Chebyshev's inequality, providing an upper bound on the probability that the point is occluded.*

6.  **Final Rendering:**
    Combine the visibility factor with the direct illumination and scattering terms:
    $$
    I(\mathbf{r}) = I_d(\mathbf{r}) + I_s(\mathbf{r}) + V(\mathbf{r})\mathcal{R}(\mathbf{r})S_{SG}(\mathbf{r})
    $$
    [S1]
    Where $I_d(\mathbf{r})$ and $I_s(\mathbf{r})$ are the direct and scattered illumination components defined in the parent article.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_neRF_VSM(z_point, mu_X, mu_X2, sigma_sq):
    """
    Computes visibility V(r) for a point at depth z_point given light-view statistics.
    
    Args:
        z_point (float): Depth of the surface point from the light source.
        mu_X (float): Filtered mean depth at the corresponding light-view pixel.
        mu_X2 (float): Filtered mean squared depth at the corresponding light-view pixel.
        sigma_sq (float): Pre-computed variance sigma^2(r_l). 
                          If not provided, it is calculated as mu_X2 - mu_X^2.
    
    Returns:
        float: Visibility factor V(r) in [0, 1].
    """
    # Calculate variance if not provided
    if sigma_sq is None:
        sigma_sq = mu_X2 - (mu_X ** 2)
    
    # Guard against negative variance due to numerical precision
    if sigma_sq < 0:
        sigma_sq = 0.0
        
    # Case 1: Point is in front of the mean depth (not occluded by the "average" surface)
    if z_point <= mu_X:
        return 1.0
    
    # Case 2: Point is behind the mean depth (potentially occluded)
    # Use Chebyshev-based visibility estimate
    numerator = sigma_sq
    denominator = sigma_sq + (z_point - mu_X) ** 2
    
    # Avoid division by zero if both variance and depth difference are zero
    if denominator == 0:
        return 1.0
        
    return numerator / denominator

def render_neRF_with_VSM(ray, nerf_model, light_view_depth_map, light_view_depth_sq_map, 
                         light_view_mu_depth, light_view_mu_depth_sq, camera_params, light_pos):
    """
    Renders a single ray using Variance Shadow Mapping for NeRFs.
    
    Args:
        ray: The view ray object.
        nerf_model: The NeRF model providing radiance R(r) and scattering S_SG(r).
        light_view_depth_map: 2D array of raw depth values from light view.
        light_view_depth_sq_map: 2D array of squared depth values from light view.
        light_view_mu_depth: 2D array of filtered mean depth from light view.
        light_view_mu_depth_sq: 2D array of filtered mean squared depth from light view.
        camera_params: Intrinsics and extrinsics for the current camera.
        light_pos: 3D position of the light source.
        
    Returns:
        float: Final radiance I(r).
    """
    # 1. Reconstruct 3D position from camera view depth
    # Assuming standard depth map decoding: p = camera_pos + ray_direction * depth_camera
    # This step requires the depth map from the CAMERA view, not the light view.
    # The source implies this reconstruction is done first.
    depth_camera = get_depth_from_camera_view(ray, camera_params) # [G] Standard depth decoding
    p_3d = ray.origin + ray.direction * depth_camera
    
    # 2. Calculate depth from light source
    z_point = distance(p_3d, light_pos)
    
    # 3. Project p_3d to light view to find pixel coordinate r_l
    r_l = project_to_light_view(p_3d, light_pos, camera_params) # [G] Standard projection
    
    # 4. Retrieve light-view statistics
    mu_X = light_view_mu_depth[r_l.y][r_l.x]
    mu_X2 = light_view_mu_depth_sq[r_l.y][r_l.x]
    
    # 5. Compute visibility
    V = compute_visibility_neRF_VSM(z_point, mu_X, mu_X2, None)
    
    # 6. Get NeRF properties
    R_r = nerf_model(ray) # Radiance
    S_SG_r = nerf_model_scattering(ray) # Scattering coefficient
    
    # 7. Combine with direct and scattered illumination (from parent article)
    I_d = get_direct_illumination(ray) # [G] Defined in parent
    I_s = get_scattered_illumination(ray) # [G] Defined in parent
    
    I_final = I_d + I_s + V * R_r * S_SG_r
    
    return I_final
```

**Parameters:**
*   **Box Filter Size:** The size of the box filter applied to $X$ and $X^2$ controls the softness of the shadow. Larger filters result in softer shadows but may introduce blur artifacts. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Negative Variance:** Due to floating-point precision errors, $\mu(X^2) - \mu^2(X)$ may become slightly negative. It must be clamped to 0 [G].
*   **Depth Discontinuities:** The method relies on the assumption that depth varies smoothly within the filter window. Sharp depth discontinuities (object edges) may cause incorrect variance estimates, leading to shadow artifacts.
*   **Occlusion Ambiguity:** The visibility formula is an upper bound based on Chebyshev's inequality. It may overestimate visibility in complex occlusion scenarios where the depth distribution is multi-modal [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for the visibility calculation, assuming pre-computed filtered depth maps. The dominant cost is the NeRF evaluation and the initial rendering of depth maps from the light view.
*   **Memory Complexity:** Requires storing two additional depth maps (depth and squared depth) and their filtered versions for the light view. This is significantly less memory than storing full transmittance volumes or performing multiple shadow ray samples.
*   **Performance:** The method is described as "efficient in practice" because it avoids the cumulative transmittance calculation along shadow rays [S1].
*   **Strengths:** Produces soft shadows naturally; computationally efficient compared to ray-traced shadows; compatible with standard NeRF rendering pipelines.
*   **Limitations:** Relies on accurate depth estimation from the light view; may produce artifacts at depth discontinuities; visibility is an approximation (upper bound).

## Background
This method was proposed in the paper "Relighting Scenes With Object Insertions in Neural Radiance Fields" [S1]. It is inspired by Variance Shadow Mapping (VSM) from traditional computer graphics [S1]. The key insight is to transform the discontinuous depth comparison of traditional shadow mapping into a continuous depth probability distribution, allowing for soft shadow rendering in NeRFs without explicit shadow ray casting.

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
