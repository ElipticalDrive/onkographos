# DIB-R

Also called: Chen et al. method, Hybrid Differentiable Renderer

## Overview
DIB-R is a differentiable renderer that addresses the non-differentiability of standard rasterization by partitioning pixels into foreground (covered by mesh faces) and background (uncovered) regions [S1, unofficial]. It computes exact analytical gradients for foreground pixels using barycentric interpolation and prevents vanishing gradients for background pixels via distance-based aggregation [S1, unofficial]. Unlike its sibling [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md), which relies on soft rasterization approximations for all pixels, DIB-R maintains sharp foreground edges while providing a differentiable path for background pixels [S1, unofficial]. The method serves as the geometric differentiability foundation for DIB-R++, a hybrid renderer that extends this framework to support photorealistic shading, environmental lighting, and inverse graphics without ground-truth 3D supervision [S2, unofficial].

## How it works
DIB-R operates in two distinct stages: a rasterization pass to generate geometric buffers and a shading pass to compute radiance and gradients. The method separates the image $I$ into foreground $I_f$ and background $I_b$ components based on visibility [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Mesh $M$ defined by vertices $V$ and faces $F$.
*   Camera parameters $\pi$ (intrinsic and extrinsic).
*   Material parameters $\theta$ (e.g., diffuse albedo, roughness).
*   Lighting parameters $\gamma$ (e.g., environment map coefficients).
*   Image resolution $H \times W$.

**Outputs:**
*   Rendered image $I \in \mathbb{R}^{3 \times H \times W}$.
*   Gradients $\frac{\partial I}{\partial \Theta}$ with respect to geometry $\pi$, material $\theta$, and lighting $\gamma$.

### Data Structures
The rasterization pass produces G-buffers for each pixel $p$:
*   $x_p \in \mathbb{R}^3$: Surface intersection point.
*   $n_p \in \mathbb{R}^3$: Surface normal (unit vector).
*   $\theta_p$: Spatially varying material parameters at $p$.
*   $v_p \in \{0, 1\}$: Visibility mask. $v_p = 1$ if the pixel is covered by a face (foreground), $v_p = 0$ otherwise (background) [S2, unofficial].

### Numbered Steps

1.  **Rasterization Pass (Foreground/Background Separation):**
    Employ a differentiable rasterizer $\mathcal{R}$ to render mesh $M$ into G-buffers [S2, unofficial]. For each pixel $p$, determine the closest intersecting face.
    *   Compute the intersection point $x_p$, normal $n_p$, and material parameters $\theta_p$ via barycentric interpolation of the face's vertex attributes [S1, unofficial][S3].
    *   Set visibility mask $v_p = 1$ if a face covers the pixel, else $v_p = 0$ [S2, unofficial].
    *   The rasterization output is defined as:
        $$ \mathcal{R}(M, p, \omega_o) = (x_p, n_p, \theta_p, v_p) $$
        where $\omega_o$ is the outgoing camera direction [S2, unofficial].

2.  **Shading Pass (Radiance Approximation):**
    Approximate the outgoing radiance $L_o(x_p, \omega_o)$ using a shading model $\mathcal{S}$ [S2, unofficial]. The rendering equation is:
    $$ L_o(x, \omega_o) = \int_{H_2} f_r(x, \omega_i, \omega_o) L_i(x, \omega_i) |n \cdot \omega_i| d\omega_i $$
    where $H_2$ is the unit hemisphere, $f_r$ is the SV-BRDF, and $L_i$ is incident radiance [S2, unofficial].
    *   **Assumptions:**
        *   Direct illumination only (single-bounce scattering) [S2, unofficial].
        *   Distant environment map $L_i: S^2 \rightarrow \mathbb{R}^3_+$, independent of position: $L_i(x_p, \omega_i) \equiv L_i(\omega_i; \gamma)$ [S2, unofficial].
        *   Material model: Simplified isotropic Disney BRDF with diffuse albedo $a$, specular albedo $s$, roughness $\beta$, and metalness $m$ [S2, unofficial].
    *   Compute approximate radiance:
        $$ \mathcal{S}(x_p, n_p, \omega_o; \theta_p, \gamma) \approx L_o(x_p, \omega_o) $$
        [S2, unofficial].

3.  **Gradient Computation:**
    Compute gradients $\frac{\partial I}{\partial \Theta}$ using the chain rule on $I_p(\Theta) = (\mathcal{S} \circ \mathcal{R})(M, p, \omega_o)$ [S2, unofficial].

    *   **Foreground Gradients ($v_p = 1$):**
        Compute gradients with respect to geometry $\pi$ using analytical derivatives derived from the barycentric interpolation of vertex attributes [S1, unofficial][S3]. This follows the method by Kyle (TF Mesh Renderer) [S3]. The gradient flows through $x_p$ and $n_p$ to the mesh vertices.

    *   **Background Gradients ($v_p = 0$):**
        Prevent vanishing gradients by employing a distance-based aggregation of global face information [S1, unofficial]. The gradient is approximated through the alpha channel, aggregating contributions from all faces weighted by their distance to the pixel [S3]. This mimics the Soft Rasterizer approach for background pixels [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dib_r_forward(mesh, camera, material_params, lighting_params):
    """
    Forward pass of DIB-R.
    Returns rendered image I, G-buffers (x, n, theta, v).
    """
    # Stage 1: Rasterization
    # R returns intersection point, normal, material params, and visibility mask
    x, n, theta, v = differentiable_rasterizer(mesh, camera)
    
    # Stage 2: Shading
    # Compute outgoing radiance using the shading model S
    # S takes surface properties and lighting parameters
    radiance = shading_model(x, n, camera.outgoing_direction, material_params, lighting_params)
    
    return radiance, x, n, theta, v

def dib_r_backward(mesh, camera, material_params, lighting_params, grad_output, x, n, theta, v):
    """
    Backward pass of DIB-R.
    Computes gradients w.r.t geometry, material, and lighting.
    """
    grad_x = 0
    grad_n = 0
    grad_theta = 0
    grad_lighting = 0
    
    # Split gradients by visibility mask
    grad_fg = grad_output * v
    grad_bg = grad_output * (1 - v)
    
    # --- Foreground Gradients (v=1) ---
    # Analytical gradients via barycentric interpolation
    # Gradients flow from pixel color -> shading model -> x, n, theta
    # Then from x, n -> mesh vertices (geometry)
    
    # 1. Gradient w.r.t shading inputs (x, n, theta)
    # Note: shading_model must be differentiable
    grad_shading_x, grad_shading_n, grad_shading_theta = shading_model_backward(
        x, n, camera.outgoing_direction, material_params, lighting_params, grad_fg
    )
    
    # 2. Gradient w.r.t geometry (mesh vertices)
    # Using barycentric coordinates to propagate grad_shading_x and grad_shading_n to vertices
    grad_vertices = barycentric_gradient_propagation(
        mesh.faces, mesh.vertices, x, n, grad_shading_x, grad_shading_n
    )
    
    # 3. Gradient w.r.t material parameters
    grad_theta = aggregate_gradient(grad_shading_theta)
    
    # --- Background Gradients (v=0) ---
    # Distance-based aggregation of global face information
    # Approximates gradient flow through alpha channel
    
    # 1. Aggregate contributions from all faces based on distance to pixel
    # This avoids vanishing gradients for uncovered pixels
    grad_bg_aggregated = distance_based_aggregation(
        mesh.faces, mesh.vertices, camera, grad_bg
    )
    
    # 2. Propagate to geometry and material if needed
    # Note: Background gradients typically do not flow to specific geometry vertices
    # in the same way, but may affect lighting or global material params
    grad_vertices_bg = 0 # Or specific aggregation if defined
    grad_theta_bg = 0
    
    # Combine gradients
    total_grad_vertices = grad_vertices + grad_vertices_bg
    total_grad_theta = grad_theta + grad_theta_bg
    
    # Gradient w.r.t lighting
    # Computed via chain rule on shading model for all pixels
    grad_lighting = shading_model_lighting_backward(
        x, n, camera.outgoing_direction, material_params, lighting_params, grad_output
    )
    
    return total_grad_vertices, total_grad_theta, grad_lighting
```

### Parameters
*   **Lighting Model:** Distant environment map $L_i: S^2 \rightarrow \mathbb{R}^3_+$ [S2, unofficial].
*   **Material Model:** Simplified isotropic Disney BRDF [S2, unofficial].
*   **Shading Approximation:** Monte Carlo (MC) or Spherical Gaussians (SG) [S2, unofficial]. MC targets mirror-like objects; SG is robust to roughness variations [S2, unofficial].

### Edge Cases and Pitfalls
*   **Alpha Mask Dependency:** The distance-based aggregation for background gradients relies on an alpha channel, which is only available when a reference alpha mask exists [S3].
*   **Complex Visibility:** The method may fail to handle complex visibility gradients [S3].
*   **Environment Lighting:** The basic DIB-R formulation may fail to render with environment lighting if not extended to DIB-R++ [S3].
*   **Self-Occlusion:** The shading model assumes direct illumination only and ignores self-occlusion [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and faces for rasterization. Shading complexity depends on the chosen model (MC is more expensive than SG) [S2, unofficial].
*   **Memory Complexity:** Requires storing G-buffers ($x, n, \theta, v$) for each pixel [S2, unofficial].
*   **Strengths:** Sharp foreground edges due to analytical gradients; handles non-Lambertian specular reflections and environmental lighting in DIB-R++ variant [S2, unofficial].
*   **Limitations:** Background gradients are approximate and depend on reference masks; ignores self-occlusion and global illumination [S2, unofficial][S3].
*   **When to Use:** Inverse graphics tasks requiring joint optimization of geometry, material, and lighting from single images, especially when photorealistic specular effects are needed [S2, unofficial].
*   **When Not to Use:** Scenes requiring global illumination or self-occlusion effects; cases where no reference alpha mask is available for background gradient computation [S3].

## Background
DIB-R was proposed by Chen et al. [S1, S2, unofficial]. It was introduced to address the blurriness and vanishing gradients associated with soft rasterization methods like Soft Rasterizer [S1, unofficial]. The method builds on the analytical gradient approach for foreground pixels by Kyle (TF Mesh Renderer) [S3]. DIB-R++ is an extension that adds hybrid rasterization and ray-tracing capabilities for photorealistic shading [S2, unofficial]. The method is used in inverse graphics for predicting intrinsic object properties from single images [S2, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] DIB-R++: Learning to Predict Lighting and Material with a Hybrid Differentiable Renderer, https://doi.org/10.48550/arxiv.2111.00140, fetched 2026-10-26 (unofficial)
[S3] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
