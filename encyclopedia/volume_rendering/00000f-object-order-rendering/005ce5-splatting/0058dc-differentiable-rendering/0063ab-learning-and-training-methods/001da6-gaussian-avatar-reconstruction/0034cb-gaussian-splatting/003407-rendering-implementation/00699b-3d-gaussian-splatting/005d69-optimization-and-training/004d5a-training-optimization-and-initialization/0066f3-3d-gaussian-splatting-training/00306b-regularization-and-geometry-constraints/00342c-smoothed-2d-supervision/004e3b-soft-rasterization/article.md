# Soft Rasterization

Also called: Blurring Primitives, SAR Feature Rasterizer, Soft Rasterization for Gradient Enrichment

## Overview
Soft rasterization is a differentiable rendering technique that mitigates the discontinuities inherent in standard rasterization by blending facet contributions for pixels near primitive edges [S1]. It solves the problem of sparse and vanishing gradients caused by the non-uniformity of bilinear interpolation operands, which typically have similar values except at object boundaries [S2, unofficial]. Unlike standard hard rasterization, it yields slightly distorted images but allows backpropagating more useful gradients, thereby stabilizing optimization and preventing gradient vanishing at edges [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D primitive geometry (e.g., triangles or bitmap primitives), camera parameters (view and projection matrices), and a blending radius parameter $\sigma$.
*   **Outputs:** A rendered image (or feature map) with softened edges and enriched gradients suitable for backpropagation.

**Data Structures:**
The method relies on standard graphics pipeline data structures but modifies the rasterization stage.
*   **Coordinate Space:** The pipeline transforms coordinates through specific spaces depending on the application (e.g., SAR vs. standard EO). For SAR, the sequence is World Space $\rightarrow$ View Space $\rightarrow$ Range Space $\rightarrow$ Ground Space $\rightarrow$ NDC Space [S1]. For standard rendering, World Space $\rightarrow$ View Space $\rightarrow$ NDC Space is typical.
*   **Fragment Buffer:** Stores per-pixel attributes (color, alpha, normals) derived from the rasterized primitives.

**Numbered Steps:**
1.  **Coordinate Transformation:** Transform primitive vertices from World Space to View Space, then to NDC (Normalized Device Coordinates) [S1].
    *   *Note:* In SAR-specific implementations, intermediate steps include a Range Transform (z becomes y) and Ground-plane Projection (stretching y-dimension) [S1].
2.  **Soft Rasterization (Edge Blending):**
    *   For each pixel in the target image, determine the contribution of all overlapping primitives.
    *   Instead of a binary "inside/outside" test, compute a soft coverage value based on the distance from the pixel to the primitive edge.
    *   The blending is controlled by a radius parameter $\sigma$. Pixels near the edge receive blended contributions from both the primitive and the background (or adjacent primitives) [S1].
    *   *Implementation Detail:* This is often achieved by modifying the rasterizer's edge function to output a continuous coverage value rather than a binary mask [S1].
3.  **Shading (Feature Computation):**
    *   Compute per-pixel features based on the rasterized fragments.
    *   **Silhouette:** Create a mask from fragments that have no corresponding facets (background) [S1].
    *   **Surface Normals:** Compute per-pixel surface normals $\hat{n}$ relative to the illumination direction $\hat{q}$. For monostatic SAR, the normal feature $d$ is calculated as:
        $$ d = -\hat{q} \cdot \hat{n} $$
        where $\cdot$ denotes the inner product [S1].
    *   **Alpha Channel:** Determine visibility. In SAR contexts, this may involve rasterizing a reference view from the radar perspective to mask out self-shadowed facets [S1].
4.  **Loss Computation:** Compute the loss between the rendered image and the ground truth. The gradients flow through the soft rasterization stage, providing non-zero gradients for primitive parameters (position, scale, rotation) even when the primitive is not strictly "inside" the pixel in a hard-rasterization sense [S2, unofficial].

**Parameters:**
*   **Blending Radius ($\sigma$):** Controls the level of soft rasterization distortion. Typical values used in training are $0$ (hard rasterization), $10^{-5}$, $10^{-4}$, and $10^{-3}$ [S1].
*   **Gaussian Blur Kernel:** In some implementations (e.g., DiffBMP), Gaussian blur is applied to the primitives *before* optimization to enrich gradients [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_rasterize(primitives, camera, sigma):
    """
    Renders primitives using soft rasterization with blending radius sigma.
    
    Args:
        primitives: List of primitive objects (e.g., triangles or bitmaps) with attributes 
                    (position, scale, rotation, color, alpha).
        camera: Camera object with view and projection matrices.
        sigma: Blending radius for soft edges.
        
    Returns:
        rendered_image: Tensor of shape (H, W, C).
        gradients: Gradients w.r.t primitive parameters.
    """
    # 1. Coordinate Transformation
    # Transform vertices to NDC space
    ndc_vertices = []
    for prim in primitives:
        # Standard pipeline: World -> View -> NDC
        # For SAR: World -> View -> Range -> Ground -> NDC
        view_vertices = apply_view_transform(prim.vertices, camera.view_matrix)
        ndc_vertices = apply_projection_transform(view_vertices, camera.projection_matrix)
        prim.ndc_vertices = ndc_vertices

    # 2. Soft Rasterization
    # Initialize output buffer
    H, W = camera.resolution
    rendered_image = torch.zeros((H, W, 3), device=primitives[0].color.device)
    
    # For each pixel, compute soft coverage
    for y in range(H):
        for x in range(W):
            pixel_coord = (x, y)
            total_color = torch.zeros(3)
            total_alpha = 0.0
            
            # Sort primitives by depth (z-buffer) for correct blending
            sorted_prims = sort_primitives_by_depth(primitives, pixel_coord)
            
            for prim in sorted_prims:
                # Compute soft coverage based on distance to edge
                # This replaces the binary inside/outside test
                coverage = compute_soft_coverage(prim, pixel_coord, sigma)
                
                if coverage > 0:
                    # Interpolate primitive attributes at pixel location
                    color = interpolate_attributes(prim, pixel_coord)
                    alpha = prim.alpha
                    
                    # Blend with current buffer (over operator)
                    # Note: Soft rasterization modifies the alpha/coverage term
                    blended_alpha = coverage * alpha
                    total_color = total_color * (1 - blended_alpha) + color * blended_alpha
                    total_alpha += blended_alpha * (1 - total_alpha) # Approximate over
            
            rendered_image[y, x] = total_color

    # 3. Backpropagation
    # The soft coverage function is differentiable w.r.t primitive parameters
    # Gradients flow through compute_soft_coverage
    loss = compute_loss(rendered_image, ground_truth)
    loss.backward()
    
    return rendered_image
```

**Edge Cases and Pitfalls:**
*   **Artifacts:** Soft rasterization yields slightly distorted images, particularly near edges, due to the blending [S1]. These artifacts must be tolerated as they enable gradient flow.
*   **Gradient Sparsity:** Vanilla rasterization produces sparse gradients because bilinear interpolation operands have similar values except at boundaries. Soft rasterization mitigates this by smoothing edges [S2, unofficial].
*   **Precision:** The choice of $\sigma$ is critical. Too small, and gradients remain sparse; too large, and image quality degrades significantly [S1].

## Operating characteristics
*   **Time Complexity:** The soft rasterization step adds minimal computational overhead compared to standard rasterization, as it primarily modifies the edge evaluation logic [S2, unofficial]. The overhead is reported to be almost the same as standard rendering in ablation studies [S2, unofficial].
*   **Memory Complexity:** No significant additional memory is required beyond standard frame buffers and primitive attributes [S2, unofficial].
*   **Performance:** Soft rasterization yields consistent PSNR gains. In ablation studies, enabling soft rasterization improved PSNR from 24.4 to 24.7 on 512×512 targets and from 25.5 to 25.7 on 1024×1024 targets [S2, unofficial].
*   **Strengths:** Stabilizes optimization, prevents gradient vanishing at edges, and enriches gradients for better convergence [S2, unofficial].
*   **Limitations:** Introduces slight image distortion/artifacts [S1]. Not suitable for applications requiring pixel-perfect geometric accuracy without post-processing.

## Background
Soft rasterization was introduced in the context of differentiable rendering to address the non-differentiability of standard rasterization operators [S1]. It is based on the soft rasterizer described in [25] (likely Kerbl et al. or similar differentiable rasterizer literature, though the specific citation number [25] refers to a source in the SAR paper). The technique has been adapted for various domains, including Synthetic Aperture Radar (SAR) imagery [S1] and bitmap primitive rendering (DiffBMP) [S2, unofficial]. In SAR, it is part of the SAR Feature Rasterizer pipeline [S1]. In DiffBMP, it is used as an optimization helper to align sparse gradients [S2, unofficial].

## Sources
[S1] Differentiable Rendering for Synthetic Aperture Radar Imagery, https://doi.org/10.48550/arxiv.2204.01248, fetched 2026-10-26
[S2] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
