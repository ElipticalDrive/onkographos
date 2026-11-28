# Masking with Gaussian Transition

Also called: Focus+Context Masking, Mask lens

## Overview
Masking with Gaussian Transition provides a focus+context interaction technique that smoothly fades the visibility of specific volume regions or parameters from a center point outward, rather than using hard clipping boundaries [S1]. It solves the problem of abrupt visual discontinuities in volume rendering by applying a Gaussian weighting function to parameter weights (such as opacity or color) within a defined masking region [S1]. Unlike [Clipping](1444-clipping.md) which removes data entirely, this method preserves context by gradually reducing the influence of masked elements [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** User-defined mask center/region parameters, volume data, and current rendering parameters (opacity, color, etc.).
*   **Output:** A rendered image where the masked region exhibits a smooth transition from full visibility to full transparency (or parameter suppression).

**Data Structures:**
*   **G-Buffer (Geometric Buffer):** A 2.5D data structure storing per-pixel view-dependent information required for blending. Fields typically include:
    *   Rendering Color (RGB)
    *   Depth
    *   Opacity/Transparency
    *   Virtual Thickness (optional, for depth-based blending)
    *   Normal Vectors (optional)
    *   Object ID (optional)
    *   Texture Coordinates (optional) [S1]

**Method Steps:**
1.  **Define Mask Region:** Identify the center and extent of the masking region in the view space.
2.  **Calculate Gaussian Weight:** For each pixel or sample point within the masking region, calculate a weight $w$ based on its distance $d$ from the mask center $c$. The weight follows a Gaussian function:
    $$ w(d) = e^{-\frac{d^2}{2\sigma^2}} $$
    Where $\sigma$ controls the spread of the transition [S1]. The weight is 1.0 at the center ($d=0$) and approaches 0.0 at the edges.
3.  **Apply Weight to Parameters:** Multiply the original rendering parameters (e.g., opacity $\alpha$, color $C$) by the calculated weight $w$.
    $$ \alpha_{masked} = \alpha_{original} \times w $$
    $$ C_{masked} = C_{original} \times w $$
    This effectively fades the parameters from the center to the edge of the masking region [S1].
4.  **Render and Blend:** Render the scene using the weighted parameters. The G-Buffer stores these intermediate results. Final image composition uses 3D blending techniques (e.g., depth-based blending) to composite the masked and unmasked regions into a single view [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_gaussian_mask(rendering_params, mask_center, mask_sigma, pixel_position):
    """
    Applies Gaussian masking to rendering parameters.
    
    Args:
        rendering_params: Tuple of (opacity, color) for the current pixel.
        mask_center: 2D or 3D coordinates of the mask center.
        mask_sigma: Standard deviation for the Gaussian falloff.
        pixel_position: 2D or 3D coordinates of the current pixel/sample.
        
    Returns:
        Tuple of (masked_opacity, masked_color).
    """
    # Calculate squared distance from mask center
    dist_sq = sum((p - c)**2 for p, c in zip(pixel_position, mask_center))
    
    # Calculate Gaussian weight
    # w = exp(-dist^2 / (2 * sigma^2))
    weight = math.exp(-dist_sq / (2 * mask_sigma**2))
    
    # Apply weight to parameters
    original_opacity, original_color = rendering_params
    masked_opacity = original_opacity * weight
    masked_color = original_color * weight
    
    return masked_opacity, masked_color
```

**Parameters:**
*   `mask_sigma`: Controls the smoothness of the transition. Larger values create a wider, softer fade. [S1]
*   `mask_center`: The focal point of the mask. [S1]

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure floating-point precision is sufficient for small distances to avoid numerical instability in the exponential function [G].
*   **Boundaries:** Pixels outside the mask region should retain their original parameters (weight = 1.0) or be handled by a separate unmasked rendering pass [S1].
*   **Degenerate Input:** If `mask_sigma` is 0, the weight becomes a Dirac delta function, potentially causing artifacts. A minimum non-zero value should be enforced [G].

## Subtopics

### Defocus-aware Mask Generation
This variant generates masks based on defocus cues (Circle of Confusion - CoC) rather than explicit user-defined centers. It simulates defocus by rendering with specific focal length (`valid_F=1.0`) and kernel size (`valid_K=0.0`) parameters to create a fake defocused view [S2, unofficial]. The mask is computed by a neural network (`mask_net`) that takes the rendered image, normalized depth, and normalized CoC alpha as input [S2, unofficial]. The network outputs an `outfocus_mask`, and the `infocus_mask` is derived as `1.0 - outfocus_mask` [S2, unofficial]. This allows for automatic focus+context separation based on depth-of-field effects [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels/samples in the masking region. The Gaussian calculation is constant time per pixel.
*   **Memory Complexity:** $O(N)$ for storing G-Buffer data if used for intermediate blending [S1].
*   **Strengths:** Provides visually smooth transitions, preserving context without hard edges. Effective for highlighting specific regions while maintaining spatial awareness [S1].
*   **Limitations:** Requires additional rendering passes or G-Buffer storage if complex blending is needed. Performance impact depends on the number of masked regions and the complexity of the blending technique [S1].
*   **When to use:** When smooth visual transitions are preferred over hard clipping, especially for focus+context visualization [S1].
*   **When not to use:** When precise, hard boundaries are required for measurement or analysis tasks [G].

## Background
The technique is described in the context of "Ghosted Illustration Rendering" which uses depth-based blending and G-buffers to create smooth visibility transitions [S1]. It is part of a broader family of focus+context visualization methods that aim to reduce visual clutter while maintaining scene context [S1]. The specific implementation details regarding G-buffers and Gaussian weighting are attributed to this work [S1].

## Sources
[S1] Ghosted Illustration Rendering using Depth-based Blending Techniques, https://doi.org/10.15701/kcgs.2020.26.3.1, fetched 2026-10-29
[S2] rongduo/DOFGS_implementation, https://github.com/rongduo/DOFGS_implementation, fetched 2026-10-27 (unofficial)
