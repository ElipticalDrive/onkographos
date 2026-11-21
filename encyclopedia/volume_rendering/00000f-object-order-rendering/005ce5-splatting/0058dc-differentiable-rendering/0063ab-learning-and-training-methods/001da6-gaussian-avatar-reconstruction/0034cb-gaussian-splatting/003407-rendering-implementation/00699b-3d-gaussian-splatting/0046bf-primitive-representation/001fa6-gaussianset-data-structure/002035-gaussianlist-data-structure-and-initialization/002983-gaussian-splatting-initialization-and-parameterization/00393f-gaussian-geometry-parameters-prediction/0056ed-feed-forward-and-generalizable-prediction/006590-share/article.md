# SHARE

Also called: Pose-free 3D Gaussian splatting via shape-ray estimation

## Overview
SHARE is a pose-free framework for generalizable 3D Gaussian splatting that estimates camera poses via predicted rays and uses anchor-aligned coarse-to-fine geometry estimation to predict 3D Gaussian parameters without ground-truth poses [S1]. It sets itself apart from sibling methods like [PixelSplat](16423-pixelsplat.md) and [MVSplat](16423-mvsplat.md) by explicitly embedding pose awareness into multi-view cost volumes via Plücker rays, making it robust to pose inaccuracies where generalizable 3D Gaussian splatting methods are highly sensitive [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $M$ input images $\{I_i\}_{i=1}^M$. For DTU, $M=4$ (3 context, 1 target); for RealEstate10K, $M=5$ (2 context, 3 targets) [S1]. Input resolution is $256 \times 256$ [S1].
*   **Outputs:** A set of 3D Gaussians $\mathcal{G} = \{\mu, \alpha, \Sigma, c\}$ and estimated camera poses (implicitly via rays) [S1].

**Data Structures:**
*   **Cost Volume $C_i$:** For each viewpoint $i$, a cost volume of shape $\mathbb{R}^{H/4 \times W/4 \times D}$, where $D$ is the number of depth hypotheses [S1].
*   **Spatial Weight Volume $W_i$:** Shape $\mathbb{R}^{H/4 \times W/4}$, accounting for view-dependent effects (occlusion, lighting) [S1].
*   **Canonical Geometry Volume $V_g$:** Shape $\mathbb{R}^{H/4 \times W/4 \times C}$, fusing refined cost volumes [S1].
*   **Canonical Feature Volume $V_f$:** Shape $\mathbb{R}^{H \times W \times C}$, constructed from upsampled multi-view features [S1].
*   **Gaussian Primitives:** $K=3$ offsets per anchor [S1].

**Steps:**

1.  **Feature Extraction and Cost Volume Construction:**
    Extract features from input images. Construct initial cost volumes $C_i \in \mathbb{R}^{H/4 \times W/4 \times D}$ for each viewpoint $i$ using matching Transformer layers [S1]. The cost volume is computed as:
    $$ C_i = \frac{1}{\sqrt{C_i}} \sum_{j \neq i} F_{j \to i} $$
    where $F_{j \to i}$ are warped features [S1].

2.  **Pose-Aware Cost Aggregation:**
    Refine cost volumes $C_i$ through cost aggregation conditioned on predicted Plücker rays. Use patch-wise cross-attention to embed pose awareness into the cost volumes, introducing geometric bias [S1].

3.  **Spatial Weight Estimation:**
    Estimate a spatial weight volume $W_i \in \mathbb{R}^{H/4 \times W/4}$ for each cost volume to account for view-dependent effects such as occlusion and lighting variations [S1].

4.  **Canonical Volume Construction:**
    Fuse refined cost volumes $C'_i$ and weights $W_i$ into a canonical geometry volume $V_g \in \mathbb{R}^{H/4 \times W/4 \times C}$:
    $$ V_g = \sum_{i=1}^M W_i C'_i + \phi \oplus C' $$
    where $C'$ is the average of all cost volumes, $\oplus$ denotes channel-wise concatenation, and $\phi$ is a CNN layer for channel projection. A mean and variance-based volume is also added following previous MVS techniques [S1]. Similarly, construct the canonical feature volume $V_f \in \mathbb{R}^{H \times W \times C}$ using the same aggregation method with upsampled multi-view features [S1].

5.  **Coarse Depth Estimation:**
    Compute initial coarse depth $z$ as the expected value over depth hypothesis planes $D \in \mathbb{R}^D$. The probability of each depth candidate is predicted using $V_g$ through a lightweight MLP depth head $f_g$ followed by a softmax function:
    $$ z = \sum_d \text{softmax}(f_g(V_g))_d \cdot D_d $$
    The estimated 3D anchor position $p_i$ for pixel $i$ is:
    $$ p_i = o + z_i d_i $$
    where $o$ is the ray origin and $d_i$ is the ray direction [S1].

6.  **Anchor-Aligned Offset Prediction:**
    Estimate $K$ offset vectors $\Delta p_{i,k}$ for each anchor using the canonical feature volume $V_f$. The offset prediction MLP head $f_o$ outputs:
    $$ \Delta p_{i,k} = f_o(V_f) $$
    where $k \in \{1, \dots, K\}$ and $K=3$ [S1].

7.  **Gaussian Parameter Prediction:**
    Define Gaussian means as $\mu_{i,k} = p_i + \Delta p_{i,k}$. Predict remaining parameters (opacity $\alpha$, covariance $\Sigma$, color $c$) using an MLP head $f_p$ applied to the positional encoding of offsets and the feature volume:
    $$ \{\alpha, \Sigma, c\}_{i,k} = f_p(\text{PE}(\Delta p_{i,k}), V_f) $$
    where $\text{PE}(\cdot)$ denotes positional encoding [S1].

8.  **Rendering:**
    Rasterize the predicted Gaussians into 2D space to render the target image [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def share_forward(images, num_depth_hypotheses, num_offsets=3):
    """
    Forward pass for SHARE.
    images: List of M images, shape (M, 3, 256, 256)
    num_depth_hypotheses: D, number of depth planes
    num_offsets: K, number of Gaussian offsets per anchor
    """
    M = len(images)
    
    # 1. Feature Extraction and Cost Volume Construction
    # Extract features using multi-view fusion backbone (e.g., 6 matching Transformer layers)
    features = extract_features(images) # Shape: (M, C_feat, H/4, W/4)
    
    # Construct initial cost volumes C_i
    # C_i shape: (M, H/4, W/4, D)
    cost_volumes = []
    for i in range(M):
        # Warp features from other views j to view i
        warped_features = warp_features(features[j] for j in range(M) if j != i, features[i])
        # Compute cost volume (simplified aggregation)
        C_i = aggregate_cost(warped_features, features[i])
        cost_volumes.append(C_i)
        
    # 2. Pose-Aware Cost Aggregation
    # Predict Plücker rays for pose awareness
    plucker_rays = predict_plucker_rays(images) # Shape: (M, H/4, W/4, 6)
    
    refined_cost_volumes = []
    for i in range(M):
        # Patch-wise cross-attention conditioned on Plücker rays
        C_refined = patch_cross_attention(cost_volumes[i], plucker_rays[i])
        refined_cost_volumes.append(C_refined)
        
    # 3. Spatial Weight Estimation
    # Estimate spatial weight volumes W_i
    weight_volumes = []
    for i in range(M):
        W_i = predict_spatial_weights(refined_cost_volumes[i]) # Shape: (M, H/4, W/4)
        weight_volumes.append(W_i)
        
    # 4. Canonical Volume Construction
    # Average cost volumes
    avg_cost = torch.mean(torch.stack(refined_cost_volumes), dim=0)
    
    # Fuse into canonical geometry volume V_g
    # V_g shape: (H/4, W/4, C)
    V_g = torch.zeros(H/4, W/4, C)
    for i in range(M):
        # Weighted sum and concatenation
        term1 = weight_volumes[i] * refined_cost_volumes[i]
        term2 = conv_project(torch.cat([refined_cost_volumes[i], avg_cost], dim=-1))
        V_g += term1 + term2
        
    # Construct canonical feature volume V_f
    # Upsample multi-view features and aggregate similarly
    V_f = construct_canonical_feature_volume(images, features) # Shape: (H, W, C)
    
    # 5. Coarse Depth Estimation
    # Lightweight MLP depth head f_g
    depth_logits = depth_head(V_g) # Shape: (H/4, W/4, D)
    depth_probs = softmax(depth_logits, dim=-1)
    
    # Depth hypothesis planes D
    D_planes = create_depth_planes(num_depth_hypotheses) # Shape: (D,)
    
    # Expected depth z
    z = torch.sum(depth_probs * D_planes, dim=-1) # Shape: (H/4, W/4)
    
    # Ray origins and directions (from predicted poses/Plücker rays)
    ray_origins, ray_directions = get_ray_info(plucker_rays) # Shape: (H/4, W/4, 3)
    
    # Anchor positions p_i
    anchors = ray_origins + z.unsqueeze(-1) * ray_directions # Shape: (H/4, W/4, 3)
    
    # 6. Anchor-Aligned Offset Prediction
    # Offset prediction MLP head f_o
    offsets = offset_head(V_f) # Shape: (H, W, K, 3)
    
    # 7. Gaussian Parameter Prediction
    # Upsample anchors to match V_f resolution if necessary, or interpolate
    anchors_upsampled = interpolate(anchors, target_size=(H, W)) # Shape: (H, W, 3)
    
    # Gaussian means
    means = anchors_upsampled.unsqueeze(-2) + offsets # Shape: (H, W, K, 3)
    
    # Remaining parameters
    # Positional encoding of offsets
    pe_offsets = positional_encoding(offsets)
    
    # MLP head f_p
    alphas, sigmas, colors = gaussian_params_head(pe_offsets, V_f) # Shapes: (H, W, K, 1), (H, W, K, 6), (H, W, K, C_color)
    
    # 8. Rendering
    rendered_image = rasterize_gaussians(means, alphas, sigmas, colors, camera_intrinsics)
    
    return rendered_image, means, alphas, sigmas, colors
```

**Parameters:**
*   **Context/Target Images:** 3 context + 1 target for DTU; 2 context + 3 targets for RealEstate10K [S1].
*   **Input Resolution:** $256 \times 256$ [S1].
*   **Number of Gaussian Primitives per Anchor ($K$):** 3 [S1].
*   **Number of Depth Hypotheses ($D$):** Not given in the sources.
*   **Training Iterations:** 140K on DTU, 300K on RealEstate10K [S1].
*   **Optimizer:** Adam [S1].
*   **Hardware:** NVIDIA RTX 4090 GPU [S1].

**Edge Cases and Pitfalls:**
*   **Pose Uncertainty:** SHARE is designed to be robust to pose inaccuracies, whereas generalizable 3D Gaussian splatting methods (e.g., PixelSplat, MVSplat) are highly sensitive to pose errors, with even minor Gaussian noise ($\sigma = 0.01$) causing significant degradation [S1].
*   **Depth Discontinuities:** Coarse depth estimation may lead to over-smooth geometry in regions with depth discontinuities or complex surfaces. The anchor-aligned coarse-to-fine estimation with offsets mitigates this [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **DTU (Predicted Poses via DUSt3R):** PSNR 19.36, SSIM 0.61, LPIPS 0.31 [S1].
    *   **RealEstate10K (Predicted Poses via DUSt3R):** PSNR 21.23, SSIM 0.64, LPIPS 0.26 [S1].
*   **Scaling:** Demonstrates scalability to complex, large-scale scenes (RealEstate10K) [S1].
*   **Strengths:** Robust to pose uncertainty; outperforms generalizable 3D Gaussian splatting under noisy pose conditions; achieves LPIPS/SSIM scores comparable to ground-truth poses [S1].
*   **Limitations:** Not given in the sources.
*   **When to Use:** When ground-truth camera poses are unavailable or unreliable, and robust generalizable 3D Gaussian splatting is required [S1].
*   **When Not to Use:** When high-precision ground-truth poses are available and pose-free methods are not necessary [S1].

## Background
SHARE was introduced in the paper "Pose-free 3D Gaussian splatting via shape-ray estimation" [S1]. It addresses the challenge of generalizable 3D Gaussian splatting without requiring ground-truth camera poses, which are often unavailable or inaccurate in real-world scenarios [S1]. The method builds on [Feed-forward and Generalizable Prediction](15832-feed-forward-and-generalizable-prediction.md) by incorporating pose estimation directly into the geometry prediction pipeline [S1].

## Sources
[S1] Pose-free 3D Gaussian splatting via shape-ray estimation, https://arxiv.org/abs/2505.22978, fetched 2026-10-26
