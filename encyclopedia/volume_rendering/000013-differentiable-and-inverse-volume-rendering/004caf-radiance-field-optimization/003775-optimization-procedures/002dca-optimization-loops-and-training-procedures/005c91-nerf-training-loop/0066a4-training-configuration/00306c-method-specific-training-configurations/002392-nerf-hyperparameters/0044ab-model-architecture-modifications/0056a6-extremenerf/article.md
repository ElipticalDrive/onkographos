# ExtremeNeRF

Also called: Few-shot NeRF under varying-illumination, NeRF Extreme

## Overview
ExtremeNeRF is a few-shot view synthesis framework designed to generate novel views from sparse multi-view images captured under unconstrained, varying illumination [S1]. Unlike standard NeRF methods that require dense input views and consistent lighting, ExtremeNeRF achieves illumination robustness by enforcing multi-view albedo consistency, supported by geometric alignment [S1]. It distinguishes itself from other few-shot methods (e.g., RegNeRF, FreeNeRF) by utilizing a two-stage intrinsic decomposition pipeline (FIDNet and PIDNet) to guide NeRF optimization, allowing it to work effectively with significantly fewer input images (e.g., 3 views) while maintaining high geometric fidelity [S1].

## How it works
ExtremeNeRF builds on the standard NeRF architecture [Parent](11313-model-architecture-modifications.md) by integrating an intrinsic decomposition module that separates albedo (illumination-invariant surface color) from shading (illumination-dependent effects). The method operates in two stages: an offline intrinsic decomposition of input images and an online patch-wise decomposition during NeRF training.

### Inputs and Outputs
- **Inputs**: A set of $N$ input images $\{I_i\}_{i=1}^N$ with known camera poses $(K_i, T_i)$ and intrinsics $K_i$, captured under varying illumination. The number of views $N$ is small (e.g., $N=3$) [S1].
- **Outputs**: A trained NeRF model $F_\Theta$ capable of rendering novel views $\hat{c}(x)$ and depth maps $\hat{d}(x)$ for arbitrary camera poses, along with estimated albedo maps [S1].

### Data Structures
1.  **NeRF Model ($F_\Theta$)**: Standard MLP mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [Parent].
2.  **FIDNet**: A pre-trained Full-image Intrinsic Decomposition Network (based on IIDWW [S1]) used offline. It takes an input image $I$ and outputs a pseudo-albedo map $a_{gt}(x)$ [S1].
3.  **PIDNet**: A Patch-wise Intrinsic Decomposition Network used online. It takes a patch of synthesized color $\hat{c}(x)$ and outputs estimated albedo $\hat{a}(x)$ and shading $\hat{s}(x)$ [S1].
4.  **Camera Parameters**: Intrinsics $K \in \mathbb{R}^{3 \times 3}$ and extrinsics $T \in SE(3)$ for each view [S1].

### Numbered Steps

1.  **Offline Albedo Extraction (FIDNet)**:
    For each input image $I_i$, apply FIDNet to extract the global context-aware pseudo-albedo ground truth $a_{gt}(x)$ [S1]. This step is performed before NeRF training begins [S1].

2.  **NeRF Optimization Loop**:
    For each training iteration:
    a. **Ray Sampling**: Sample rays from novel view camera poses. Render color $\hat{c}(x)$ and depth $\hat{d}(x)$ using the NeRF model [S1].
    b. **Color Loss**: Compute the mean squared error between rendered color $\hat{c}(x)$ and ground-truth color $c_{gt}(x)$ for sampled pixels $S$ [S1]:
       $$ L_{color} = \sum_{x \in S} \|\hat{c}(x) - c_{gt}(x)\|^2_2 $$
    c. **Patch-wise Intrinsic Decomposition (PIDNet)**: Extract patches of size $S_{patch} \times S_{patch}$ from the rendered novel view color $\hat{c}(x)$ [S1]. Feed these patches into PIDNet to estimate albedo $\hat{a}(x)$ and shading $\hat{s}(x)$ [S1].
    d. **Albedo Consistency Loss ($L_{ac}$)**: Enforce that the albedo extracted from the novel view $\hat{a}(x)$ matches the pseudo-albedo $a_{gt}(x')$ from the corresponding input view pixel $x'$ [S1]. This requires geometric alignment (Step 3) to find the correspondence $x'$ [S1].
    e. **Geometric Alignment**: Map pixel $x$ in the novel view to pixel $x'$ in the input view using the rendered depth $\hat{d}(x)$ [S1]:
       $$ x_w = T \hat{d}(x) K^{-1} \bar{x} $$
       $$ x' = (K' T'^{-1} T) \hat{d}(x) K^{-1} \bar{x} $$
       where $\bar{x} = [x^T, 1]^T$ is the homogeneous coordinate, $T, K$ are the novel view pose/intrinsics, and $T', K'$ are the input view pose/intrinsics [S1].
    f. **Visibility Mask Weight ($\omega(x)$)**: Calculate a weight to exclude invalid correspondences caused by occlusions or large projection errors [S1]. Define projection error $E_{proj}(x')$ as the squared difference between rendered depth $\hat{d}(x)$ and projected depth $\tilde{d}(x')$ [S1]:
       $$ E_{proj}(x') = (\hat{d}(x) - \tilde{d}(x'))^2 $$
       The weight is:
       $$ \omega(x) = r_e \left( 1 - \frac{E_{proj}(x')}{M_{proj}} \right) $$
       where $r_e$ is an error rate coefficient that diminishes toward the end of training, and $M_{proj}$ is the maximum projection error [S1].
    g. **Depth Consistency Loss ($L_{dc}$)**: Regularize the projection error between adjacent pixels to prevent smoothing of unrelated surfaces due to occlusions [S1]:
       $$ L_{dc} = \sum_{x' \in P'} \sum_{y \in N(x')} (E_{proj}(y) - E_{proj}(x'))^2 $$
       where $P'$ is the set of pixels in the input view and $N(x')$ are the 4-neighbor adjacent pixels of $x'$ [S1].
    h. **Total Loss**: Combine $L_{color}$, $L_{ac}$ (weighted by $\omega(x)$), $L_{dc}$, and additional regularizations ($L_{ds}, L_{edge}, L_{pid}, L_{chrom}$, frequency regularization) [S1].
    i. **Update**: Update NeRF parameters $\Theta$ and PIDNet parameters using the total loss [S1].

### Formulas and Definitions
- **Intrinsic Decomposition (Lambertian)**: $\log c(x) = \log a(x) + \log s(x)$ [S1].
- **Intrinsic Decomposition (Real-world)**: $\log c(x) = \log a(x) + \log s(x) + l + R$, where $l$ is light color vector and $R$ is non-Lambertian residuals [S1].
- **Projection Error**: $E_{proj}(x') = (\hat{d}(x) - \tilde{d}(x'))^2$ [S1].
- **Visibility Mask Weight**: $\omega(x) = r_e (1 - E_{proj}(x')/M_{proj})$ [S1].
- **Pixel Correspondence**: $x' = (K' T'^{-1} T) \hat{d}(x) K^{-1} \bar{x}$ [S1].
- **3D Point Projection**: $x_w = T \hat{d}(x) K^{-1} \bar{x}$ [S1].
- **Inverse Projection**: $x = K T^{-1} x_w$ [S1].

### Parameters
- **Spatch**: Patch size for PIDNet input/output, set to $32 \times 32$ [S1].
- **Training Resolution**: Image size used for training is $300 \times 400$ [S1].
- **Training Duration**: 70,000 iterations per scene [S1].
- **Hardware**: 4 NVIDIA A100 GPUs [S1].
- **Error Rate Coefficient ($r_e$)**: Diminishes toward the end of training to reduce the number of pairs enforced for cross-view consistency as geometry improves [S1].
- **Maximum Projection Error ($M_{proj}$)**: Threshold for visibility mask calculation [S1].

### Edge Cases and Pitfalls
- **Projective Transformation Errors**: Incorrect synthesized depth values lead to projection errors. The visibility mask $\omega(x)$ handles cases where alignment fails [S1].
- **Occlusions**: Projection errors caused by scene geometry (occlusions) must be distinguished from floating artifacts. The weight $\omega(x)$ diminishes $r_e$ toward the end criteria to rectify inaccurate geometry alignments while occlusions persist [S1].
- **Direct Minimization of $E_{proj}$**: Directly minimizing projection error is counterproductive due to occlusion smoothing unrelated surface depths. Instead, $L_{dc}$ regularizes the projection error between adjacent pixels [S1].
- **FIDNet Substitution**: Replacing FIDNet with other intrinsic decomposition models (e.g., Das, Karaoglu, and Gevers 2022) shows degraded performance, indicating FIDNet is a suitable guide for cross-view consistency [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_extreme_neRF(input_images, input_poses, input_intrinsics, novel_view_poses, novel_view_intrinsics):
    """
    Trains ExtremeNeRF using the two-stage intrinsic decomposition pipeline.
    """
    # Stage 1: Offline Albedo Extraction
    pseudo_albedo_ground_truths = []
    for img in input_images:
        # FIDNet extracts global context-aware pseudo-albedo
        a_gt = FIDNet(img)
        pseudo_albedo_ground_truths.append(a_gt)

    # Initialize NeRF and PIDNet
    nerf_model = NeRF()
    pidnet = PIDNet()
    
    # Parameters
    spatch = 32
    training_iterations = 70000
    re = 1.0  # Error rate coefficient, diminishes over time
    M_proj = 100.0  # Maximum projection error (example value)
    
    for iteration in range(training_iterations):
        # Decay re toward end of training
        current_re = re * (1.0 - iteration / training_iterations)
        
        # Sample rays from novel views
        rays = sample_rays(novel_view_poses)
        
        # Render color and depth
        rendered_color, rendered_depth = nerf_model.forward(rays)
        
        # Compute Color Loss
        L_color = mean_squared_error(rendered_color, ground_truth_color)
        
        # Patch-wise Intrinsic Decomposition
        patches = extract_patches(rendered_color, size=spatch)
        estimated_albedo, estimated_shading = pidnet.forward(patches)
        
        # Geometric Alignment and Visibility Mask
        L_ac = 0.0
        L_dc = 0.0
        
        for x in novel_view_pixels:
            # Map novel view pixel x to input view pixel x'
            x_homogeneous = [x.x, x.y, 1.0]
            x_w = transform_to_world(novel_view_poses, novel_view_intrinsics, rendered_depth[x], x_homogeneous)
            x_prime = transform_to_input_view(input_poses, input_intrinsics, x_w)
            
            # Calculate Projection Error
            projected_depth = get_projected_depth(input_poses, input_intrinsics, x_w)
            E_proj = (rendered_depth[x] - projected_depth) ** 2
            
            # Visibility Mask Weight
            if x_prime is within_fov(input_images) and E_proj < M_proj:
                omega = current_re * (1.0 - E_proj / M_proj)
                
                # Albedo Consistency Loss
                a_gt_prime = pseudo_albedo_ground_truths[input_view_index][x_prime]
                a_est = estimated_albedo[x]
                L_ac += omega * (a_est - a_gt_prime) ** 2
            else:
                omega = 0.0
            
            # Depth Consistency Loss (for adjacent pixels)
            if x_prime is valid:
                for neighbor in get_4_neighbors(x_prime):
                    E_proj_neighbor = calculate_projection_error(input_poses, input_intrinsics, neighbor)
                    L_dc += (E_proj_neighbor - E_proj) ** 2
        
        # Total Loss
        L_total = L_color + L_ac + L_dc + L_regularization
        
        # Update parameters
        update(nerf_model, L_total)
        update(pidnet, L_total)
```

## Operating characteristics
- **Training Time**: ExtremeNeRF requires approximately 1.5 hours for 3 views on 4 NVIDIA A100 GPUs, compared to 2.4 hours for RegNeRF and 2 hours for FreeNeRF [S1]. Most other few-shot NeRF methods require 10 to 20 hours [S1].
- **Memory**: Uses standard NeRF memory plus overhead for FIDNet and PIDNet [S1].
- **Strengths**: Works with very few input images (few-shot) under unconstrained illumination; achieves better geometry details than baselines that struggle with inconsistent illumination [S1].
- **Limitations**: Relies on the quality of FIDNet; replacing FIDNet with other intrinsic decomposition models degrades performance [S1]. Performance may be limited by the accuracy of depth estimation in the presence of occlusions [S1].
- **When to Use**: When input data is sparse (few views) and illumination varies significantly across views [S1].
- **When Not to Use**: When dense input views are available and illumination is consistent (standard NeRF is sufficient); when real-time inference is required (training time is non-trivial) [S1].

## Background
ExtremeNeRF was introduced in the paper "Few-Shot Neural Radiance Fields under Unconstrained Illumination" presented at the AAAI Conference on Artificial Intelligence in 2024 [S1]. It addresses the limitation of existing NeRF methods that require extensive multi-view images under constrained illumination [S1]. The method builds upon the concept of intrinsic image decomposition, separating albedo and shading, and applies it to the NeRF optimization process to handle varying illumination in few-shot settings [S1]. It utilizes the NeRF Extreme benchmark, the first in-the-wild benchmark for novel view synthesis under varying illumination [S1].

## Sources
[S1] Few-Shot Neural Radiance Fields under Unconstrained Illumination, https://doi.org/10.1609/aaai.v38i4.28075, fetched 2026-10-27
