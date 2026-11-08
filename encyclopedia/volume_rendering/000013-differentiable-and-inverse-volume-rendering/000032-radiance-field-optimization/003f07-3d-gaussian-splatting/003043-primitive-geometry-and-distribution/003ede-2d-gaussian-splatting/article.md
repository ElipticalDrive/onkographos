# 2D Gaussian Splatting

Also called: 2D-GS, Surfel Splatting

## Overview
2D Gaussian Splatting (2DGS) is a surface reconstruction and novel view synthesis technique that replaces the volumetric 3D Gaussian primitives of [3D Gaussian Splatting](6228-3d-gaussian-splatting.md) with oriented 2D disks (surfels) [S1, S3][S4, S5, unofficial]. It solves the problem of poor spatial localization and inaccurate geometry extraction inherent in 3D Gaussians, which are symmetric and unbounded [S3][S5, unofficial]. By aligning primitives with local surface tangent planes, 2DGS enables high-resolution normal map recovery and accurate mesh extraction [S1][S4, S5, unofficial]. It sets itself apart from [3D Gaussian Splatting](6228-3d-gaussian-splatting.md) by using surface-centric 2D disks rather than 3D ellipsoids, and from [Flattened Gaussian Ellipsoid Rendering](1667-flattened-gaussian-ellipsoid-rendering.md) by natively supporting anisotropic ellipsoidal kernels with near-zero determinant covariance within a differentiable volume rendering framework [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of calibrated multi-view images, initial camera poses, and an initial point cloud (typically from Structure-from-Motion) [S1, S2][S5, unofficial].
- **Outputs:** A set of optimized 2D Gaussian primitives, rendered novel views, and a reconstructed mesh with normal maps [S1][S5, unofficial].

**Data Structures:**
Each primitive $i$ is defined by:
1. **Mean Position** ($\boldsymbol{\mu}_i$): A 3D vector $(x, y, z)$ representing the center of the disk in world space [G].
2. **Covariance Matrix** ($\boldsymbol{\Sigma}_i$): A $3 \times 3$ symmetric positive-definite matrix. For 2DGS, this matrix is constrained to have a near-zero determinant, effectively flattening the 3D ellipsoid into a 2D disk [S1]. It is parameterized by a rotation matrix $\mathbf{R}_i$ (from quaternion $\mathbf{q}_i$) and a scaling vector $\mathbf{s}_i = (s_{i,x}, s_{i,y}, s_{i,z})$ where one scale component (typically the normal direction) is constrained to be near zero [S1, G].
   $$ \boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T $$
   where $\mathbf{S}_i = \text{diag}(s_{i,x}, s_{i,y}, s_{i,z})$ [G].
3. **Opacity** ($\alpha_i$): A scalar value in $[0, 1]$ [G].
4. **Color/Appearance**: Represented by spherical harmonics (SH) coefficients or a base color vector [G].

**Rendering Steps:**
1. **Projection:** For each camera, transform the 3D covariance $\boldsymbol{\Sigma}_i$ into screen space using the Jacobian of the projection transformation $\mathbf{J}$ and the view rotation matrix $\mathbf{W}$:
   $$ \boldsymbol{\Sigma}'_i = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma}_i \mathbf{W}^T \mathbf{J}^T $$
   [G]
2. **Degeneracy Handling:** Apply a low-pass filter to the projected 2D covariance to avoid degenerated line projections when the disk is viewed edge-on [S2].
3. **Rasterization:** Project the 2D Gaussian onto the image plane. The 2D Gaussian density is evaluated at each pixel covered by its bounding box [G].
4. **Compositing:** Sort primitives by depth and compute the final color $C$ via alpha compositing:
   $$ C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
   where $c_i$ is the color of the $i$-th primitive at the pixel location, and $\alpha_i$ is the accumulated opacity derived from the 2D Gaussian density function [G].

**Optimization:**
The parameters ($\boldsymbol{\mu}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \text{SH coefficients}$) are optimized using gradient descent. Gradients flow through the rasterization process back to the mean, covariance, and opacity parameters [G]. The covariance matrix is updated by optimizing the scale and rotation parameters directly to preserve positive definiteness and the near-zero determinant constraint [G, S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_2d_gaussians(primitives, camera):
    """
    Renders a scene using 2D Gaussian Splatting.
    primitives: List of primitives, each with mean (mu), rotation (q), scale (s), opacity (alpha), color (c).
    camera: Camera object with projection matrix and view matrix.
    """
    image_height, image_width = camera.resolution
    output_image = np.zeros((image_height, image_width, 3))
    output_alpha = np.zeros((image_height, image_width))
    
    # 1. Project primitives to screen space
    screen_primitives = []
    for prim in primitives:
        # Transform covariance to screen space
        # Sigma' = J * W * Sigma * W^T * J^T
        # J is Jacobian of projection, W is view rotation
        sigma_3d = compute_covariance(prim.q, prim.s)
        sigma_2d = project_covariance(sigma_3d, camera)
        
        # Apply low-pass filter to avoid degenerate line projection [S2]
        sigma_2d = apply_low_pass_filter(sigma_2d)
        
        # Compute bounding box and center
        center_2d = project_point(prim.mu, camera)
        bounding_box = compute_bounding_box(center_2d, sigma_2d)
        
        screen_primitives.append({
            'center': center_2d,
            'covariance': sigma_2d,
            'alpha': prim.alpha,
            'color': prim.c,
            'bounding_box': bounding_box
        })
    
    # 2. Sort primitives by depth (z-coordinate)
    screen_primitives.sort(key=lambda p: p['center'][2])
    
    # 3. Rasterize and composite
    for prim in screen_primitives:
        center = prim['center']
        cov = prim['covariance']
        alpha = prim['alpha']
        color = prim['color']
        bbox = prim['bounding_box']
        
        # Iterate over pixels in bounding box
        for y in range(bbox.y_min, bbox.y_max):
            for x in range(bbox.x_min, bbox.x_max):
                if y < 0 or y >= image_height or x < 0 or x >= image_width:
                    continue
                
                # Evaluate 2D Gaussian density at pixel (x, y)
                dist = np.array([x - center[0], y - center[1]])
                density = np.exp(-0.5 * dist.T @ np.linalg.inv(cov) @ dist)
                
                # Accumulate color and alpha
                # C = sum(c_i * alpha_i * prod(1 - alpha_j))
                # Alpha_i here is density * alpha (intrinsic)
                pixel_alpha = density * alpha
                
                # Pre-multiplied alpha compositing
                # New color = old_color * (1 - pixel_alpha) + color * pixel_alpha
                # Note: Standard alpha compositing assumes pre-multiplied colors or handles alpha accumulation
                # Here we follow the standard equation: C = sum(c_i * alpha_i * prod(1 - alpha_j))
                
                # To implement efficiently, we can use the standard over operator
                # C_out = C_in * (1 - alpha) + C_prim * alpha
                # But the equation in the prompt is: C = sum(c_i * alpha_i * prod_{j=1}^{i-1} (1 - alpha_j))
                # This implies we need to track the accumulated opacity product.
                
                # Let's implement the standard alpha compositing loop
                # C_final = C_current * (1 - p_alpha) + c_prim * p_alpha
                # where p_alpha = density * alpha
                
                p_alpha = density * alpha
                if p_alpha > 0:
                    output_image[y, x] = output_image[y, x] * (1 - p_alpha) + color * p_alpha
                    output_alpha[y, x] = output_alpha[y, x] * (1 - p_alpha) + p_alpha
                    
    return output_image, output_alpha

def compute_covariance(q, s):
    """Compute 3D covariance from quaternion and scale."""
    R = quaternion_to_rotation_matrix(q)
    S = np.diag(s)
    return R @ S @ S.T @ R.T

def project_covariance(sigma_3d, camera):
    """Project 3D covariance to 2D screen space."""
    # J is Jacobian of projection
    # W is view rotation
    J = camera.get_projection_jacobian()
    W = camera.get_view_rotation()
    return J @ W @ sigma_3d @ W.T @ J.T

def apply_low_pass_filter(sigma_2d):
    """Apply low-pass filter to avoid degenerate line projection [S2]."""
    # Add a small epsilon to the diagonal to prevent singularity
    epsilon = 1e-3
    return sigma_2d + epsilon * np.eye(2)
```

## Subtopics

### [Flattened Gaussian Ellipsoid Rendering](1667-flattened-gaussian-ellipsoid-rendering.md)
Approximates 2D Gaussian ellipses by flattening 3D Gaussian ellipsoids. It differs from 2DGS by operating within a differentiable volume rendering framework that natively supports anisotropic ellipsoidal kernels and enforces a near-zero determinant covariance.

### [3D Surface Splatting](4524-3d-surface-splatting.md)
A differentiable surface splatting renderer for physically-based inverse rendering. It sets itself apart from 2DGS by using a surface rendering model rather than a volumetric one (surfels carry no learnable opacity) and handles multiple surface layers per pixel through depth-interval merging.

### [Spatially Varying Gaussian Splatting (SVGS)](8729-spatially-varying-gaussian-splatting-svgs.md)
Utilizes spatially varying colors and opacity within a single Gaussian primitive. It sets itself apart from 2DGS by defining color and opacity as spatially varying functions of the intersection point $p$ on the Gaussian primitive, rather than constant values per primitive.

### [Learning View-Dependent Splatting Kernels](9548-learning-view-dependent-splatting-kernels.md)
A differentiable framework that automatically learns view-dependent 2D splatting kernels. It sets itself apart from 2DGS by introducing learned view-dependency to serve as a form of data-driven consistency, allowing the kernel profile to vary with the camera view.

### [4D Scale-adaptive Filter](9868-4d-scale-adaptive-filter.md)
A post-processing and regularization technique for 4D Gaussian Splatting. It distinguishes itself from fixed-scale smoothing filters by adaptively adjusting the dilation coefficient based on the primitive's current scale and visibility.

## Operating characteristics
- **Time Complexity:** Rendering is $O(N)$ per pixel, where $N$ is the number of primitives covering the pixel, similar to 3DGS. Optimization complexity depends on the number of primitives and views.
- **Memory Complexity:** $O(N)$, where $N$ is the number of primitives. 2DGS may require more primitives than 3DGS to achieve similar visual quality in some cases, but fewer for surface reconstruction tasks [S3][S4, unofficial].
- **Measured Performance:** 2DGS achieves high-quality novel view synthesis and significantly improves geometry and mesh extraction compared to 3DGS [S4, S5, unofficial]. It recovers high-resolution normal maps [S5, unofficial].
- **Strengths:** Accurate spatial localization, high-resolution normal map recovery, accurate mesh extraction, better modeling of hard surfaces and sharp edges [S3][S4, S5, unofficial].
- **Limitations:** May require more primitives for complex volumetric scenes compared to 3DGS. The near-zero determinant constraint can lead to numerical instability if not handled properly (e.g., via low-pass filtering) [S2, S1].
- **When to use:** When surface reconstruction, mesh extraction, or normal map recovery is required. When high-frequency geometric details and sharp edges are important.
- **When not to use:** When the scene is highly volumetric (e.g., smoke, fire) and surface representation is not the primary goal. When computational resources are extremely limited and 3DGS is sufficient.

## Background
2D Gaussian Splatting (2DGS) was proposed by Huang et al. at SIGGRAPH 2024 [S4, S5, unofficial]. It was introduced to address the limitations of 3D Gaussian Splatting in surface reconstruction, specifically its poor spatial localization and inability to accurately extract meshes [S1][S5, unofficial]. The method builds on the differentiable rasterization framework of 3DGS but replaces the 3D Gaussian primitives with oriented 2D disks (surfels) [S1, S3][S4, S5, unofficial]. This surface-centric approach allows the primitives to align with local surface tangent planes, enabling better capture of high-frequency details and reduction of artifacts [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
