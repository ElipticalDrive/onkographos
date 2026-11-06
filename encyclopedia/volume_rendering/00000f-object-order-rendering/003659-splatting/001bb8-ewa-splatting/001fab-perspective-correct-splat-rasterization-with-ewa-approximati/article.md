# Perspective Correct Splat Rasterization with EWA Approximation

Also called: Affine approximation to projective mapping, Approximate EWA splatting, EWA point splatting, Fast splat rasterization, Perspective Accurate Splatting with EWA Filter

## Overview
This method is an improved splat rendering pipeline that combines a perspectively correct fast splat rasterization procedure with an approximation of the Elliptical Weighted Average (EWA) filter [S1][S2, unofficial]. It solves the problem of aliasing artifacts and visual holes in point-sampled geometry rendering, particularly when the model is viewed from flat angles or shifted from the viewing axis [S2, unofficial]. Unlike its siblings, it uses an affine approximation to the projective mapping to ensure perspective-correct splat shapes without requiring per-point normals or complex anisotropic footprint calculations, trading exact band-limited footprints for a formulation that avoids holes and Moiré patterns [S2, unofficial]. It builds on [EWA Splatting](1069-ewa-splatting.md) by replacing the standard EWA covariance computation with a faster, perspective-correct affine approximation.

## How it works
The method operates in a multi-pass pipeline, typically involving a visibility pass, an attribute pass, and a normalization pass. It relies on an affine approximation of the projective mapping to compute the splat's screen-space shape.

### Inputs and Outputs
*   **Inputs:** Point-sampled geometry (positions, attributes), camera projection matrix, view matrix, window resolution.
*   **Outputs:** Final composited image with alpha blending.

### Data Structures
*   **Lookup Table (LUT):** A 1D texture storing Gaussian filter weights. The index is determined by the squared distance $r^2$ from the splat center in screen space, normalized by the filter width.
*   **Conic Matrix:** A $2 \times 2$ matrix representing the inverse of the resampling filter's variance in screen space.

### Numbered Steps

1.  **Vertex Processing (Visibility and Shape Approximation):**
    For each splat $k$:
    a.  Compute the projective mapping matrix $M_k$ that maps the local Gaussian reconstruction kernel to screen space.
    b.  Compute the inverse $M_k^{-1}$ to derive the approximate variance matrix $Q_{000}$ [S2, unofficial].
    c.  **Numerical Stability Check:** Calculate the condition number of $M_k$. If it exceeds a threshold (indicating the splat is degenerating into a line segment or is numerically ill-conditioned), discard the splat [S2, unofficial].
    d.  **Ellipse Check:** Determine if the resulting conic $Q_{000}$ represents an ellipse using specific criteria (e.g., determinant sign and trace properties) [S2, unofficial]. If it is not an ellipse, discard the splat [S2, unofficial].
    e.  **Bounding Box:** Compute the bounding box for the OpenGL point size based on the ellipse extent [S2, unofficial].
    f.  Pass the $2 \times 2$ conic matrix $(Q_{000} + H)^{-1}$ to the fragment program, where $H$ is the Gaussian prefilter variance [S2, unofficial].

2.  **Fragment Processing (EWA Filtering):**
    For every pixel $x$ covered by the OpenGL point primitive:
    a.  Evaluate the ellipse equation:
        $$ r^2 = x (Q_{000} + H)^{-1} x^T $$
        where $x$ is the pixel coordinate vector relative to the splat center, available via the `wpos` input parameter [S2, unofficial].
    b.  If $r^2 < F_g^2$ (where $F_g$ is the Gaussian filter width threshold), use $r^2$ as an index into the 1D LUT to retrieve the Gaussian filter weight [S2, unofficial].
    c.  Otherwise, discard the fragment [S2, unofficial].
    d.  Output the weighted attribute value and alpha.

3.  **Normalization Pass:**
    a.  Copy the result of the attribute pass (weighted sum of attributes and alpha) to a texture.
    b.  Render a window-sized rectangle using this texture.
    c.  In the fragment program, perform the division by the accumulated alpha to normalize the color values [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Program
def vertex_program(splat_position, projection_matrix, view_matrix):
    # 1. Compute projective mapping matrix Mk
    Mk = compute_projective_mapping(splat_position, projection_matrix, view_matrix)
    
    # 2. Invert Mk to get Q000
    Mk_inv = invert_2x2_matrix(Mk)
    Q000 = compute_variance_from_inverse(Mk_inv)
    
    # 3. Check condition number for numerical stability
    cond_number = condition_number(Mk)
    if cond_number > THRESHOLD:
        return discard_splat()
        
    # 4. Check if conic is an ellipse
    if not is_ellipse(Q000):
        return discard_splat()
        
    # 5. Compute bounding box for OpenGL point size
    point_size = compute_bounding_box_size(Q000)
    
    # 6. Compute final conic matrix for fragment shader
    # H is the Gaussian prefilter variance (constant or per-splat)
    H = get_gaussian_prefilter_variance()
    conic_matrix = invert_2x2_matrix(Q000 + H)
    
    return {
        'point_size': point_size,
        'conic_matrix': conic_matrix,
        'splat_center': project_to_screen(splat_position)
    }

# Fragment Program
def fragment_program(conic_matrix, splat_center, wpos):
    # wpos is the pixel coordinate
    x = wpos - splat_center
    
    # Evaluate ellipse equation r^2 = x * (Q000 + H)^-1 * x^T
    # Note: x is a row vector, so x^T is column
    r_squared = dot_product(x, multiply_matrix_vector(conic_matrix, transpose(x)))
    
    # F_g is the Gaussian filter width threshold
    if r_squared < F_g_squared:
        # Lookup Gaussian weight from 1D texture
        weight = texture_lookup_1d(gaussian_lut, r_squared)
        return {
            'color': splat_color * weight,
            'alpha': splat_alpha * weight
        }
    else:
        return discard_fragment()

# Normalization Pass
def normalization_pass(rendered_texture):
    # Render full-screen quad
    for each pixel:
        accumulated_color = texture_lookup(rendered_texture, pixel_coord).color
        accumulated_alpha = texture_lookup(rendered_texture, pixel_coord).alpha
        
        if accumulated_alpha > 0:
            final_color = accumulated_color / accumulated_alpha
        else:
            final_color = 0
        return final_color
```

### Parameters
*   **Threshold for Condition Number:** Not given in the sources.
*   **Gaussian Filter Width ($F_g$):** Not given in the sources.
*   **Prefilter Variance ($H$):** Not given in the sources.

### Edge Cases and Pitfalls
*   **Numerical Ill-Conditioning:** The inversion of $M_k$ can be numerically unstable if the splat degenerates into a line segment. This is handled by checking the condition number and discarding the splat [S2, unofficial].
*   **Non-Elliptic Conics:** The projective mapping can result in a conic that is not an ellipse. These must be detected and discarded to avoid incorrect rasterization [S2, unofficial].
*   **Splat Center Approximation:** The splat centers are not mapped to the perspective-correct position (approximation in Section 4.4 of [S2, unofficial]). However, no visual artifacts were observed due to this approximation [S2, unofficial].
*   **Clipping Overhead:** Splat clipping adds overhead. The implementation switches between two fragment shaders (clipped and non-clipped) because typically only a few splats require clipping [S2, unofficial].

## Subtopics

### Deferred Surface Splatting
Renders splats by first computing surface normals and then projecting them. It differs by explicitly handling surface orientation, whereas this method uses an affine approximation to projective mapping for perspective correctness without explicit normals. [Link](1072-deferred-surface-splatting.md)

### Opacity-Thin-Side (OTS)
Optimizes rendering for thin surfaces by adjusting opacity based on viewing angle. It differs by focusing on opacity adjustment for thin geometry, while this method focuses on perspective-correct shape and EWA filtering. [Link](1155-opacity-thin-side-ots.md)

### OTS (EWA-Based Splatting)
Combines OTS with EWA filtering. It differs by integrating opacity thin-side logic with the EWA filter, whereas this method uses a simpler affine approximation for perspective correction. [Link](1181-ots-ewa-based-splatting.md)

### GPU-Accelerated Volume Splatting With Elliptical RBFs
Uses elliptical Radial Basis Functions for volume rendering. It differs by using elliptical RBFs for volume data, while this method is for point-sampled geometry with an affine EWA approximation. [Link](1186-gpu-accelerated-volume-splatting-with-elliptical-rbfs.md)

### Ellipsoidal Kernel Splatting with Warped Slicing
Uses warped slicing to handle ellipsoidal kernels. It differs by using a slicing-based approach for ellipsoidal kernels, whereas this method uses direct splat rasterization with an affine approximation. [Link](1290-ellipsoidal-kernel-splatting-with-warped-slicing.md)

### Adaptive EWA Volume Splatting
Adapts the EWA filter based on local density. It differs by adapting the filter size, while this method uses a fixed affine approximation for perspective correctness. [Link](2137-adaptive-ewa-volume-splatting.md)

### EWA Approximation with Tangent Vector Adjustment
Adjusts tangent vectors to improve EWA approximation. It differs by adjusting tangent vectors, whereas this method uses an affine approximation to the projective mapping. [Link](2143-ewa-approximation-with-tangent-vector-adjustment.md)

### Phong Splatting
Applies Phong shading to splats. It differs by adding shading, while this method focuses on perspective-correct rasterization and EWA filtering. [Link](2802-phong-splatting.md)

### EWA Splatting Renderer (Mesh-Free)
A general EWA splatting renderer for mesh-free data. It differs by being a general renderer, while this method specifically uses an affine approximation for perspective correctness. [Link](4665-ewa-splatting-renderer-mesh-free.md)

### EWA Splatting with kNN Covariance Estimation
Uses k-NN for covariance estimation. It differs by using k-NN for covariance, whereas this method uses an affine approximation to the projective mapping. [Link](4690-ewa-splatting-with-knn-covariance-estimation.md)

## Operating characteristics
*   **Time Complexity:** The vertex program is relatively simple, while the fragment program is more complex due to the EWA filter evaluation. Compared to raycasting, the splatting approach requires simpler vertex programs but significantly more complex fragment programs [S1]. However, since the number of rasterized fragments is significantly larger than the number of splats (at least a factor of four in complete minification), the splatting approach is always more performant [S1].
*   **Memory Complexity:** Requires storage for the 1D LUT and intermediate textures for the normalization pass [S2, unofficial].
*   **Measured Performance:**
    *   On Athlon64 3500+ / GeForce 7800 with Igea dataset at 1024x1024: 154k splats at 50 fps (7.7M splats/sec) vs 33 fps for raycasting; 460k splats at 40 fps (18.4M splats/sec) vs 26 fps; 1.4M splats at 22 fps (31M splats/sec) vs 13 fps; 2.5M splats at 15 fps (37.5M splats/sec) vs 9.2 fps; 5M splats at 6.5 fps (32.5M splats/sec) vs 5 fps [S1].
    *   On GeForceFX 5950 GPU and Pentium IV 3.0 GHz CPU:
        *   Cg: 1 pass rendering achieved 2.9 million splats/sec at 512x512 and 2.8 million splats/sec at 1280x1024. The 3-pass algorithm achieved 1.4 million splats/sec at 512x512 and 1.2 million splats/sec at 1280x1024 [S2, unofficial].
        *   Assembler: 1 pass rendering achieved 11.2 million splats/sec at 512x512 and 10.4 million splats/sec at 1280x1024. The 3-pass algorithm achieved 3.1 million splats/sec at 512x512 and 2.8 million splats/sec at 1280x1024 [S2, unofficial].
*   **Strengths:** Higher quality anti-aliasing filtering for splats in both minification and magnification cases compared to previous approximations [S1]. Avoids holes in the rendered image at flat viewing angles [S2, unofficial]. Avoids Moiré patterns by using a Gaussian prefilter [S2, unofficial].
*   **Limitations:** The fragment program is more complex than the vertex program [S1]. The approximation of splat centers may lead to minor inaccuracies, though no visual artifacts were observed [S2, unofficial].
*   **When to use:** When high image quality and anti-aliasing are required for point-sampled geometry, especially with sharp features or CSG-generated objects [S2, unofficial]. When perspective-correct splat shapes are needed to avoid holes at flat viewing angles [S2, unofficial].
*   **When not to use:** When hardware does not support the required fragment program complexity or texture lookups. When real-time performance is critical and the overhead of the 3-pass algorithm is unacceptable.

## Background
The method was presented in "Perspective Accurate Splatting" [S2, unofficial] and "Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering" [S1]. It builds on the EWA splatting technique by Heckbert [S2, unofficial] and improves upon previous approximations by Botsch et al. [BHZK05] and ZRB*04 [S1]. The approach was developed to address the limitations of earlier splatting methods, such as aliasing artifacts and holes at flat viewing angles [S2, unofficial]. It is used in point-based rendering systems to render objects with sharp features created using CSG operations [S2, unofficial].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
[S2] Perspective Accurate Splatting, http://infoscience.epfl.ch/record/149345, fetched 2026-10-26 (unofficial)
