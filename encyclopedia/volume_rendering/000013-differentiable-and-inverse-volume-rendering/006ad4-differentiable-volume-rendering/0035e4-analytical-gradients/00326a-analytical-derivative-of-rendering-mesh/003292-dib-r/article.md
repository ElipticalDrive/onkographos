# DIB-R

Also called: Chen et al. method, Hybrid Differentiable Renderer

## Overview
DIB-R is a differentiable renderer that addresses the non-differentiability of standard rasterization by partitioning pixels into foreground (covered by mesh faces) and background (uncovered) regions [S1, unofficial]. It computes exact analytical gradients for foreground pixels using barycentric interpolation and prevents vanishing gradients for background pixels via distance-based aggregation [S1, unofficial]. The method serves as the geometric differentiability foundation for DIB-R++, a hybrid renderer that extends this framework to support photorealistic shading, environmental lighting, and inverse graphics without ground-truth 3D supervision [S2, unofficial].

## How it works
DIB-R operates in two distinct stages: a rasterization pass to generate geometric buffers and a shading pass to compute radiance and gradients [S2, unofficial]. The method separates the image $I$ into foreground $I_f$ and background $I_b$ components based on visibility [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Mesh $M$ defined by vertices $V$ and faces $F$ [G].
*   Camera parameters $\pi$ (intrinsic and extrinsic) [G].
*   Material parameters $\theta$ (e.g., diffuse albedo, roughness) [S2, unofficial].
*   Lighting parameters $\gamma$ (e.g., environment map coefficients) [S2, unofficial].
*   Image resolution $H \times W$ [G].

**Outputs:**
*   Rendered image $I \in \mathbb{R}^{3 \times H \times W}$ [S2, unofficial].
*   Gradients $\frac{\partial I}{\partial \Theta}$ with respect to geometry $\pi$, material $\theta$, and lighting $\gamma$ [S2, unofficial].

### Data Structures
The rasterization pass produces G-buffers for each pixel $p$:
*   $x_p \in \mathbb{R}^3$: Surface intersection point [S2, unofficial].
*   $n_p \in \mathbb{R}^3$: Surface normal (unit vector) [S2, unofficial].
*   $\theta_p$: Spatially varying material parameters at $p$ [S2, unofficial].
*   $v_p \in \{0, 1\}$: Visibility mask. $v_p = 1$ if the pixel is covered by a face (foreground), $v_p = 0$ otherwise (background) [S2, unofficial].

### Numbered Steps

1.  **Rasterization Pass (Foreground/Background Separation):**
    Employ a differentiable rasterizer $\mathcal{R}$ to render mesh $M$ into G-buffers [S2, unofficial]. For each pixel $p$, determine the closest intersecting face.
    *   Compute the intersection point $x_p$, normal $n_p$, and material parameters $\theta_p$ via standard rasterization and barycentric interpolation of the face's vertex attributes [S2, unofficial].
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
import numpy as np

def dib_r_forward(mesh, camera, material_params, lighting_params):
    # Stage 1: Rasterization Pass
    x, n, theta, v = rasterize(mesh, camera)
    # Stage 2: Shading Pass
    radiance = shade(x, n, camera.outgoing_direction, material_params, lighting_params)
    return radiance, x, n, theta, v

def rasterize(mesh, camera):
    # Standard differentiable rasterization: find closest face per pixel
    # Intersect rays, compute barycentric coordinates, interpolate attributes
    # Returns (x, n, theta, v) as defined in [S2, unofficial]
    # Implementation relies on standard graphics routines [G]
    pass

def shade(x, n, omega_o, theta, gamma):
    # Simplified isotropic Disney BRDF with distant environment map
    # Computes S(x_p, n_p, omega_o; theta_p, gamma) approx Lo(x_p, omega_o) [S2, unofficial]
    # Direct illumination only, single-bounce scattering [S2, unofficial]
    pass

def dib_r_backward(mesh, camera, material_params, lighting_params, grad_output, x, n, theta, v):
    # Split gradients by visibility mask
    grad_fg = grad_output * v
    grad_bg = grad_output * (1 - v)

    # Foreground gradients (v=1): analytical derivatives via barycentric interpolation
    # Gradient flows through x_p and n_p to mesh vertices [S1, unofficial][S3]
    grad_vertices_fg = barycentric_gradient_propagation(mesh, x, n, grad_fg)
    grad_theta_fg = np.sum(grad_fg * theta, axis=(0, 1))

    # Background gradients (v=0): distance-based aggregation of global face info
    # Approximates gradient flow through alpha channel to prevent vanishing gradients [S1, unofficial][S3]
    grad_vertices_bg = distance_based_aggregation(mesh, camera, grad_bg)
    grad_theta_bg = np.zeros_like(theta)

    # Lighting gradients via chain rule on shading model [S2, unofficial]
    grad_lighting = shading_lighting_backward(x, n, camera.outgoing_direction, material_params, lighting_params, grad_output)

    return grad_vertices_fg + grad_vertices_bg, grad_theta_fg + grad_theta_bg, grad_lighting

def barycentric_gradient_propagation(mesh, x, n, grad_color):
    # Propagate gradients from pixel space to mesh vertices using barycentric coordinates
    # Standard differentiable rasterization backward pass [G]
    pass

def distance_based_aggregation(mesh, camera, grad_bg):
    # Aggregate global face information weighted by distance to pixel
    # Implementation detail not specified in sources [G]
    pass

def shading_lighting_backward(x, n, omega_o, theta, gamma, grad_output):
    # Chain rule gradient for lighting parameters gamma
    # Differentiable parameterization of S enables joint optimization [S2, unofficial]
    pass
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
*   **Time Complexity:** Not given in the sources. Shading complexity depends on the chosen model (MC is more expensive than SG) [S2, unofficial].
*   **Memory Complexity:** Requires storing G-buffers ($x, n, \theta, v$) for each pixel [S2, unofficial].
*   **Strengths:** Sharp foreground edges due to analytical gradients; handles non-Lambertian specular reflections and environmental lighting in DIB-R++ variant [S2, unofficial].
*   **Limitations:** Background gradients are approximate and depend on reference masks; ignores self-occlusion and global illumination [S2, unofficial][S3].
*   **When to Use:** Inverse graphics tasks requiring joint optimization of geometry, material, and lighting from single images, especially when photorealistic specular effects are needed [S2, unofficial].

## Background
DIB-R was proposed by Chen et al. [S1, unofficial][S2, unofficial]. It was introduced to address the blurriness and vanishing gradients associated with soft rasterization methods like Soft Rasterizer [S1, unofficial]. The method builds on the analytical gradient approach for foreground pixels by Kyle (TF Mesh Renderer) [S3]. DIB-R++ is an extension that adds hybrid rasterization and ray-tracing capabilities for photorealistic shading [S2, unofficial]. The method is used in inverse graphics for predicting intrinsic object properties from single images [S2, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] DIB-R++: Learning to Predict Lighting and Material with a Hybrid Differentiable Renderer, https://doi.org/10.48550/arxiv.2111.00140, fetched 2026-10-26 (unofficial)
[S3] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
