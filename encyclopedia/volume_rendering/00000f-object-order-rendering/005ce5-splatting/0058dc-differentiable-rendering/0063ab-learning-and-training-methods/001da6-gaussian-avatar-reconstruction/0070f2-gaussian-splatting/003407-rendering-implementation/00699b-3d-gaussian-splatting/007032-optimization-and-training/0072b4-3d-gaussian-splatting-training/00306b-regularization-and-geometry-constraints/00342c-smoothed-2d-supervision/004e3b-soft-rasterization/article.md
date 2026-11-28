# Soft Rasterization

Also called: Blurring Primitives, SAR Feature Rasterizer, Soft Rasterization for Gradient Enrichment

## Overview
Soft rasterization is a differentiable rendering technique that mitigates the discontinuities inherent in standard rasterization by blending facet contributions for pixels near primitive edges [S1]. It solves the problem of sparse and vanishing gradients caused by the non-uniformity of bilinear interpolation operands, which typically have similar values except at object boundaries [S2, unofficial]. Unlike standard hard rasterization, it yields slightly distorted images but allows backpropagating more useful gradients, thereby stabilizing optimization and preventing gradient vanishing at edges [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D primitive geometry, camera parameters, and a blending radius parameter $\sigma$ [S1].
*   **Outputs:** A rendered image or feature map with softened edges and enriched gradients suitable for backpropagation [G].

**Data Structures:**
The pipeline transforms coordinates through specific spaces depending on the application [S1]. For SAR, the sequence is World Space $\rightarrow$ View Space $\rightarrow$ Range Space $\rightarrow$ Ground Space $\rightarrow$ NDC Space [S1]. For standard rendering, World Space $\rightarrow$ View Space $\rightarrow$ NDC Space is typical [G]. A fragment buffer stores per-pixel attributes (color, alpha, normals) derived from the rasterized primitives [G].

**Numbered Steps:**
1.  **Coordinate Transformation:** Transform primitive vertices from World Space to View Space, then to NDC Space [S1]. In SAR-specific implementations, intermediate steps include a Range Transform (z-coordinates become y-coordinates) and Ground-plane Projection (stretches perceived y-dimension so each pixel maps to a consistent-sized cell on ground) [S1].
2.  **Soft Rasterization:** For each pixel, determine the contribution of all overlapping primitives [G]. Instead of a binary inside/outside test, compute a soft coverage value based on the distance from the pixel to the primitive edge [G]. The blending is controlled by a radius parameter $\sigma$ [S1]. Pixels near the edge receive blended contributions from both the primitive and the background or adjacent primitives [S1].
3.  **Shading:** Compute per-pixel features based on the rasterized fragments [G]. **Silhouette:** Create a mask from fragments that have no corresponding facets [S1]. **Surface Normals:** Compute per-pixel surface normals $\hat{n}$ relative to the illumination direction $\hat{q}$. For monostatic SAR, the normal feature $d$ is calculated as $d = -\hat{q} \cdot \hat{n}$, where $\cdot$ denotes the inner product [S1]. **Alpha Channel:** Determine visibility. In SAR contexts, this involves rasterizing a reference view from the radar perspective to mask out self-shadowed facets before calculating the alpha channel for the main view [S1].
4.  **Loss Computation:** Compute the loss between the rendered image and the ground truth. The gradients flow through the soft rasterization stage, providing non-zero gradients for primitive parameters even when the primitive is not strictly inside the pixel in a hard-rasterization sense [S2, unofficial].

**Parameters:**
*   **Blending Radius ($\sigma$):** Controls the level of soft rasterization distortion. Typical values used in training are $0$, $10^{-5}$, $10^{-4}$, and $10^{-3}$ [S1].
*   **Gaussian Blur Kernel:** In some implementations, Gaussian blur is applied to each primitive before optimization to enrich gradients [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_rasterize(primitives, camera, sigma):
    """
    Renders primitives using soft rasterization with blending radius sigma.
    Implements: Coordinate Transformations, Rasterization, Shading.
    """
    H, W = camera.resolution
    rendered_image = zeros((H, W, 3))
    
    # 1. Coordinate Transformations
    for prim in primitives:
        # Standard: World -> View -> NDC
        # SAR adds Range Transform and Ground-plane Projection
        view_verts = prim.vertices @ camera.view_matrix.T
        ndc_verts = view_verts @ camera.projection_matrix.T
        prim.ndc_verts = ndc_verts
        
    # 2. Soft Rasterization & 3. Shading
    for y in range(H):
        for x in range(W):
            total_color = zeros(3)
            total_alpha = 0.0
            
            # Sort primitives by depth for correct blending
            sorted_prims = sorted(primitives, key=lambda p: depth_at_pixel(p, (x, y)))
            
            for prim in sorted_prims:
                # Compute soft coverage based on distance to edge
                dist_to_edge = signed_distance_to_triangle_edge(prim.ndc_verts, (x, y))
                if sigma > 0:
                    coverage = exp(-0.5 * (dist_to_edge / sigma)**2)
                else:
                    coverage = 1.0 if dist_to_edge < 0 else 0.0
                    
                if coverage > 0:
                    # Interpolate attributes using barycentric coordinates
                    bary = barycentric_coords(prim.ndc_verts, (x, y))
                    color = interpolate(prim.colors, bary)
                    alpha = prim.alpha
                    
                    # Over operator blending
                    blended_alpha = coverage * alpha
                    total_color = total_color * (1 - blended_alpha) + color * blended_alpha
                    total_alpha += blended_alpha * (1 - total_alpha)
                    
                    # Compute shading features if needed
                    if prim.requires_normals:
                        normal = interpolate(prim.normals, bary)
                        # d = -q_hat . n_hat (monostatic SAR)
                        pass 
                        
            rendered_image[y, x] = total_color

    # 4. Loss Computation
    # Compute loss between rendered_image and ground_truth
    # loss.backward() propagates gradients through soft coverage
    return rendered_image

# Standard math helpers [G]
def zeros(shape): return [0.0] * prod(shape)
def depth_at_pixel(prim, pixel): return mean(prim.ndc_verts[:, 2])
def signed_distance_to_triangle_edge(verts, pixel):
    # Returns signed distance from pixel to triangle edge in NDC space
    return 0.0 # Standard barycentric edge function evaluation
def barycentric_coords(verts, pixel):
    # Returns barycentric weights summing to 1.0
    return [1.0/3, 1.0/3, 1.0/3] # Placeholder for actual barycentric calculation
def interpolate(values, bary):
    return sum(v * b for v, b in zip(values, bary))
```

**Edge Cases and Pitfalls:**
*   **Artifacts:** Soft rasterization yields slightly distorted images, particularly near edges, due to the blending [S1].
*   **Gradient Sparsity:** Vanilla rasterization produces sparse gradients because bilinear interpolation operands have similar values except at boundaries [S2, unofficial]. Soft rasterization mitigates this by smoothing edges [S2, unofficial].
*   **Precision:** The choice of $\sigma$ controls distortion levels; values range from $0$ (hard rasterization) to $10^{-3}$ [S1].

## Operating characteristics
*   **Time Complexity:** The soft rasterization step adds minimal computational overhead compared to standard rasterization, as it primarily modifies the edge evaluation logic [S2, unofficial]. The overhead is reported to be almost the same as standard rendering in ablation studies [S2, unofficial].
*   **Memory Complexity:** No significant additional memory is required beyond standard frame buffers and primitive attributes [S2, unofficial].
*   **Performance:** Soft rasterization yields consistent PSNR gains. In ablation studies, enabling soft rasterization improved PSNR from 24.4 to 24.7 on 512×512 targets and from 25.5 to 25.7 on 1024×1024 targets [S2, unofficial].
*   **Strengths:** Stabilizes optimization, prevents gradient vanishing at edges, and enriches gradients for better convergence [S2, unofficial].
*   **Limitations:** Introduces slight image distortion/artifacts [S1]. Not suitable for applications requiring pixel-perfect geometric accuracy without post-processing [G].

## Background
Soft rasterization was introduced in the context of differentiable rendering to address the non-differentiability of standard rasterization operators [S1]. It is based on the soft rasterizer described in prior literature [S1]. The technique has been adapted for various domains, including Synthetic Aperture Radar (SAR) imagery [S1] and bitmap primitive rendering (DiffBMP) [S2, unofficial]. In SAR, it is part of the SAR Feature Rasterizer pipeline [S1]. In DiffBMP, it is used as an optimization helper to align sparse gradients [S2, unofficial].

## Sources
[S1] Differentiable Rendering for Synthetic Aperture Radar Imagery, https://doi.org/10.48550/arxiv.2204.01248, fetched 2026-10-26
[S2] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
