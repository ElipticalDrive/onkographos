# Custom Shadow Mapping for Satellite Imagery

Also called: Geometrically consistent shadows, Sun camera shadow mapping

## Overview
This method casts geometrically consistent shadows in 3D Gaussian Splatting (3DGS) by comparing elevation renders from a satellite camera and a "sun camera," avoiding ray marching which is incompatible with the splatting primitive [S1]. It differs from NeRF-based shadowing (e.g., EO-NeRF) by relying solely on rasterization and elevation comparison rather than volumetric ray marching [S1].

## How it works
**Inputs:**
*   3D Gaussian primitives: means $\mu_k$, covariances $\Sigma_k$, colors $f_k$, opacities $\alpha_k$.
*   Satellite camera projection matrix $A$ (affine model).
*   Sun camera projection matrix $S$ (affine model, aligned with directional light).
*   Elevation function $E: \mathbb{R}^3 \to \mathbb{R}$ mapping world coordinates to altitude in meters.
*   Density parameter $\rho$ for shadow attenuation.
*   Per-camera ambient light coefficient $\psi^A$.
*   Camera-specific affine color correction $\phi^A$.

**Outputs:**
*   Final image intensity $I_{A,S}(u)$ for each pixel $u$.

**Data Structures:**
*   **Elevation Render:** A 2D texture or buffer storing $E_A(u)$, computed by rendering the scene using altitude values instead of colors.
*   **Sun Elevation Render:** A 2D texture or buffer storing $E_S(\tilde{u})$, computed by rendering the scene from the sun camera's perspective.

**Steps:**

1.  **Construct Sun Camera:** Define an affine camera $S$ aligned with the sun's directional light vector. Since the sun is distant, it is treated as a directional light source [S1].
2.  **Render Elevation Maps:**
    *   Render the satellite camera view to produce elevation map $E_A(u)$ using the formula:
        $$ E_A(u) = \sum_{k=1}^K [E_{\mu_k}] \omega^A_k(u) $$
        where $[E_{\mu_k}]$ is the altitude at the Gaussian mean $\mu_k$, and $\omega^A_k(u)$ is the splat weight [S1].
    *   Render the sun camera view to produce elevation map $E_S(\tilde{u})$ using the same logic [S1].
3.  **Localize Pixel to 3D:** For each pixel $u$ in the satellite image, determine its 3D world position $x$ using the localization function $loc_A(u, E_A(u))$. This function finds $x$ such that $A(x) = u$ and $E(x) = E_A(u)$ [S1].
4.  **Project to Sun Camera:** Project the localized 3D point $x$ into the sun camera's coordinate system to find the homologous point $\tilde{u}$:
    $$ \tilde{u} = S x $$
    This is equivalent to the homologous point function $hom_{A,S}(u)$ [S1].
5.  **Resample Sun Elevation:** Sample the sun elevation map $E_S$ at the projected point $\tilde{u}$ to get $E_S(\tilde{u})$ [S1].
6.  **Compute Elevation Difference:** Calculate the height difference $\Delta h_{A,S}(u)$:
    $$ \Delta h_{A,S}(u) = E_S(hom_{A,S}(u)) - E_A(u) $$
    If $\Delta h_{A,S}(u) \approx 0$, the point is in light (both cameras see the same surface). If $\Delta h_{A,S}(u) > 0$, the point is in shadow (the sun camera sees a higher surface blocking the view) [S1].
7.  **Compute Shadow Coefficient:** Calculate the darkening coefficient $s_{A,S}(u)$:
    $$ s_{A,S}(u) = \min(\exp(-\rho \Delta h_{A,S}(u)), 1) $$
    This models attenuation through a homogeneous medium of density $\rho$ [S1].
8.  **Compute Lighting Coefficient:** Combine shadow and ambient light:
    $$ l_{A,S}(u) = s_{A,S}(u) + (1 - s_{A,S}(u))\psi^A $$
    where $\psi^A$ prevents shadows from becoming completely black [S1].
9.  **Render Final Image:** Compute the final intensity:
    $$ I_{A,S}(u) = l_{A,S}(u) \sum_{k=1}^K \phi^A(f_k)\omega^A_k(u) $$
    where $\phi^A$ is a camera-specific affine color correction applied to intrinsic primitive colors $f_k$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shadow_mapping(satellite_camera, sun_camera, gaussians, elevation_func, rho, ambient_light, color_correction):
    """
    satellite_camera: Camera object with projection matrix A
    sun_camera: Camera object with projection matrix S
    gaussians: List of Gaussian primitives with means, covariances, colors
    elevation_func: Function E mapping 3D point to altitude
    rho: Shadow density parameter
    ambient_light: Per-camera ambient coefficient psi^A
    color_correction: Camera-specific affine color correction phi^A
    """
    
    # Step 2: Render Elevation Maps
    # Render satellite elevation map
    sat_elevation_map = render_elevation(satellite_camera, gaussians, elevation_func)
    
    # Render sun elevation map
    sun_elevation_map = render_elevation(sun_camera, gaussians, elevation_func)
    
    # Initialize output image
    output_image = zeros_like(sat_elevation_map)
    
    # Step 3-9: Process each pixel
    for u in pixels(sat_elevation_map):
        # Get altitude at pixel u from satellite view
        h_sat = sat_elevation_map[u]
        
        # Step 3: Localize pixel to 3D point x
        # Find x such that A(x) = u and E(x) = h_sat
        # Note: In practice, this requires inverting the projection or using depth/elevation lookup
        x = localize_3d_point(satellite_camera, u, h_sat, elevation_func)
        
        # Step 4: Project to sun camera
        u_sun = project_to_camera(sun_camera, x)
        
        # Step 5: Resample sun elevation
        # Handle boundary checks for u_sun
        if is_valid_pixel(u_sun, sun_elevation_map):
            h_sun = sample_elevation(sun_elevation_map, u_sun)
        else:
            # If point is outside sun camera view, assume it's in shadow or handle as needed
            h_sun = infinity 
            
        # Step 6: Compute elevation difference
        delta_h = h_sun - h_sat
        
        # Step 7: Compute shadow coefficient
        # If delta_h <= 0, point is in light (s=1). If delta_h > 0, exponential decay.
        if delta_h <= 0:
            s = 1.0
        else:
            s = min(exp(-rho * delta_h), 1.0)
            
        # Step 8: Compute lighting coefficient
        l = s + (1 - s) * ambient_light
        
        # Step 9: Render final image
        # Compute standard 3DGS splatting color for pixel u
        base_color = 0.0
        for k in gaussians:
            weight = compute_splat_weight(satellite_camera, k, u)
            corrected_color = apply_color_correction(color_correction, k.color)
            base_color += corrected_color * weight
            
        output_image[u] = l * base_color
        
    return output_image

def render_elevation(camera, gaussians, elevation_func):
    """
    Renders the scene from the given camera, but stores elevation values instead of colors.
    """
    elevation_map = zeros_like(camera.image_shape)
    for k in gaussians:
        # Project Gaussian mean to 2D
        mu_2d = project_to_2d(camera, k.mean)
        # Compute splat weight
        weight = compute_splat_weight(camera, k, mu_2d) # Simplified for illustration
        # Add elevation contribution
        elevation = elevation_func(k.mean)
        # Accumulate (simplified blending)
        elevation_map += elevation * weight
    return elevation_map

def localize_3d_point(camera, u, h, elevation_func):
    """
    Finds 3D point x such that camera(x) = u and elevation_func(x) = h.
    This is a geometric inverse problem.
    """
    # In an affine camera model, this can be solved analytically.
    # For a general perspective camera, this might require ray casting or lookup.
    # Given the source specifies affine camera model for sun and satellite,
    # we assume an affine inverse projection exists.
    
    # Get ray direction from camera center through u
    ray_origin = camera.center
    ray_direction = unproject_pixel(camera, u)
    
    # Solve for t such that elevation_func(ray_origin + t * ray_direction) = h
    # This is a root-finding problem or linear solve if elevation is linear.
    # For affine elevation E(x) = a*x + b*y + c*z + d:
    # E(ray_origin + t * ray_direction) = h
    # a*(ox + t*dx) + b*(oy + t*dy) + c*(oz + t*dz) + d = h
    # t * (a*dx + b*dy + c*dz) = h - (a*ox + b*oy + c*oz + d)
    
    denom = dot(elevation_func.gradient, ray_direction)
    if abs(denom) < epsilon:
        return None # Ray parallel to elevation plane
        
    t = (h - elevation_func(ray_origin)) / denom
    x = ray_origin + t * ray_direction
    return x
```

**Parameters:**
*   $\rho$: Density of the homogeneous medium. Typical value not given in sources.
*   $\psi^A$: Per-camera ambient light coefficient. Typical value not given in sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The elevation difference $\Delta h$ must be computed with sufficient precision to distinguish between light and shadow, especially for small height differences.
*   **Boundaries:** If the projected point $\tilde{u}$ is outside the sun camera's image bounds, the elevation $E_S(\tilde{u})$ is undefined. The pseudocode handles this by assuming infinity (shadow), but implementation may vary.
*   **Degenerate Input:** If the ray from the satellite camera is parallel to the elevation plane, localization fails.

## Operating characteristics
*   **Time Complexity:** Requires two full rasterization passes (satellite and sun elevation maps) plus per-pixel shadow computation. Complexity is $O(N + P)$ where $N$ is number of Gaussians and $P$ is number of pixels.
*   **Memory Complexity:** Requires storing two elevation maps of size $W \times H$.
*   **Strengths:** Geometrically consistent shadows; compatible with Gaussian Splatting (no ray marching); efficient compared to volumetric shadow casting.
*   **Limitations:** Assumes directional light source; shadows are based on elevation only, not occlusion by non-elevation features (e.g., overhangs might not cast shadows correctly if elevation is not representative); requires accurate elevation function.
*   **When to use:** Satellite imagery rendering where solar direction is known and geometric consistency is important.
*   **When not to use:** Scenes with complex lighting (multiple light sources, non-directional); scenes where elevation is not a good proxy for occlusion.

## Background
This method was introduced in "Gaussian Splatting for Efficient Satellite Image Photogrammetry" (CVPR 2025) [S1]. It builds on the EO-NeRF shadow mapping concept but adapts it for 3DGS by using elevation renders instead of ray marching [S1]. The technique leverages the affine camera model to handle both satellite and sun cameras uniformly [S1].

## Sources
[S1] Gaussian Splatting for Efficient Satellite Image Photogrammetry, https://doi.org/10.1109/cvpr52734.2025.00559, fetched 2026-10-26
