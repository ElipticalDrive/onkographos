# 3D Gaussian Splatting with Reflection and Relighting

Also called: Gaussian-Shader, 3DGS-DR, IRGS, Relightable 3D Gaussians, GS-IR

## Overview
This method extends standard 3D Gaussian Splatting (3DGS) by incorporating classical physically based rendering (PBR) concepts to address limitations with highly reflective surfaces and enable relighting under novel illumination [S1][S2, unofficial]. Unlike standard 3DGS, which bakes lighting into spherical harmonics coefficients via an emission-absorption model, this approach decomposes geometry, material, and lighting, allowing for dynamic specular shading synthesis [S1, S3]. It is distinguished from sibling methods by its specific integration of secondary ray tracing for interreflections and the derivation of normals from the Gaussian kernel's anisotropy [S1, S3].

## How it works
**Inputs:**
*   Optimized 3D Gaussian primitives (position, covariance, opacity, base color/SH).
*   Input images/videos for training.
*   Environment map or lighting parameters for relighting.

**Outputs:**
*   Relit images under novel illumination.
*   Decomposed material maps (albedo, roughness, metallic).

**Data Structures:**
Each Gaussian $i$ stores additional PBR attributes beyond standard 3DGS:
*   **Albedo ($\alpha_i$):** Per-Gaussian diffuse color.
*   **Roughness ($\rho_i$):** Scalar or vector defining surface microfacet distribution.
*   **Metallic ($m_i$):** Scalar defining the blend between dielectric and conductor.
*   **Normal ($\mathbf{n}_i$):** Derived from the Gaussian shape or explicitly optimized.
*   **Lighting:** Scene environment map or learned lighting coefficients.

**Method Steps:**

1.  **Normal Estimation:**
    The surface normal $\mathbf{n}_i$ for each Gaussian is derived from the shortest axis of the Gaussian kernel's covariance matrix $\Sigma_i$. This leverages the anisotropy of the Gaussian to approximate surface orientation without expensive normal estimation networks [S3].
    Let $\Sigma_i$ be the $3 \times 3$ covariance matrix. The eigenvector corresponding to the smallest eigenvalue provides the normal direction [S3].

2.  **Material Decomposition:**
    During training, the model optimizes per-Gaussian albedo, roughness, and metallic values. These parameters define the Bidirectional Reflectance Distribution Function (BRDF) for each primitive [S2, unofficial].

3.  **Primary Ray Intersection:**
    For each pixel, a primary ray is cast. The ray intersects Gaussians, and the 2D Gaussian value $\tilde{\mathcal{G}}_i(\mathbf{p})$ is computed at the intersection point $\mathbf{p}$ using the projected covariance $\Sigma_{2D}$ [G].

4.  **BRDF Evaluation:**
    The color contribution is calculated using a PBR shading model (e.g., Cook-Torrance) rather than simple spherical harmonics. The direct lighting is reconstructed using the environment map, while indirect lighting may be modeled via spherical harmonics or baked into the albedo if precomputed [S1].

5.  **Secondary Ray Tracing (Interreflections):**
    To model interreflections, secondary rays (reflections and refractions) are spawned upon primary ray interactions with Gaussians. Ray tracing techniques are integrated to trace these secondary rays, allowing light to bounce between surfaces [S1]. This step distinguishes it from methods that only compute direct lighting [S1].

6.  **Compositing:**
    Colors are accumulated using front-to-back alpha blending. The final intensity $I_f$ for a pixel is:
    $$ I_f = \sum_{i=1}^N c_i \tilde{\sigma}_i \tilde{\mathcal{G}}_i(\mathbf{p}) \prod_{j=1}^{i-1} (1 - \tilde{\sigma}_j \tilde{\mathcal{G}}_j(\mathbf{p})) $$
    where $c_i$ is the PBR-computed color (including direct and indirect lighting contributions), $\tilde{\sigma}_i$ is the opacity, and $\tilde{\mathcal{G}}_i(\mathbf{p})$ is the 2D Gaussian density [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gaussian_splatting_with_reflection(gaussians, camera, env_map):
    """
    Renders a scene using 3D Gaussians with PBR and secondary rays.
    """
    image_height, image_width = camera.resolution
    output_image = zeros((image_height, image_width, 3))
    
    # Precompute normals from covariance matrices
    for g in gaussians:
        # Eigenvector of smallest eigenvalue of covariance matrix Sigma
        eigenvalues, eigenvectors = eig(g.covariance)
        g.normal = eigenvectors[:, argmin(eigenvalues)]
        
    # Sort gaussians by depth for compositing
    gaussians = sort_by_depth(gaussians, camera)
    
    for y in range(image_height):
        for x in range(image_width):
            # Cast primary ray
            ray = camera.get_ray(x, y)
            
            # Collect visible gaussians along ray
            visible_gaussians = []
            for g in gaussians:
                if intersects_ray(g.bounding_box, ray):
                    # Compute 2D Gaussian value at intersection
                    # Project covariance to 2D: Sigma_2D = J * W * Sigma * W^T * J^T
                    sigma_2d = project_covariance(g.covariance, camera)
                    t = intersect_ray_gaussian(ray, g)
                    if t is not None:
                        p = ray.origin + t * ray.direction
                        g_val = gaussian_2d_value(p, g.center, sigma_2d)
                        if g_val > epsilon:
                            visible_gaussians.append((g, g_val, t))
            
            # Sort by depth (front-to-back)
            visible_gaussians.sort(key=lambda item: item[2])
            
            accumulated_color = zeros(3)
            accumulated_alpha = 0.0
            
            for g, g_val, t in visible_gaussians:
                # Evaluate BRDF
                # Direct lighting from env map
                direct_light = evaluate_direct_lighting(g.normal, ray.direction, env_map)
                
                # Secondary rays for interreflections
                # Spawn reflection ray
                reflection_ray = reflect_ray(ray, g.normal)
                indirect_light = trace_secondary_ray(reflection_ray, gaussians, env_map)
                
                # Combine lighting
                total_lighting = direct_light + indirect_light
                
                # Compute material color
                # c_i = albedo * diffuse_term + specular_term * total_lighting
                color = compute_pbr_color(g.albedo, g.roughness, g.metallic, 
                                          g.normal, ray.direction, total_lighting)
                
                # Alpha blending
                alpha = g.opacity * g_val
                accumulated_color += color * alpha * (1 - accumulated_alpha)
                accumulated_alpha += alpha * (1 - accumulated_alpha)
                
                if accumulated_alpha > 1.0 - epsilon:
                    break
            
            output_image[y, x] = accumulated_color
            
    return output_image

def trace_secondary_ray(ray, gaussians, env_map):
    """
    Traces a secondary ray (reflection/refraction) to model interreflections.
    """
    # Simple recursive trace or limited depth
    # Intersect with scene geometry (Gaussians)
    closest_hit = None
    min_t = infinity
    
    for g in gaussians:
        if intersects_ray(g.bounding_box, ray):
            t = intersect_ray_gaussian(ray, g)
            if t is not None and t < min_t:
                min_t = t
                closest_hit = g
                
    if closest_hit is None:
        # Hit background, sample environment map
        return sample_env_map(ray.direction, env_map)
    else:
        # Recursive call or bounce
        # For simplicity, assume one bounce
        new_ray = reflect_ray(ray, closest_hit.normal)
        return trace_secondary_ray(new_ray, gaussians, env_map)
```

**Parameters:**
*   **Ray Tracing Depth:** Not given in the sources. [G] Typically limited to 1-2 bounces for performance.
*   **BRDF Model:** Cook-Torrance or similar microfacet model. [S1]

**Edge Cases and Pitfalls:**
*   **Normal Estimation Accuracy:** The normal derived from the shortest axis of the Gaussian kernel may be inaccurate for highly anisotropic or noisy Gaussians [S3].
*   **Training Stability:** Optimizing per-Gaussian BRDF parameters can be unstable. Constraints such as anisotropy loss may be needed to prevent excessive stretching [S3].
*   **Performance:** Secondary ray tracing significantly increases computational cost compared to standard 3DGS [S1].

## Operating characteristics
*   **Time Complexity:** Higher than standard 3DGS due to BRDF evaluation and secondary ray tracing. [S1]
*   **Memory Complexity:** Increased storage per Gaussian for albedo, roughness, metallic, and normal attributes. [S2, unofficial]
*   **Strengths:** Enables relighting under novel illumination; handles highly reflective surfaces better than standard 3DGS; leverages 3DGS's natural normal estimation for efficiency. [S1, S3]
*   **Limitations:** Slower rendering speed due to ray tracing; complex training process; potential artifacts from inaccurate normal estimation. [S1, S3]
*   **When to use:** Scenes with significant specular reflections or when relighting is required. [S1]
*   **When not to use:** Real-time applications requiring maximum performance where reflections are not critical; scenes with complex subsurface scattering not modeled by the BRDF. [S1]

## Background
Standard 3D Gaussian Splatting uses an emission-absorption model, which bakes lighting into spherical harmonics, limiting its ability to render reflective surfaces or support relighting [S1]. Methods like Gaussian-Shader, 3DGS-DR, and IRGS were developed to incorporate classical reflection and shading concepts into the 3DGS framework [S1]. These methods estimate per-Gaussian BRDF and lighting decomposition, enabling relighting under novel illumination [S2, unofficial]. The approach leverages the fact that 3DGS naturally offers good normal estimation via the shortest axis of the Gaussian kernel, avoiding the time-consuming training and slow rendering speeds associated with normal estimation in NeRF-based PBR methods [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-09
