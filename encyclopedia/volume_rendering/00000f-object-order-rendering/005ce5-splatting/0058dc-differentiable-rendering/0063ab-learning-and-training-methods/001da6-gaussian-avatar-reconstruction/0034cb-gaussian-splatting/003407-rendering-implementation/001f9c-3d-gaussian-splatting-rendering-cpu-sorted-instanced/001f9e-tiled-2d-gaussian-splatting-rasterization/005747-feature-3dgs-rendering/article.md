# Feature 3DGS Rendering

Also called: Differentiable Volumetric Feature Rendering, Joint RGB and Feature Rasterization

## Overview
Feature 3DGS is a method for rendering high-dimensional semantic feature maps alongside RGB images using 3D Gaussians, enabling the distillation of knowledge from 2D foundation models (such as SAM or LSeg) into an explicit 3D scene representation [S1]. It sets itself apart from NeRF-based feature field methods by using an explicit representation that avoids interference between the radiance and feature fields, allowing for equal-weighted joint optimization without the sensitivity to hyperparameters required by implicit neural networks [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of 3D Gaussians with optimized parameters (position, rotation, scale, opacity, SH coefficients, and feature vectors); camera parameters (view and projection matrices); image resolution $H \times W$.
*   **Outputs:** A rendered RGB image $\hat{I}$ and a rendered feature map $F_s(\hat{I})$ of resolution $H \times W \times N$ (where $N$ is the internal feature dimension, upsampled to $M$ if necessary) [S1].

**Data Structures**
Each 3D Gaussian $i$ stores the following fields [S1]:
*   **Center:** $x \in \mathbb{R}^3$
*   **Rotation:** Quaternion $q \in \mathbb{R}^4$ (representing rotation matrix $R$)
*   **Scaling:** Factor $s \in \mathbb{R}^3$ (representing scaling matrix $S$)
*   **Opacity:** $\alpha \in \mathbb{R}$
*   **Appearance:** Spherical Harmonics (SH) coefficients up to 3rd order (4 bands)
*   **Semantic Feature:** Vector $f_i \in \mathbb{R}^N$

**Algorithm Steps**
1.  **Tiling and Culling:** Divide the screen into $16 \times 16$ tiles. For each tile, cull 3D Gaussians against the view frustum and the tile's axis-aligned bounding box [S1].
2.  **Projection:** For each remaining Gaussian, compute the 2D covariance matrix $\Sigma'$ in camera coordinates to project the 3D Gaussian to 2D space [S1]:
    $$ \Sigma' = J W \Sigma W^{T} J^{T} $$
    where $W$ is the world-to-camera transformation matrix, $J$ is the Jacobian of the affine approximation of the projective transformation, and $\Sigma$ is the 3D covariance matrix [S1].
    *   *Note:* To ensure $\Sigma$ remains positive semi-definite during optimization, it is decomposed into rotation $R$ and scaling $S$ such that $\Sigma = R S S^{T} R^{T}$ [S1].
3.  **Sorting:** Sort the Gaussians overlapping each pixel by their depth (distance from camera) [S1].
4.  **Joint Rasterization:** For each pixel, perform alpha blending over the sorted overlapping Gaussians $i \in \mathcal{N}$ to compute both the color $C$ and the feature $F_s$ simultaneously [S1]:
    $$ C = \sum_{i \in \mathcal{N}} c_i \alpha_i T_i $$
    $$ F_{s} = \sum_{i \in \mathcal{N}} f_i \alpha_i T_i $$
    where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance, $c_i$ is the color derived from SH coefficients, and $f_i$ is the semantic feature vector [S1].
5.  **Upsampling (Speed-up):** If the target teacher feature dimension $M$ (e.g., 512 for LSeg) is larger than the initialized Gaussian feature dimension $N$ ($N \ll M$), apply a lightweight $1 \times 1$ convolutional decoder to upsample $F_s(\hat{I})$ from $H \times W \times N$ to $H \times W \times M$ [S1].
6.  **Interpolation:** Apply bilinear interpolation to resize the rendered feature map $F_s(\hat{I})$ to match the exact resolution $H \times W$ of the teacher feature map $F_t(I)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_feature_3dgs(gaussians, camera, image_h, image_w, teacher_encoder, decoder):
    """
    Render RGB and Feature maps using Tiled 2D Gaussian Splatting with Feature Distillation.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - center: (3,)
            - q: (4,) quaternion
            - s: (3,) scale
            - alpha: scalar
            - sh_coeffs: array for SH evaluation
            - f: (N,) feature vector
        camera: Object with view_matrix (W) and projection properties.
        image_h, image_w: Output resolution.
        teacher_encoder: Function to encode ground truth image I to F_t.
        decoder: Lightweight 1x1 conv decoder for upsampling features.
        
    Returns:
        rendered_rgb: (H, W, 3)
        rendered_feature: (H, W, M)
    """
    # 1. Initialize output buffers
    # Note: In a real GPU implementation, this is done in parallel via tiles.
    # Here we simulate the logic for a single pixel or tile block.
    
    # 2. Tile-based processing (Simulated for clarity)
    tile_size = 16
    rendered_rgb = np.zeros((image_h, image_w, 3))
    rendered_feature_raw = np.zeros((image_h, image_w, N)) # N is internal dim
    
    for ty in range(0, image_h, tile_size):
        for tx in range(0, image_w, tile_size):
            # Define tile bounds
            y_min, y_max = ty, min(ty + tile_size, image_h)
            x_min, x_max = tx, min(tx + tile_size, image_w)
            
            # Cull Gaussians against tile AABB and view frustum
            active_gaussians = cull_gaussians(gaussians, x_min, x_max, y_min, y_max, camera)
            
            for y in range(y_min, y_max):
                for x in range(x_min, x_max):
                    # Project pixel to get ray direction/origin if needed for depth sorting
                    # Sort active gaussians by depth relative to camera
                    sorted_gs = sort_by_depth(active_gaussians, camera)
                    
                    color_acc = 0.0
                    feature_acc = np.zeros(N)
                    transmittance = 1.0
                    
                    for g in sorted_gs:
                        # Evaluate 2D Gaussian at (x, y)
                        # Compute 2D covariance Sigma' from 3D Sigma using J, W
                        sigma_2d = project_covariance(g, camera)
                        
                        # Calculate alpha contribution
                        # alpha_g = alpha * exp(-0.5 * (x,y)^T Sigma'^-1 (x,y))
                        alpha_g = evaluate_gaussian_alpha(g, x, y, sigma_2d)
                        
                        if alpha_g < epsilon:
                            continue
                            
                        # Evaluate color c_i from SH
                        c_i = evaluate_sh(g.sh_coeffs, g.center, camera.view_direction)
                        
                        # Accumulate
                        weight = alpha_g * transmittance
                        color_acc += c_i * weight
                        feature_acc += g.f * weight
                        
                        transmittance *= (1.0 - alpha_g)
                        
                        if transmittance < epsilon:
                            break
                            
                    rendered_rgb[y, x] = color_acc
                    rendered_feature_raw[y, x] = feature_acc
                    
    # 3. Upsampling
    # If N < M, use decoder to upsample
    if N < M:
        rendered_feature = decoder(rendered_feature_raw)
    else:
        rendered_feature = rendered_feature_raw
        
    # 4. Resize to match teacher resolution if necessary (usually HxW is fixed)
    # Bilinear interpolation is applied if teacher resolution differs
    rendered_feature = bilinear_resize(rendered_feature, teacher_encoder.resolution)
    
    return rendered_rgb, rendered_feature

def cull_gaussians(gaussians, x_min, x_max, y_min, y_max, camera):
    """Filter gaussians that overlap with the tile and are in view."""
    active = []
    for g in gaussians:
        # Compute 2D bounding box of Gaussian in screen space
        bbox = get_screen_bbox(g, camera)
        if intersects(bbox, x_min, x_max, y_min, y_max) and is_in_frustum(g, camera):
            active.append(g)
    return active

def project_covariance(gaussian, camera):
    """Compute 2D covariance Sigma' = J W Sigma W^T J^T"""
    W = camera.view_matrix
    # Sigma is reconstructed from R (from q) and S (from s)
    R = quaternion_to_rotation_matrix(gaussian.q)
    S = np.diag(gaussian.s)
    Sigma_3d = R @ S @ S.T @ R.T
    
    # Jacobian J depends on camera intrinsics and extrinsics
    J = compute_jacobian(camera)
    
    Sigma_2d = J @ W @ Sigma_3d @ W.T @ J.T
    return Sigma_2d
```

**Parameters**
*   **Tile Size:** $16 \times 16$ pixels [S1].
*   **Feature Loss Weight ($\gamma$):** 1.0 [S1].
*   **D-SSIM Weight ($\lambda$):** 0.2 [S1].
*   **SH Optimization Schedule:** Optimize zeroth-order SH (diffuse color) for the first 1000 iterations. Introduce 1 band every 1000 iterations until all 4 bands are represented [S1].

**Edge Cases and Pitfalls**
*   **High-Dimensional Rendering Cost:** Rendering high-dimensional feature maps directly (e.g., $M=512$ for LSeg) is time-consuming [S1]. The solution is to initialize Gaussians with a lower-dimensional feature vector $N \ll M$ and use a lightweight $1 \times 1$ convolutional decoder to upsample the rendered features to dimension $M$ [S1].
*   **Positive Semi-Definite Covariance:** The 3D covariance matrix $\Sigma$ must remain positive semi-definite. This is enforced by parameterizing it via rotation $R$ and scaling $S$ rather than optimizing $\Sigma$ directly [S1].
*   **Resolution Mismatch:** The rendered feature map $F_s(\hat{I})$ must be resized via bilinear interpolation to match the resolution $H \times W$ of the teacher feature map $F_t(I)$ before calculating the $\mathcal{L}_1$ loss [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the rasterization step. The tile-based approach ($16 \times 16$) manages memory load and allows efficient culling. However, rendering high-dimensional features ($N$ or $M$) increases computational cost linearly with the feature dimension [S1]. The speed-up module (1x1 conv) mitigates this by allowing optimization in lower dimensions [S1].
*   **Memory Complexity:** Stores $N$-dimensional feature vectors per Gaussian. Using $N \ll M$ reduces memory footprint during training compared to storing full $M$-dimensional features [S1].
*   **Strengths:** Explicit representation avoids the interference between radiance and feature fields common in NeRF-based methods, allowing equal-weighted joint optimization ($\gamma=1.0$) without adverse effects [S1]. It preserves per-pixel accuracy for features by using the same rasterization procedure as RGB [S1].
*   **Limitations:** Requires a teacher model (e.g., SAM, LSeg) for supervision. High-dimensional feature rendering can be slow without the upsampling decoder [S1].
*   **When to use:** When distilling semantic features from 2D foundation models into a 3D explicit representation for tasks like promptable segmentation or language-driven 3D understanding [S1].

## Background
Feature 3DGS was introduced to enable the distillation of 2D foundation models (such as Segment Anything [20] and LSeg [23]) into 3D Gaussian Splatting representations [S1]. Unlike NeRF-based approaches where feature and radiance fields share layers, leading to interference and sensitivity to the feature loss weight $\gamma$, Feature 3DGS uses an explicit representation that decouples these fields [S1]. This allows for robust joint optimization of RGB and semantic features, facilitating zero-shot and language-driven 3D computer vision tasks [S1].

## Sources
[S1] Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields, https://doi.org/10.1109/cvpr52733.2024.02048, fetched 2026-10-26
