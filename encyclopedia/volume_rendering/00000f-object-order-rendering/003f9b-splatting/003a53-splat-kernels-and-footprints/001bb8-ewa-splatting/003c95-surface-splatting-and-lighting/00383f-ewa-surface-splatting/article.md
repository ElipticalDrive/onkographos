# EWA Surface Splatting

Also called: EWA Volume Splatting, Elliptical Weighted Average splatting, Zwicker EWA Volume Splatting

## Overview
EWA Surface Splatting is a high-quality point rendering algorithm that renders point samples by superimposing band-limited reconstruction kernels, called surface splats, in the image domain [S1]. It efficiently suppresses aliasing artifacts and guarantees hole-free reconstructions even for moderate sampling densities and under arbitrary transformations [S1]. Unlike simple forward-projection of pure point-samples which causes holes if sampling density is insufficient, EWA splatting uses splats with spatial extent to ensure hole-free results [S1]. It builds on [Surface Splatting and Lighting](9275-surface-splatting-and-lighting.md) by flattening the 3D Gaussian kernel along the volume gradient to reduce EWA volume splats to surface splats suitable for high-quality iso-surface rendering [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of point samples $P = \{p_i\}$, where each sample $p_i$ has a position $\mathbf{p}$, a color $\mathbf{c}$, and a normal $\mathbf{n}$ [S2, unofficial]. Each sample is augmented with two tangent axes $(u, v)$ and corresponding radii, representing disks or ellipses in object space [S1].
*   **Output:** A rasterized image with accumulated color and attributes, free of holes and aliasing artifacts [S1].

**Data Structures:**
Each splat $S_j$ is defined by:
*   Center $c_j$ (position $\mathbf{p}$)
*   Tangent vectors $u_j, v_j$
*   Radii $r_u, r_v$
*   Attributes: Color $\mathbf{c}$, Normal $\mathbf{n}$ [S2, unofficial].

**Steps:**
1.  **Object-Space Preparation:** For each point in object-space, define a quadrilateral texture-mapped with a Gaussian texture [S2, unofficial]. The splats in object space must have mutual overlap to guarantee watertight rendering [S1].
2.  **Deformation:** Deform the object-space quadrilaterals to result in the correct screen-space EWA splat after projection [S2, unofficial]. This involves computing the screen-space footprint, which is analytically defined as a single 2D Gaussian footprint that integrates an elliptical 3D Gaussian reconstruction kernel and a 2D Gaussian low-pass filter [S2, unofficial].
3.  **Projection:** Transform the deformed screen-space quadrilateral into screen coordinates [S2, unofficial].
4.  **Rasterization:** Rasterize the screen-space footprint. The footprint acts as a low-pass filter to suppress aliasing [S1][S2, unofficial].
5.  **Accumulation:** Blend the contributions of overlapping splats. Compute a weight $w$ based on the distance from the splat center and accumulate attributes (color, normal, depth) additively [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ewa_surface_splatting(points, view_matrix, proj_matrix, screen_width, screen_height):
    """
    Renders point samples using EWA Surface Splatting.
    points: List of dicts with keys 'p' (vec3), 'n' (vec3), 'c' (vec3), 'u' (vec3), 'v' (vec3), 'ru', 'rv'
    """
    image = zeros((screen_height, screen_width, 3))
    weight_image = zeros((screen_height, screen_width))
    
    for p in points:
        # 1. Object-space quadrilateral definition
        # Define quad vertices in object space based on tangent axes and radii
        # This quad is texture-mapped with a Gaussian kernel
        quad_obj = define_quad(p['p'], p['u'], p['v'], p['ru'], p['rv'])
        
        # 2. Deformation to screen space
        # Deform the quad to account for perspective projection and EWA footprint
        quad_screen = deform_quad_for_ewa(quad_obj, view_matrix, proj_matrix, p['n'])
        
        # 3. Rasterization and Accumulation
        # Rasterize the screen-space quad
        for pixel in rasterize(quad_screen):
            # Compute weight based on 2D Gaussian footprint
            # The footprint integrates the 3D Gaussian kernel flattened along the volume gradient
            w = gaussian_weight(pixel['uv'], quad_screen['center'], quad_screen['covariance'])
            
            # Accumulate color and weight
            image[pixel['x'], pixel['y']] += p['c'] * w
            weight_image[pixel['x'], pixel['y']] += w
            
    # Normalize accumulated color by weight
    for y in range(screen_height):
        for x in range(screen_width):
            if weight_image[y, x] > 0:
                image[y, x] /= weight_image[y, x]
                
    return image

def define_quad(center, u, v, ru, rv):
    """
    Defines a quadrilateral in object space centered at 'center'
    with tangent axes 'u' and 'v' and radii 'ru' and 'rv'.
    """
    # Vertices are at center +/- ru*u and center +/- rv*v
    # This forms an ellipse-like disk in object space
    v0 = center - ru * u - rv * v
    v1 = center + ru * u - rv * v
    v2 = center + ru * u + rv * v
    v3 = center - ru * u + rv * v
    return [v0, v1, v2, v3]

def deform_quad_for_ewa(quad_obj, view_matrix, proj_matrix, normal):
    """
    Deforms the object-space quad to screen space for EWA splatting.
    This step analytically defines the screen-space footprint as a 2D Gaussian.
    """
    # Transform vertices to clip space
    clip_vertices = [transform(proj_matrix * view_matrix * v) for v in quad_obj]
    
    # Perform perspective division to get NDC
    ndc_vertices = [v / v.w for v in clip_vertices]
    
    # Map to screen space
    screen_vertices = [ndc_to_screen(v) for v in ndc_vertices]
    
    # Compute the EWA covariance matrix in screen space
    # This involves the Jacobian of the projection and the flattening of the 3D Gaussian
    # along the volume gradient (normal)
    cov = compute_ewa_covariance(screen_vertices, normal, proj_matrix, view_matrix)
    
    return {
        'vertices': screen_vertices,
        'center': screen_vertices[0], # Approximate center
        'covariance': cov
    }

def gaussian_weight(uv, center, covariance):
    """
    Computes the weight of a pixel at 'uv' based on the 2D Gaussian defined by 'center' and 'covariance'.
    """
    diff = uv - center
    # Weight = exp(-0.5 * diff^T * Sigma^-1 * diff)
    inv_cov = inverse(covariance)
    exponent = -0.5 * dot(diff, matmul(inv_cov, diff))
    return exp(exponent)
```

**Parameters:**
*   **Sampling Density:** Must be sufficient to ensure mutual overlap of surface splats in object space [S1]. Not given in the sources.
*   **Gaussian Kernel Width:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Insufficient Sampling Density:** Simple forward-projection of pure point-samples causes holes. EWA splatting mitigates this by using splats with spatial extent [S1].
*   **Arbitrary Transformations:** The method guarantees hole-free reconstructions even under arbitrary transformations [S1].

## Subtopics

### Deferred Surface Splatting
A variant that computes illumination in a final screen-space pass. [Deferred Surface Splatting](1072-deferred-surface-splatting.md)

### Phong Splatting
A variant that computes illumination per-splat using Phong shading. [Phong Splatting](2802-phong-splatting.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** High-quality rendering, suppression of aliasing artifacts, hole-free reconstructions, handles arbitrary transformations [S1].
*   **Limitations:** Requires sufficient sampling density to ensure overlap, though less strict than pure point sampling [S1].
*   **When to use:** High-quality direct visualization of point-sampled surfaces [S1].
*   **When not to use:** When sampling density is extremely low and overlap cannot be guaranteed [S1].

## Background
EWA Surface Splatting was introduced by Zwicker et al. in 2001 [S1]. Perspective accurate variants were introduced starting with Zwicker et al. 2004 and Botsch et al. 2004 [S1]. It was designed for high-quality direct visualization of point-sampled surfaces [S1]. The method combines the advantages of triangles and point-based representations [S1].

## Sources
[S1] A hardware architecture for surface splatting, https://doi.org/10.1145/1275808.1276490, fetched 2026-10-28
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
