# Variance Shadow Mapping for NeRFs

Also called: NeRF shadow mapping, VSM for NeRFs

## Overview
Variance Shadow Mapping for NeRFs is a shadow rendering method for neural radiance fields that uses spherical Gaussians (SGs) to render cast shadows, inspired by traditional variance shadow mapping [S1]. It avoids the cumulative transmittance of light along sampling points on shadow rays by comparing depth inconsistencies from light and camera views [S1]. The method transforms discontinuous depth from the light perspective into continuous depth, producing soft shadows with visibility values in $[0, 1]$ rather than the hard shadows produced by naive depth comparison [S1].

## How it works
**Inputs:**
- A trained NeRF model [S1].
- Camera parameters (intrinsics and extrinsics) for the view being rendered [S1].
- Light source position and orientation (defining a virtual camera at the light source) [S1].
- A depth map $D$ rendered from the camera view [S1].
- A box filter kernel size (typically small, e.g., $3\times3$ or $5\times5$) [G].

**Outputs:**
- A visibility factor $V(\mathbf{r}) \in [0, 1]$ for each pixel $\mathbf{r}$ in the camera view [S1].
- The final pixel color $I(\mathbf{r})$ incorporating the shadow [S1].

**Data Structures:**
1. **Depth Map ($X$):** A 2D array of floats, where each element $X(\mathbf{r}_c)$ represents the depth $z$ of the surface point at camera pixel $\mathbf{r}_c$ [S1].
2. **Squared Depth Map ($X^2$):** A 2D array of floats, where each element $X^2(\mathbf{r}_c) = (X(\mathbf{r}_c))^2$ [S1].
3. **Filtered Depth Map ($\mu(X)$):** The result of applying a box filter to $X$ [S1].
4. **Filtered Squared Depth Map ($\mu(X^2)$):** The result of applying a box filter to $X^2$ [S1].
5. **Variance Map ($\sigma^2$):** A 2D array of floats computed from the filtered maps [S1].

**Steps:**
1. **Reconstruct 3D Points:** For each pixel $\mathbf{r}$ in the camera view, use the depth map value $z(\mathbf{r})$ and the camera parameters to reconstruct the 3D coordinate $\mathbf{P}$ of the surface point [S1].
2. **Render Light View Maps:** Set up a virtual camera at the light source position, oriented towards the scene. Render two depth maps from this light view: the depth map $X_L$ (where each pixel $\mathbf{r}_l$ contains the depth $z_L$ of the closest surface point along the ray from the light) and the squared depth map $X^2_L$ (where each pixel $\mathbf{r}_l$ contains $z_L^2$) [S1].
3. **Filter the Light View Maps:** Apply a box filter (uniform average) to both $X_L$ and $X^2_L$ to obtain $\mu(X_L)$ and $\mu(X^2_L)$ [S1].
4. **Calculate Variance:** For each pixel $\mathbf{r}_l$ in the light view, calculate the variance $\sigma^2(\mathbf{r}_l)$ using the filtered maps [S1].
5. **Map Camera Point to Light View:** For the 3D point $\mathbf{P}$ reconstructed in Step 1, project it into the light camera's coordinate system to find the corresponding pixel $\mathbf{r}_l$ in the light view. Retrieve the depth $z(\mathbf{P})$ of this point from the light's perspective, denoted as $z(\mathbf{r})$ in the source notation [S1].
6. **Compute Visibility:** Calculate the visibility $V(\mathbf{r})$ for the camera pixel $\mathbf{r}$ using the Chebyshev inequality-based formula [S1].
7. **Final Rendering:** Combine the visibility with the NeRF's radiance to compute the final pixel color [S1].

**Formulas:**
- Variance: $\sigma^2(\mathbf{r}_l) = \mu(X^2_L)(\mathbf{r}_l) - \mu^2(X_L)(\mathbf{r}_l)$ [S1].
- Visibility: $V(\mathbf{r}) = \begin{cases} 1, & z(\mathbf{r}) \leq \mu(X_L)(\mathbf{r}_l) \\ \frac{\sigma^2(\mathbf{r}_l)}{\sigma^2(\mathbf{r}_l) + (z(\mathbf{r}) - \mu(X_L)(\mathbf{r}_l))^2}, & z(\mathbf{r}) > \mu(X_L)(\mathbf{r}_l) \end{cases}$ [S1].
- Final Color: $I(\mathbf{r}) = I_d(\mathbf{r}) + I_s(\mathbf{r}) + V(\mathbf{r})\mathcal{R}(\mathbf{r})S_{SG}(\mathbf{r})$ [S1].
Where $\mathbf{r}_l$ is the correspondence pixel of $\mathbf{r}$ in the light view, $z(\mathbf{r})$ is the depth of the shaded point as seen from the light, $\mu(X_L)(\mathbf{r}_l)$ is the filtered mean depth at $\mathbf{r}_l$, and $\sigma^2(\mathbf{r}_l)$ is the variance at $\mathbf{r}_l$ [S1]. $I_d$ and $I_s$ are diffuse and specular components, $\mathcal{R}$ is the BRDF, and $S_{SG}$ is the light source intensity [S1].

**Parameters:**
- Box Filter Kernel Size: Not explicitly given in the sources, but typical for VSM is small (e.g., $3\times3$ to $5\times5$) [G]. Larger kernels produce softer shadows but more blur [G].
- Variance Clamp: A small epsilon (e.g., $10^{-6}$) to prevent division by zero or negative variance due to numerical noise [G].

**Edge Cases and Pitfalls:**
- Negative Variance: Due to numerical precision, $\mu(X^2)$ can be slightly less than $\mu(X)^2$. Clamp variance to $\ge 0$ [G].
- Light Frustum Bounds: Points outside the light's view frustum cannot be projected. These should default to fully visible ($V=1$) or be handled based on scene requirements [G].
- Depth Discontinuities: VSM handles soft shadows well, but sharp edges may still exhibit artifacts if the filter size is too small [G].
- NeRF Depth Definition: The "depth" in NeRFs is ambiguous (surface vs. volume). The source implies using a depth map, which suggests a surface extraction or a specific depth metric (e.g., mean depth) was used to generate the initial depth map [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def box_filter(image, kernel_size=3):
    """Apply a uniform box filter to a 2D image."""
    pad = kernel_size // 2
    padded = np.pad(image, pad, mode='edge')
    h, w = image.shape
    kernel = np.ones((kernel_size, kernel_size)) / (kernel_size * kernel_size)
    filtered = np.zeros_like(image)
    for i in range(h):
        for j in range(w):
            filtered[i, j] = np.sum(padded[i:i+kernel_size, j:j+kernel_size] * kernel)
    return filtered

def variance_shadow_mapping_for_nerf(nerf_model, camera_params, light_params, depth_map, kernel_size=3):
    """
    Compute shadow visibility for a NeRF rendering.
    """
    h, w = depth_map.shape
    u_cam, v_cam = np.meshgrid(np.arange(w), np.arange(h))
    
    # Step 1: Reconstruct 3D points from depth map
    x_cam = (u_cam - camera_params['cx']) * depth_map / camera_params['fx']
    y_cam = (v_cam - camera_params['cy']) * depth_map / camera_params['fy']
    z_cam = depth_map
    points_cam = np.stack([x_cam, y_cam, z_cam, np.ones_like(z_cam)], axis=-1)
    cam_to_world = np.linalg.inv(camera_params['extrinsics'])
    points_3d = np.einsum('ij,...j->...i', cam_to_world, points_cam[..., :3])
    
    # Step 2: Render Light View Maps
    # Project 3D points to light camera space to get depth and 2D coordinates
    light_world_to_cam = light_params['world_to_cam']
    points_light_cam = np.einsum('ij,...j->...i', light_world_to_cam, points_3d)
    x_l, y_l, z_l = points_light_cam[..., 0], points_light_cam[..., 1], points_light_cam[..., 2]
    
    # Project to light view pixel coordinates
    u_l = (x_l / z_l) * light_params['fx'] + light_params['cx']
    v_l = (y_l / z_l) * light_params['fy'] + light_params['cy']
    
    # Create light view depth maps (simplified accumulation for pseudocode)
    light_h, light_w = light_params['height'], light_params['width']
    light_depth = np.full((light_h, light_w), np.inf)
    light_depth_sq = np.full((light_h, light_w), np.inf)
    
    # Scatter points to light view grid
    u_l_int = np.clip(u_l.astype(int), 0, light_w - 1)
    v_l_int = np.clip(v_l.astype(int), 0, light_h - 1)
    valid = z_l > 0
    light_depth[v_l_int[valid], u_l_int[valid]] = z_l[valid]
    light_depth_sq[v_l_int[valid], u_l_int[valid]] = z_l[valid] ** 2
    
    # Step 3: Filter Light View Maps
    mu_light_depth = box_filter(light_depth, kernel_size)
    mu_light_depth_sq = box_filter(light_depth_sq, kernel_size)
    
    # Step 4: Calculate Variance
    variance = mu_light_depth_sq - mu_light_depth ** 2
    variance = np.maximum(variance, 1e-6)
    
    # Step 5 & 6: Compute Visibility
    # Retrieve filtered mean depth at the projected light-view coordinates
    mu_at_point = mu_light_depth[v_l_int, u_l_int]
    var_at_point = variance[v_l_int, u_l_int]
    
    diff = z_l - mu_at_point
    visibility = np.ones_like(diff)
    
    occluded_mask = diff > 0
    visibility[occluded_mask] = var_at_point[occluded_mask] / \
                                (var_at_point[occluded_mask] + diff[occluded_mask] ** 2)
    
    # Handle invalid pixels
    visibility[~valid] = 1.0
    
    return visibility
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels, dominated by the projection and filtering steps [G]. The filtering is $O(N \cdot K^2)$ where $K$ is kernel size [G].
- **Memory Complexity:** $O(N)$ to store the depth maps and filtered maps [G].
- **Performance:** Efficient compared to ray-traced shadows or cumulative transmittance methods, as it avoids per-ray shadow ray traversal [S1].
- **Strengths:** Produces soft shadows, efficient, easy to implement with standard graphics primitives [S1].
- **Limitations:** Relies on the accuracy of the depth map from the NeRF. Artifacts may occur if the depth map is noisy or if the NeRF representation is not surface-like [G].

## Background
This method is inspired by Variance Shadow Mapping (VSM) from traditional computer graphics [S1]. It was proposed for NeRFs in the context of relighting scenes with object insertions, aiming to provide realistic shadow effects without the computational overhead of volumetric shadow ray marching [S1].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
