# Per-Primitive Texture Mapping for Gaussian Splatting

Also called: TextureSplat

## Overview
Per-Primitive Texture Mapping for Gaussian Splatting assigns material properties as texture maps in the splat's local coordinate system rather than single attribute values per primitive, enabling higher-frequency spatial detail without increasing primitive count [S1]. It decouples geometry from appearance, allowing complex visual features and detailed normal maps to be captured within the existing Gaussian representation [S1]. This method differs from standard Gaussian Splatting by treating alpha-blending as a smoothing filter in a deferred rendering pipeline, which stabilizes the optimization of texture-sampled features and produces more cohesive results compared to shading directly on Gaussians [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of 2D Gaussian primitives, each defined by position, covariance, opacity, and rotation [S1].
*   A target image or set of images for training [S1].
*   A learnable environment map for direct lighting [S1].
*   An extracted mesh for indirect lighting visibility approximation [S1].

**Outputs:**
*   Rendered RGB image [S1].
*   Optimized primitive parameters: positions, covariances, opacities [S1].
*   Optimized texture maps for each primitive: Albedo, Metallic, Roughness, and Tangent Normal [S1].
*   Per-splat indirect lighting Spherical Harmonics (SH) coefficients [S1].

### Data Structures
Each primitive $k$ maintains the following optimized parameters [S1]:
*   **Splat parameters:** Position, covariance (or 2D projection parameters), opacity $o_k$.
*   **Local coordinates:** $t_{uk}, t_{vk}$ defining the local space of the Gaussian.
*   **Texture coordinates:** $s_{uk}, s_{vk}$ mapping local space to texture space.
*   **Material texture maps:** $T^k_\lambda$ (Albedo, $\in [0, 1]^3$), $T^k_m$ (Metallic, $\in [0, 1]$), $T^k_r$ (Roughness, $\in [0, 1]$), $T^k_n$ (Tangent Normal, $\in [0, 1]^3$) [S1].
*   **Indirect Lighting:** SH coefficients for $L^{ind}$ [S1].
*   **Environment Maps:** $L^{diffuse}_{env}$ and $L_{env}$ (specular) [S1].

**Texture Atlas:**
To leverage hardware-accelerated texture filtering, individual primitive textures are packed into texture atlases, inspired by Ptex and seamless texture atlases [S1]. The specific construction details are referenced in the supplementary material of the source [S1].

### Numbered Steps

1.  **Coordinate Mapping:**
    Map the local splat coordinates $(u, v)$ to texture coordinates $(s, t)$ accounting for the Gaussian kernel's support. The Gaussian kernel effectively drops to zero at approximately $S\sigma = 3$ standard deviations. The effective support $[-S\sigma, S\sigma] \times [-S\sigma, S\sigma]$ is mapped to texture space $[0, 1] \times [0, 1]$ [S1]:
    $$
    s = \frac{u + S\sigma}{2S\sigma}, \quad t = \frac{v + S\sigma}{2S\sigma}
    $$
    where $S\sigma = 3$ [S1].

2.  **Attribute Splatting (Deferred Rendering):**
    Splat material attributes to screen-space buffers using alpha blending. For each pixel $(x, y)$, the combined buffer $X(x, y)$ aggregates attributes from $N$ primitives [S1]:
    $$
    X(x, y) = \sum_{i=1}^{N} x_i(x, y) \alpha_i(x, y) \prod_{j=1}^{i-1} (1 - \alpha_j(x, y))
    $$
    where $x_i(x, y)$ is the attribute vector for primitive $i$ [S1]:
    $$
    x_i(x, y) = \begin{bmatrix} \lambda_i(x, y) \\ m_i(x, y) \\ r_i(x, y) \\ N_i(x, y) \\ L^{ind}_i(x, y) \end{bmatrix}
    $$
    Here, $\lambda$ is Albedo, $m$ is Metallic, $r$ is Roughness, $N$ is the world-space normal, and $L^{ind}$ is indirect lighting [S1].

3.  **Texture Sampling:**
    Obtain the attribute value $a_k(x, y)$ for primitive $k$ by bilinear filtering from its texture map $T^k$ using the mapped coordinates [S1]:
    $$
    a_k(x, y) = \text{BilinearFilter}(T^k, s(u_k(x, y)), t(v_k(x, y)))
    $$
    where $T^k$ is the texture map for primitive $k$, and $u_k, v_k$ are the local coordinates [S1].

4.  **Normal Reconstruction and Transformation:**
    *   **Reconstruction:** Tangent normals are encoded using only two components $(n_{tx}, n_{ty})$ for memory efficiency. The third component is reconstructed at runtime [S1]:
        $$
        n_{tz} = \max(0, \sqrt{1 - (n_{tx})^2 - (n_{ty})^2})
        $$
    *   **Transformation:** Transform the tangent normal $n_t^k(x, y)$ to world space using the primitive's rotation matrix $R_k$ [S1]:
        $$
        n_k(x, y) = R_k \cdot n_t^k(x, y)
        $$
        This enables detailed normal variations across the surface of each primitive [S1].

5.  **Physically-Based Shading:**
    Compute the outgoing radiance $L_o(x, y, \omega_o)$ in the direction $\omega_o$ using the deferred buffers [S1]:
    $$
    L_o(x, y, \omega_o) = L_d(x, y, \omega_o) + L_s(x, y, \omega_o)
    $$
    *   **Diffuse Term:**
        $$
        L_d(x, y, \omega_o) = \frac{(1 - M(x, y)) L^{diffuse}_{env}(N(x, y))}{\pi}
        $$
        where $M(x, y)$ is the aggregated metallic value and $L^{diffuse}_{env}$ is the pre-integrated diffuse environment irradiance [S1].
    *   **Specular Term:**
        $$
        L_s(x, y, \omega_o) \approx \text{BRDFLUT}(N(x, y) \cdot \omega_o, R(x, y)) \cdot (V(x, y)L_{dir}(x, y, \omega_r, R(x, y)) + (1 - V(x, y))L^{ind}(x, y))
        $$
        *   $\text{BRDFLUT}$ is a precomputed 2D lookup texture depending on view angle and roughness [S1].
        *   $L_{dir}$ is the direct environment lighting queried from a learnable environment map in the reflection direction $\omega_r$, using roughness $R(x, y)$ for mipmap selection [S1].
        *   $L^{ind}$ is the indirect lighting, modeled by approximating visibility with ray tracing an extracted mesh and encoding indirect lighting with spherical harmonics [S1].
        *   $V(x, y)$ represents visibility [S1].

6.  **Training Optimization:**
    Use a two-stage optimization process for stability [S1]:
    *   **Stage 1:** Train for half the total number of iterations using per-splat single attribute optimization (standard Gaussian Splatting attributes) [S1].
    *   **Stage 2:** Optimize material and normal textures initialized from the corresponding attributes from the first stage. Freeze the positions of the primitives [S1].
    *   **Loss Function:**
        $$
        \mathcal{L} = \mathcal{L}_{img} + \lambda_n \mathcal{L}_n
        $$
        where $\mathcal{L}_{img} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{D-SSIM}$ with balancing weight $\lambda = 0.2$ [S1].
    *   **Normal Consistency Loss:**
        $$
        \mathcal{L}_n = 1 - \tilde{N}(x, y)^T N(x, y)
        $$
        This minimizes the cosine difference between the rendered normal $N(x, y)$ and the surface normal $\tilde{N}(x, y)$ derived from rendered depth [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_per_primitive_texture_gaussians(primitives, camera, env_map, mesh):
    """
    Renders a scene using per-primitive texture mapping for Gaussian Splatting.
    
    Args:
        primitives: List of Gaussian primitives. Each primitive k has:
            - pos: 3D position
            - rot: Rotation matrix R_k
            - scale: Scale factors
            - opacity: o_k
            - tex_albedo: Texture map T^k_lambda
            - tex_metallic: Texture map T^k_m
            - tex_roughness: Texture map T^k_r
            - tex_normal: Texture map T^k_n (2 components stored)
            - sh_indirect: SH coefficients for indirect lighting
        camera: Camera parameters (view matrix, projection matrix, intrinsics)
        env_map: Learnable environment map for direct lighting
        mesh: Extracted mesh for indirect lighting visibility
        
    Returns:
        rendered_image: RGB image
    """
    # 1. Project Gaussians to 2D
    projected_gaussians = []
    for k in primitives:
        # Compute 2D covariance and inverse (builds on 2D Gaussian Projection with Covariance Stabilization)
        cov_2d, inv_cov_2d = project_gaussian_2d(k.pos, k.rot, k.scale, camera)
        
        # Compute local coordinates for each pixel covered by the Gaussian
        # This is typically done via a rasterization pass or kernel evaluation
        pixels = get_pixels_covered_by_gaussian(k.pos, cov_2d, camera.resolution)
        
        projected_gaussians.append({
            'k': k,
            'cov_2d': cov_2d,
            'inv_cov_2d': inv_cov_2d,
            'pixels': pixels
        })
    
    # 2. Initialize Screen-Space Buffers
    # Buffers store: Albedo (3), Metallic (1), Roughness (1), Normal (3), Indirect Lighting (SH coeffs)
    # For simplicity, we aggregate these into a single buffer per pixel
    width, height = camera.resolution
    buffer = np.zeros((height, width, 8)) # 3+1+1+3 = 8 channels for material/normal, plus indirect
    
    # 3. Splat Attributes to Buffers
    for pg in projected_gaussians:
        k = pg['k']
        pixels = pg['pixels']
        
        for (x, y) in pixels:
            # Get local coordinates (u, v) for this pixel relative to the Gaussian center
            u, v = get_local_coordinates(x, y, k.pos, k.rot, camera)
            
            # Map to texture coordinates
            S_sigma = 3.0
            s = (u + S_sigma) / (2 * S_sigma)
            t = (v + S_sigma) / (2 * S_sigma)
            
            # Sample textures
            # Note: BilinearFilter assumes texture coordinates are in [0, 1]
            albedo = bilinear_filter(k.tex_albedo, s, t)
            metallic = bilinear_filter(k.tex_metallic, s, t)
            roughness = bilinear_filter(k.tex_roughness, s, t)
            
            # Sample tangent normal (2 components)
            n_tx, n_ty = bilinear_filter_2comp(k.tex_normal, s, t)
            
            # Reconstruct tangent normal
            n_tz = max(0, sqrt(1 - n_tx**2 - n_ty**2))
            n_tangent = np.array([n_tx, n_ty, n_tz])
            
            # Transform to world space
            n_world = k.rot @ n_tangent
            
            # Get indirect lighting (SH evaluation)
            # Simplified: assume SH coefficients are stored and evaluated
            l_indirect = evaluate_sh(k.sh_indirect, n_world)
            
            # Attribute vector
            attr = np.concatenate([albedo, [metallic], [roughness], n_world, l_indirect])
            
            # Alpha blending into buffer
            alpha = compute_alpha(pg['cov_2d'], u, v)
            buffer[y, x] = buffer[y, x] * (1 - alpha) + attr * alpha
            
    # 4. Physically-Based Shading Pass
    rendered_image = np.zeros((height, width, 3))
    
    for y in range(height):
        for x in range(width):
            buf = buffer[y, x]
            albedo = buf[0:3]
            metallic = buf[3]
            roughness = buf[4]
            normal = buf[5:8]
            l_indirect = buf[8:] # Assuming SH coeffs or pre-evaluated value
            
            # View direction
            omega_o = get_view_direction(x, y, camera)
            
            # Diffuse term
            l_diffuse_env = get_diffuse_env_irradiance(normal, env_map)
            l_diffuse = ((1 - metallic) * l_diffuse_env) / np.pi
            
            # Specular term
            # BRDF LUT lookup
            ndotv = np.dot(normal, omega_o)
            brdf_lut_val = lookup_brdf_lut(ndotv, roughness)
            
            # Direct lighting
            omega_r = reflect(omega_o, normal)
            l_dir = get_env_map_sample(env_map, omega_r, roughness)
            
            # Visibility approximation (simplified)
            v = approx_visibility(normal, x, y, mesh, camera)
            
            l_specular = brdf_lut_val * (v * l_dir + (1 - v) * l_indirect)
            
            rendered_image[y, x] = l_diffuse + l_specular
            
    return rendered_image

def train_per_primitive_texture_gaussians(primitives, images, env_map, mesh, num_iterations):
    """
    Trains the Gaussian Splatting model with per-primitive textures.
    """
    half_iter = num_iterations // 2
    
    # Stage 1: Single attribute optimization
    for i in range(half_iter):
        # Standard Gaussian Splatting training loop
        # Optimize positions, covariances, opacities, and single attributes (color)
        loss = train_stage_1(primitives, images, env_map, mesh)
        optimize(primitives, loss)
        
    # Stage 2: Texture optimization
    # Initialize textures from Stage 1 attributes
    initialize_textures_from_attributes(primitives)
    
    # Freeze positions
    freeze_positions(primitives)
    
    for i in range(half_iter, num_iterations):
        # Render with textures
        rendered = render_per_primitive_texture_gaussians(primitives, camera, env_map, mesh)
        
        # Compute loss
        # L_img = (1 - 0.2) * L1 + 0.2 * L_D-SSIM
        l_img = compute_image_loss(rendered, target_image, lambda_weight=0.2)
        
        # L_n = 1 - N_rendered^T N_depth
        n_rendered = get_rendered_normal(primitives, camera)
        n_depth = get_depth_normal(target_depth, camera)
        l_n = compute_normal_loss(n_rendered, n_depth)
        
        total_loss = l_img + lambda_n * l_n
        
        # Optimize textures and SH coefficients
        optimize_textures(primitives, total_loss)
```

### Parameters
*   **Gaussian kernel support scaling factor:** $S\sigma = 3$ standard deviations [S1].
*   **Texture resolution:** $2 \times 2$ in all evaluations [S1].
*   **Balancing weight:** $\lambda = 0.2$ for the SSIM term in the image loss [S1].
*   **BRDFLUT:** Precomputed 2D lookup texture depending on view angle and roughness [S1].
*   **Hardware:** Single NVIDIA RTX A6000 GPU used in experiments [S1].

### Edge Cases and Pitfalls
*   **Hardware Acceleration:** Individual primitive textures must be packed into texture atlases to leverage hardware-accelerated texture filtering. This is inspired by Ptex and seamless texture atlases [S1].
*   **Precision:** Tangent normal reconstruction uses $\max(0, \dots)$ to handle numerical errors that might result in negative values under the square root [S1].
*   **Optimization Stability:** A two-stage optimization is required. Training textures directly from the start can lead to instability. Positions are frozen in the second stage to focus on appearance optimization [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** Increases significantly due to the storage of texture maps per primitive. Each primitive stores Albedo, Metallic, Roughness, and Tangent Normal textures. The source uses $2 \times 2$ resolution textures, which minimizes this overhead compared to higher resolutions [S1].
*   **Time:** The deferred rendering approach adds a shading pass after splatting. The texture sampling and bilinear filtering are hardware-accelerated via texture atlases, mitigating the cost of per-primitive texture lookups [S1].

### Measured Performance
*   Evaluated on standard reflective benchmarks for novel view synthesis [S1].
*   Uses a single NVIDIA RTX A6000 GPU [S1].
*   Qualitative results show improved fidelity in reflections compared to baseline methods like Ref-Gaussian, GShader, and 3DGSDR [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Decouples geometry and appearance, allowing high-frequency details without increasing primitive count [S1].
    *   Enables normal mapping for detailed specular effects [S1].
    *   Deferred rendering stabilizes optimization [S1].
*   **Limitations:**
    *   Requires texture atlas construction and management [S1].
    *   Indirect lighting approximation relies on ray tracing an extracted mesh, which may not capture all inter-reflections perfectly [S1].

### When to Use
*   Use when high-fidelity reflective surfaces and detailed normal maps are required.
*   Use when the primitive count is constrained but high-frequency appearance details are needed [S1].

### When Not to Use
*   Use standard Gaussian Splatting if memory is extremely constrained and high-frequency details are not critical.
*   Use if real-time performance is paramount and the overhead of texture atlas management and deferred shading is unacceptable [S1].

## Background

### Origin
The method was introduced in the paper "TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting" [S1].

### History and Development
The approach builds upon 2D Gaussian Splatting and reflective Gaussian splatting methods like Ref-Gaussian and 3DGSDR [S1]. It adapts techniques from traditional rendering (Gouraud vs. Phong shading) and texture mapping (Ptex) to the Gaussian Splatting framework [S1].

### Where it is Used
Used for novel view synthesis of real and synthetic scenes with reflective materials [S1].

## Sources

[S1] TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting, https://hal.science/hal-05484246v1, fetched 2026-10-26
