# Splatt3R

Also called: Splatt3R: Zero-Shot Gaussian Splatting from Uncalibrated Image Pairs

## Overview
Splatt3R is a feed-forward method that predicts 3D Gaussian Splat primitives directly from a pair of uncalibrated natural images, without requiring camera intrinsics, extrinsics, or depth information [S1, unofficial][S2]. It extends the MASt3R foundation model by adding a Gaussian prediction head to output geometry and appearance parameters in a single forward pass [S1, unofficial]. Unlike existing generalizable 3D-GS methods that are limited to interpolating viewpoints between input stereo views, Splatt3R supports extrapolation to wider baselines using a novel loss masking strategy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Two uncalibrated RGB images $I_1, I_2 \in \mathbb{R}^{H \times W \times 3}$ (typically $512 \times 512$) [S1, unofficial]. Camera parameters are unknown [S1, unofficial].
*   **Output:** A set of 3D Gaussian primitives $\mathcal{G}$, defined by mean $\mu$, covariance $\Sigma$, opacity $\alpha$, and color $c$ (or spherical harmonics coefficients) [S1, unofficial][S2].

**Data Structures:**
The method predicts per-pixel Gaussians. For each pixel $j$ in the input images, the network outputs:
*   Base position $x_j \in \mathbb{R}^3$ (in the coordinate frame of the first image) [S1, unofficial].
*   Position offset $\Delta x_j \in \mathbb{R}^3$ [S1, unofficial][S2].
*   Rotation quaternion $q_j \in \mathbb{R}^4$ [S1, unofficial][S2].
*   Scale $s_j \in \mathbb{R}^3$ [S1, unofficial][S2].
*   Opacity $\alpha_j \in \mathbb{R}$ [S1, unofficial][S2].
*   Color $c_j$: Constant view-independent color $S \in \mathbb{R}^3$ is used in experiments, with ablation for view-dependent spherical harmonics [S1, unofficial].

**Architecture:**
1.  **Encoder:** The two input images are encoded using a shared Vision Transformer (ViT) encoder and cross-attention transformers. This backbone is based on MASt3R and is frozen during training [S1, unofficial].
2.  **Point Cloud Head:** The frozen MASt3R backbone predicts 3D point locations $\hat{X}_1, \hat{X}_2$ and confidence maps $C_1, C_2$ in the coordinate frame of the first image [S1, unofficial].
3.  **Gaussian Head:** A parallel head (using DPT architecture) predicts the additional Gaussian attributes: offsets $\Delta$, rotations $q$, scales $s$, opacities $\alpha$, and spherical harmonics $S$ for each pixel [S1, unofficial].
4.  **Primitive Construction:** The final set of Gaussian primitives is the union of primitives predicted from both input images [S1, unofficial].

**Formulas:**
The mean position $\mu_j$ of each Gaussian is calculated as the sum of the predicted base position $x_j$ and the offset $\Delta x_j$:
$$ \mu_j = x_j + \Delta x_j $$
[S1, unofficial][S2]

The covariance matrix $\Sigma_j$ is reparameterized using the rotation quaternion $q_j$ and scale $s_j$ to ensure positive semi-definiteness:
$$ \Sigma_j = R(q_j) S(s_j) S(s_j)^T R(q_j)^T $$
where $R(q_j)$ is the rotation matrix derived from the quaternion and $S(s_j)$ is the diagonal scale matrix [S1, unofficial][S2].

The total loss $L$ is a weighted sum of masked Mean Squared Error (MSE) and LPIPS losses:
$$ L = \lambda_{MSE} L_{MSE}(M \odot \hat{I}, M \odot I) + \lambda_{LPIPS} L_{LPIPS}(M \odot \hat{I}, M \odot I) $$
where $\hat{I}$ is the rendered image, $I$ is the ground truth target image, $M$ is the loss mask, and $\odot$ denotes element-wise multiplication [S1, unofficial].

**Parameters:**
*   **Activation Functions:** Normalization for quaternions, exponential for scales and offsets, and sigmoid for opacities [S1, unofficial].
*   **Color:** Constant view-independent color $S \in \mathbb{R}^3$ is used in the primary experiments [S1, unofficial].
*   **Overlap Thresholds:** Hyperparameters $\phi$ and $\psi$ control the selection of context and target images during training. Typical values tested: Close $(\phi=0.9, \psi=0.9)$, Medium $(0.7, 0.7)$, Wide $(0.5, 0.5)$, Very Wide $(0.3, 0.3)$ [S1, unofficial].

**Training Procedure:**
Training is performed in two stages:
1.  **Geometry Loss:** The 3D point cloud's geometry is optimized using a confidence-based regression loss $L_{pts}$ and $L_{regr}$ from the MASt3R backbone [S1, unofficial].
2.  **Novel View Synthesis:** The Gaussian head is trained to minimize the rendering loss on target views [S1, unofficial].

**Loss Masking:**
To prevent destructive updates from unseen parts of the scene, a loss mask $M$ is calculated for each target image:
1.  Unproject target image points to 3D.
2.  Reproject these points onto the context images.
3.  Check if the reprojected pixel is inside the frustum of at least one context view.
4.  Check if the rendered depth matches the ground truth depth within a tolerance.
5.  Pixels satisfying these conditions are marked as valid (mask $M=1$); others are masked out ($M=0$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatt3r_forward(I1, I2, backbone, gaussian_head):
    """
    Forward pass of Splatt3R.
    I1, I2: Input images (H, W, 3)
    backbone: Frozen MASt3R encoder (ViT + Cross-Attention)
    gaussian_head: DPT-based decoder for Gaussian parameters
    Returns: List of Gaussian primitives (mu, sigma, alpha, color)
    """
    # 1. Encode images using frozen MASt3R backbone
    features = backbone.encode(I1, I2)
    
    # 2. Predict base 3D points and confidences (in I1's frame)
    # x1, x2 are base positions; c1, c2 are confidences
    x1, c1 = backbone.point_head(features, view=1)
    x2, c2 = backbone.point_head(features, view=2)
    
    # 3. Predict Gaussian attributes using the parallel Gaussian Head
    # Outputs: offsets (delta), rotations (q), scales (s), opacities (alpha), colors (S)
    delta1, q1, s1, alpha1, S1 = gaussian_head(features, view=1)
    delta2, q2, s2, alpha2, S2 = gaussian_head(features, view=2)
    
    # 4. Construct Gaussian primitives
    gaussians = []
    
    # Process primitives from Image 1
    for j in range(num_pixels):
        mu = x1[j] + delta1[j]
        sigma = compute_covariance(q1[j], s1[j])
        alpha = alpha1[j]
        color = S1[j]
        gaussians.append(Gaussian(mu, sigma, alpha, color))
        
    # Process primitives from Image 2
    for j in range(num_pixels):
        # Note: x2 is also in I1's frame according to MASt3R convention
        mu = x2[j] + delta2[j]
        sigma = compute_covariance(q2[j], s2[j])
        alpha = alpha2[j]
        color = S2[j]
        gaussians.append(Gaussian(mu, sigma, alpha, color))
        
    return gaussians

def compute_covariance(q, s):
    """
    Compute covariance matrix from quaternion q and scale s.
    q: (4,) quaternion
    s: (3,) scale
    Returns: (3, 3) covariance matrix
    """
    R = quaternion_to_rotation_matrix(q)
    S = diag(s)
    return R @ S @ S.T @ R.T

def calculate_loss_mask(target_depth, target_pose, context_depths, context_poses, tolerance=0.05):
    """
    Calculate loss mask M for a target view.
    M[i,j] = 1 if pixel (i,j) is visible and consistent in context views.
    """
    H, W = target_depth.shape
    M = np.zeros((H, W), dtype=bool)
    
    # Unproject target pixels to 3D
    target_points_3d = unproject(target_depth, target_pose)
    
    for k in range(len(context_poses)):
        # Reproject onto context view k
        context_points_2d, depths = reproject(target_points_3d, context_poses[k], target_pose)
        
        # Check frustum and depth consistency
        valid_in_frustum = is_inside_frustum(context_points_2d, context_poses[k])
        depth_match = np.abs(depths - context_depths[k][context_points_2d]) < tolerance
        
        # Combine conditions
        valid_pixels = valid_in_frustum & depth_match & (context_depths[k][context_points_2d] > 0)
        
        # Update mask
        M[valid_pixels] = True
        
    return M
```

**Edge Cases and Pitfalls:**
*   **Unseen Regions:** Supervising rendering loss for pixels not visible in context views (obscured or outside frustums) is counterproductive and can destroy model performance [S1, unofficial]. This is addressed by the loss masking strategy [S1, unofficial].
*   **Local Minima:** Directly supervising positions using ground-truth point clouds avoids the local minima present in training 3D Gaussian Splats from stereo views where camera rays are unknown [S1, unofficial].

## Subtopics

### MASt3R Backbone
The foundation model for uncalibrated stereo reconstruction. It predicts 3D points and confidences without camera parameters. Splatt3R freezes this backbone and adds a Gaussian head [S1, unofficial].

### Loss Masking Strategy
A technique to restrict training supervision to pixels that are feasibly reconstructed (visible and depth-consistent in context views), enabling extrapolation beyond the stereo baseline [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Single forward pass for inference. Construction/export runtime is approximately 1.10 seconds on ScanNet++ with 2 views [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On ScanNet++ test scenes with Close view overlap ($\phi=0.9, \psi=0.9$), Splatt3R achieves PSNR 19.66, SSIM 0.757, LPIPS 0.234 [S1, unofficial]. With 2 views, it achieves PSNR 23.30, SSIM 0.823, LPIPS 0.125 [S2].
*   **Strengths:** Pose-free, works with uncalibrated images, supports extrapolation to wider baselines, fast inference [S1, unofficial][S2].
*   **Limitations:** Performance degrades with very wide baselines (e.g., PSNR drops to 18.56 for Very Wide overlap) [S1, unofficial]. Requires a large pre-trained backbone (MASt3R) [S1, unofficial].

## Background
Splatt3R was introduced in 2024 by Brandon, Zheng, Iro, and Victor from the University of Oxford [S1, unofficial]. It builds upon the MASt3R foundation model for 3D geometry reconstruction [S1, unofficial]. The method was trained on the ScanNet++ dataset [S1, unofficial]. It addresses the limitation of existing generalizable 3D-GS methods that only supervise viewpoints between input stereo views, by enabling supervision of extrapolated viewpoints [S1, unofficial].

## Sources
[S1] Splatt3R: Zero-Shot Gaussian Splatting from Uncalibrated Image Pairs, https://splatt3r.active.vision/, fetched 2026-10-30 (unofficial)
[S2] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
