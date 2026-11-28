# Joint Optimization with NeuS

Also called: NeuS-based reconstruction

## Overview
A framework for neural surface reconstruction that jointly optimizes camera poses and a neural implicit surface (SDF) by integrating photometric, geometric, and multi-view consistency losses into the NeuS architecture [S1]. It distinguishes itself from standard NeuS by explicitly handling outlier correspondences through a dual-network color loss strategy and by employing Intersection-of-Union (IoU) and re-projection losses to enforce geometric consistency across views, addressing the shape-radiance ambiguity that often causes standard methods to overfit to outliers [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images $\{I_i\}$, initial camera poses $\{(R_i, t_i)\}$, and matched keypoints $\{k_{p_i}\}$ between image pairs.
*   **Outputs:** Optimized camera poses, a signed distance field (SDF) $f(\mathbf{x})$, and a color field.

**Data Structures:**
*   **Color Networks:** Two networks are used:
    *   $C_o$: Original color network, taking geometry features and viewing direction as input.
    *   $C_n$: Detached color network, taking only geometry features (no viewing direction). Gradients from $L_{color}(C_n)$ backpropagate only to $C_n$ and the shared geometry encoder, not to $C_o$ [S1].
*   **Sampling:** Points $\mathbf{p}_i$ are sampled along rays using a Mixture of Gaussians (MoG) distribution [S1].
*   **Keypoint Representation:** Keypoints are represented as 3D Gaussian distributions $MoG(\mathbf{k}_p)$ for volume intersection calculations [S1].

**Numbered Steps:**
1.  **Ray Sampling:** For each pixel, sample points $\mathbf{p}_i$ along the ray using the MoG distribution defined by the current SDF estimate [S1].
2.  **SDF and Color Prediction:**
    *   Compute the SDF value $f(\mathbf{p}_i)$ and its gradient $\nabla f(\mathbf{p}_i)$ [S1].
    *   Compute opacity $\alpha_i$ and color $\hat{C}_i$ using volume rendering [S1].
    *   Compute color $\hat{C}_n$ using the detached network $C_n$ [S1].
3.  **Loss Calculation:** Compute the components of the total loss $L$ (see Formulas below) [S1].
4.  **Pose and Network Update:**
    *   **Inlier-Inlier Pairs:** Jointly optimize camera poses and network parameters (SDF, $C_o$, $C_n$) [S1].
    *   **Outlier-Outlier Pairs:** Bypass optimization (no gradients applied) [S1].
    *   **Inlier-Outlier Pairs:** Fix the inlier's pose and the NeuS backbone; optimize only the outlier's pose [S1].
5.  **Confidence Update:** Update confidence scores based on PSNR discrepancies between $C_o$ and $C_n$ to filter outliers [S1].

**Formulas:**
The total loss is defined as [S1]:
$$ L = L_{NSR} + \alpha L_{iou} + \beta L_{rep} $$

Where $L_{NSR}$ is the NeuS-based reconstruction loss [S1]:
$$ L_{NSR} = L_{color}(C_o) + L_{color}(C_n) + \lambda L_{reg} $$

*   **Photometric Loss ($L_{color}$):**
    $$ L_{color} = \| \hat{C} - C \|_2^2 $$
    where $\hat{C}$ is the rendered color from volume rendering and $C$ is the ground truth color [S1].

*   **Eikonal Regularization ($L_{reg}$):**
    $$ L_{reg} = \frac{1}{k} \sum_{i=1}^k (\| \nabla f(\mathbf{p}_i) \|_2^2 - 1)^2 $$
    where $f(\mathbf{p}_i)$ is the distance estimate for sampled point $\mathbf{p}_i$ [S1].

*   **Intersection-of-Union Loss ($L_{iou}$):**
    Given matched keypoints $\mathbf{k}_{p_i}$ and $\mathbf{k}_{p_j}$ from images $I_i$ and $I_j$:
    $$ I = MoG(\mathbf{k}_{p_i}) \cdot MoG(\mathbf{k}_{p_j}) $$
    $$ U = MoG(\mathbf{k}_{p_i}) + MoG(\mathbf{k}_{p_j}) - I $$
    $$ L_{iou} = 1 - \frac{I}{U} $$
    where $MoG(\cdot)$ is the mixture of Gaussians for sampling points along the ray [S1].

*   **Re-projection Loss ($L_{rep}$):**
    Approximate the depth $d_i$ of $\mathbf{k}_{p_i}$ by selecting the point with maximal weight. Let $\mathbf{k}'_{p_i}$ be the re-projected point of $\mathbf{k}_{p_i}$ into image $I_j$.
    $$ L_{rep} = L_\delta(\mathbf{k}'_{p_i}, \mathbf{k}_{p_j}) $$
    where $L_\delta$ is the Huber loss [S1].

**Parameters:**
*   $\lambda$: Weight for $L_{reg}$, set to 0.1 [S1].
*   $\alpha$: Weight for $L_{iou}$, set to 0.2 [S1].
*   $\beta$: Weight for $L_{rep}$, set to 0.001 [S1].
*   $\tau$: Threshold for scene graph sparsification (e.g., 70 for SG-NeRF, 45 for DTU) [S1].
*   $\tau_1$: Inlier-outlier threshold for PSNR discrepancy (e.g., 9) [S1].
*   $N_p$: Number of particles for Monte Carlo re-localization, set to 24 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def joint_optimization_step(images, poses, keypoints, sdf_network, color_network_o, color_network_n):
    """
    Performs one step of joint optimization for NeuS-based reconstruction.
    """
    total_loss = 0.0
    l_color_o = 0.0
    l_color_n = 0.0
    l_reg = 0.0
    l_iou = 0.0
    l_rep = 0.0

    # 1. Process each image pair for multi-view constraints
    for (i, j), keypoint_pair in enumerate(matched_keypoint_pairs):
        ray_i = generate_ray(images[i], keypoint_pair.k_pi)
        ray_j = generate_ray(images[j], keypoint_pair.k_pj)
        
        # Sample points using MoG
        points_i = sample_mog(ray_i, sdf_network)
        points_j = sample_mog(ray_j, sdf_network)
        
        # 2. Compute Photometric Losses
        # Forward pass through original network Co
        features_o, view_dirs_o = extract_features_and_views(points_i, poses[i])
        color_o = color_network_o(features_o, view_dirs_o)
        rendered_color_o = volume_render(points_i, color_o, sdf_network)
        l_color_o += mse(rendered_color_o, images[i][keypoint_pair.k_pi])
        
        # Forward pass through detached network Cn (gradients detached from Co)
        features_n = extract_features(points_i, poses[i]) # No view dir
        color_n = color_network_n(features_n)
        rendered_color_n = volume_render(points_i, color_n, sdf_network)
        l_color_n += mse(rendered_color_n, images[i][keypoint_pair.k_pi])
        
        # 3. Compute Eikonal Regularization
        grad_f = compute_gradient(sdf_network, points_i)
        l_reg += mean((norm(grad_f, 2)**2 - 1)**2)
        
        # 4. Compute IoU Loss
        iou_volume = compute_intersection_union(keypoint_pair.k_pi, keypoint_pair.k_pj)
        l_iou += 1.0 - iou_volume
        
        # 5. Compute Re-projection Loss
        depth_i = get_max_weight_depth(points_i)
        k_pi_reprojected = reproject_point(keypoint_pair.k_pi, poses[i], depth_i, poses[j])
        l_rep += huber_loss(k_pi_reprojected, keypoint_pair.k_pj)

    # Aggregate Losses
    l_nsr = l_color_o + l_color_n + lambda_param * l_reg
    total_loss = l_nsr + alpha_param * l_iou + beta_param * l_rep
    
    # 6. Backpropagation based on pair type
    if pair_type(i, j) == "inlier-inlier":
        total_loss.backward() # Optimizes poses, sdf, Co, Cn
    elif pair_type(i, j) == "outlier-outlier":
        pass # Bypass
    elif pair_type(i, j) == "inlier-outlier":
        # Fix inlier pose and backbone, optimize outlier pose only
        # Implementation detail: Mask gradients for inlier pose and network weights
        pass 
        
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Shape-Radiance Ambiguity:** Outliers may exhibit higher PSNR than inliers, leading to incorrect sampling. Mitigated by using the detached network $C_n$ and comparing PSNR discrepancies [S1].
*   **Epipolar Loss Invariance:** Epipolar loss is invariant to the scale of the translation part, making it less reliable for scene geometry than re-projection loss [S1].
*   **Precision:** The Huber loss $L_\delta$ is used for re-projection to handle large errors robustly [S1].

## Subtopics

### [PG-NeuS](4251-pg-neus.md)
A sibling method. No algorithm details provided in this article.

## Operating characteristics
*   **Time Complexity:** The method runs for 150k iterations on the SG-NeRF dataset (average 13 hours) and 300k iterations on the DTU dataset (average 22 hours) on NVIDIA RTX 3090 GPUs [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Handles outliers effectively by bypassing outlier-outlier pairs and optimizing only outlier poses in inlier-outlier pairs. Re-projection loss is more reliable to scene geometry than epipolar loss [S1].
*   **Limitations:** Requires matched keypoints. Performance depends on the accuracy of initial pose estimation and keypoint matching [S1].
*   **When to use:** When robustness to outlier correspondences is critical and high-quality mesh reconstruction is required [S1].
*   **When not to use:** When computational resources are limited and real-time performance is required, or when no keypoint matches are available [S1].

## Background
The method is built upon the NeuS framework [S1]. It addresses limitations in standard neural surface reconstruction where shape-radiance ambiguity leads to overfitting on outliers [S1]. The IoU loss was originally proposed by SG-NeRF [S1], and the re-projection loss is cited from [S1]. The approach was validated on the SG-NeRF and DTU datasets [S1].

## Sources
[S1] Robust SG-NeRF: Robust Scene Graph Aided Neural Surface Reconstruction, https://arxiv.org/abs/2411.13620v1, fetched 2026-10-26
