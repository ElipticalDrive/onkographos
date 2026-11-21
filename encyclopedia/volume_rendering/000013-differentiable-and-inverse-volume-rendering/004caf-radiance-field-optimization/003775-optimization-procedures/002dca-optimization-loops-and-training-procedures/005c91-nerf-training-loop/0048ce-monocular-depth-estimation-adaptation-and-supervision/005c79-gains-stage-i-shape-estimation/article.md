# GAINS Stage I: Shape Estimation

Also called: GAINS Shape Estimation, GAINS Stage 1

## Overview
GAINS Stage I is the geometry optimization phase of the GAINS pipeline, designed to reconstruct robust 3D shape from a sparse set of input RGB images using 2D Gaussian Splatting (2DGS) [S1]. It solves the problem of novel-view artifacts and geometric collapse that occur when standard 2DGS losses are applied to sparse inputs by augmenting the optimization with learning-based priors: monocular depth/normal consistency, local depth ranking, and Score Distillation Sampling (SDS) from a latent diffusion model [S1]. Unlike its sibling methods that may rely on differentiable volume rendering or MVS priors, GAINS explicitly leverages a pre-trained diffusion model to regularize unseen viewpoints, ensuring the geometry lies on the natural object manifold [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Sparse set of RGB images $\{I_i\}_{i=1}^N$ (typically $N \in [4, 16]$) [S1].
*   Known camera parameters $C_i = \{K_i, R_i, t_i\}$ for each view [S1].
*   Pre-trained monocular depth estimator (outputs $\hat{D}_i$) [S1].
*   Pre-trained monocular normal estimator (outputs $\tilde{N}_i$) [S1].
*   Pre-trained latent diffusion model $\epsilon_\phi$ [S1].
*   Optional: Ground-truth alpha masks for outlier removal [S1].

**Outputs:**
*   Optimized 2DGS geometry representation $G = \{\mu_j, S_j, R_j, \sigma_j\}_{j=1}^M$, where $\mu_j$ is position, $S_j$ is scaling, $R_j$ is rotation, and $\sigma_j$ is opacity [S1].

### Data Structures
The scene is represented as a collection of 2D Gaussians. Each Gaussian $j$ stores:
*   $\mu_j \in \mathbb{R}^3$: 3D position.
*   $S_j \in \mathbb{R}^3$: Scaling vector.
*   $R_j \in \mathbb{R}^{3 \times 3}$: Rotation matrix.
*   $\sigma_j \in \mathbb{R}$: Opacity.
*   Surface normal $n_j$ is derived from $R_j$ [S1].

### Parameters
*   $\lambda_{DC} = 0.005$: Depth consistency weight [S1].
*   $\lambda_{DR} = 10$: Depth ranking weight [S1].
*   $\lambda_{NC} = 1$: Normal consistency weight [S1].
*   $\lambda_{N} = 0.25$: Normal guided loss weight [S1].
*   $\lambda_{SDS} = 0.0001$: Diffusion guidance weight [S1].
*   $\lambda_{BCE} = 0.75$: Binary Cross-Entropy loss weight [S1].
*   $\lambda = 0.2$: Weight for L1 vs LD-SSIM in color loss (standard 2DGS default, implied by $(1-\lambda)L1 + \lambda L_{D-SSIM}$) [S1].
*   Total iterations: 16,000 [S1].
*   SDS activation iteration: 10,000 [S1].
*   SDS guidance scale: 100 [S1].
*   Novel viewpoints sampled: 100 [S1].
*   Floater removal interval: Every 1,000 iterations [S1].

### Step-by-Step Method

1.  **Initialization:** Initialize the 2DGS primitives $G$ using standard initialization procedures for 2DGS [S1].
2.  **Loss Calculation:** For each training iteration, compute the total loss $\mathcal{L}_{total}$ as the sum of the following components:

    **A. Re-rendering Loss ($\mathcal{L}_{col}$):**
    Render the image $R_{geo}(C_i, G)$ for each input view $C_i$. Compute:
    $$ \mathcal{L}_{col} = (1 - \lambda) \mathcal{L}_1(R_{geo}, I_i) + \lambda \mathcal{L}_{D-SSIM}(R_{geo}, I_i) $$
    where $\mathcal{L}_1$ is the L1 loss and $\mathcal{L}_{D-SSIM}$ is the D-SSIM loss [S1].

    **B. Depth Distortion Loss ($\mathcal{L}_{DD}$):**
    Optional loss to reduce depth ambiguity by bringing intersected Gaussians closer along each ray [S1].

    **C. Normal Consistency Loss ($\mathcal{L}_{NC}$):**
    Compute surface normals $\tilde{N}_i$ from the rendered depth. Compare with normals $N_i$ derived from the Gaussian rotation matrices:
    $$ \mathcal{L}_{NC} = 1 - \frac{1}{M} \sum_{j=1}^M N_j^T \tilde{N}_j $$
    (Note: The source defines $\mathcal{L}_{NC} = 1 - N_i^T \tilde{N}_i$, implying an average over primitives or pixels) [S1].

    **D. Depth Consistency Loss ($\mathcal{L}_{DC}$):**
    Render depth map $D_i$ from $G$. Obtain monocular depth estimate $\hat{D}_i$ from the pre-trained MDE network. Compute the Pearson Correlation Coefficient (PCC) between $D_i$ and $\hat{D}_i$:
    $$ \mathcal{L}_{DC} = 1 - \text{PCC}(D_i, \hat{D}_i) $$
    This enforces similarity in depth ordering and relative scale, mitigating hallucinations from MDE [S1].

    **E. Local Depth Ranking Loss ($\mathcal{L}_{DR}$):**
    Add $\mathcal{L}_{DR}$ to prevent geometric collapse from hard constraints and alleviate long-range ambiguity [S1]. (Specific formula not provided in source, cited as [9]) [S1].

    **F. Normal Guided Loss ($\mathcal{L}_{N}$):**
    Obtain reference normals $\tilde{N}_i$ from the pre-trained monocular normal estimator. Obtain rendered normals $N_i$ from the Gaussian geometry. Obtain surface normals $\hat{N}_i$ from the rendered depth. Compute:
    $$ \mathcal{L}_{N} = \mathcal{L}_1(\tilde{N}_i, \hat{N}_i) + 0.1 \cdot \mathcal{L}_1(\tilde{N}_i, N_i) + \text{TV}(\tilde{N}_i, \hat{N}_i) $$
    where $\text{TV}$ is the Total Variation loss [S1].

    **G. Diffusion Guidance Loss ($\mathcal{L}_{SDS}$):**
    *Condition:* Only compute if iteration $> 10,000$.
    1.  Sample 100 novel viewpoints $C_j$.
    2.  Render geometry $R_{geo}(C_j, G)$.
    3.  Apply forward diffusion to get noisy latents $Z_j$:
        $$ Z_j = \alpha_t R_{geo}(C_j, G) + \sigma_t \epsilon $$
        where $t \sim U(0.02, 0.98)$ and $\epsilon \sim \mathcal{N}(0, I)$ [S1].
    4.  Compute SDS loss:
        $$ \mathcal{L}_{SDS} = \mathbb{E}_{t, \epsilon} w(t) \| \epsilon_\phi(Z_j; t) - \epsilon \|_2^2 $$
        where $\epsilon_\phi$ is the pre-trained diffusion model predicting noise, and $w(t)$ is the weighting function [S1].

    **H. Outlier Removal Loss ($\mathcal{L}_{BCE}$):**
    *Condition:* Only if ground-truth alpha masks are provided.
    Compute Binary Cross-Entropy between the reconstructed alpha mask and the ground truth mask [S1].

3.  **Optimization Step:**
    Minimize the total loss:
    $$ \mathcal{L}_{total} = \mathcal{L}_{col} + \lambda_{DD} \mathcal{L}_{DD} + \lambda_{NC} \mathcal{L}_{NC} + \lambda_{DC} \mathcal{L}_{DC} + \lambda_{DR} \mathcal{L}_{DR} + \lambda_{N} \mathcal{L}_{N} + \lambda_{SDS} \mathcal{L}_{SDS} + \lambda_{BCE} \mathcal{L}_{BCE} $$
    Update parameters of $G$ using gradient descent [S1].

4.  **Periodic Maintenance:**
    Every 1,000 iterations, apply KNN-based floater removal to prune isolated or erroneous Gaussians [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gains_stage_i_shape_estimation(images, cameras, mde_net, normal_net, diffusion_model, 
                                   alpha_masks=None, total_iterations=16000):
    """
    Optimizes 2DGS geometry using GAINS Stage I losses.
    """
    # Hyperparameters
    lambda_dc = 0.005
    lambda_dr = 10
    lambda_nc = 1
    lambda_n = 0.25
    lambda_sds = 0.0001
    lambda_bce = 0.75
    lambda_dd = 1.0 # Default weight for depth distortion, not explicitly given but used in sum
    lambda_color_l1 = 0.8
    lambda_color_ss = 0.2
    sds_start_iter = 10000
    floater_interval = 1000
    num_novel_views = 100
    
    # Initialize 2DGS primitives G
    G = initialize_2dgs(images, cameras)
    
    for iteration in range(total_iterations):
        total_loss = 0.0
        gradients = {}
        
        # 1. Re-rendering Loss (L_col)
        rendered_images = []
        rendered_depths = []
        rendered_normals = []
        
        for i, (img, cam) in enumerate(zip(images, cameras)):
            # Render image, depth, and normals from G
            R_img, R_depth, R_normal = render_2dgs(G, cam)
            rendered_images.append(R_img)
            rendered_depths.append(R_depth)
            rendered_normals.append(R_normal)
            
            # L_col = (1 - lambda) * L1 + lambda * L_D-SSIM
            l1_loss = l1_loss(R_img, img)
            ssim_loss = d_ssim_loss(R_img, img)
            l_col = (1 - lambda_color_ss) * l1_loss + lambda_color_ss * ssim_loss
            
            total_loss += l_col
            gradients = accumulate_gradients(gradients, l_col)
            
        # 2. Depth Distortion Loss (L_DD)
        # Optional, reduces depth ambiguity
        l_dd = compute_depth_distortion_loss(G, rendered_depths)
        total_loss += lambda_dd * l_dd
        gradients = accumulate_gradients(gradients, l_dd)
        
        # 3. Normal Consistency Loss (L_NC)
        # Compare normals from Gaussian rotation (N_i) with normals from depth (N_tilde_i)
        # Note: Source defines L_NC = 1 - N_i^T N_tilde_i
        l_nc = 0
        for R_normal in rendered_normals:
            # N_tilde_i is derived from depth
            N_tilde = normals_from_depth(R_normal) 
            # N_i is derived from Gaussian rotation
            N_i = normals_from_gaussian_rotation(G)
            # Assuming element-wise or pixel-wise comparison averaged
            l_nc += 1 - dot_product(N_i, N_tilde)
        l_nc = l_nc / num_pixels
        total_loss += lambda_nc * l_nc
        gradients = accumulate_gradients(gradients, l_nc)
        
        # 4. Depth Consistency Loss (L_DC)
        # Enforce similarity to monocular depth estimate
        l_dc = 0
        for i, (R_depth, img) in enumerate(zip(rendered_depths, images)):
            # Get monocular depth estimate
            M_depth = mde_net(img)
            # PCC between rendered depth and monocular depth
            pcc = pearson_correlation_coefficient(R_depth, M_depth)
            l_dc += 1 - pcc
        l_dc = l_dc / len(images)
        total_loss += lambda_dc * l_dc
        gradients = accumulate_gradients(gradients, l_dc)
        
        # 5. Local Depth Ranking Loss (L_DR)
        # Prevents geometric collapse
        l_dr = compute_local_depth_ranking_loss(rendered_depths)
        total_loss += lambda_dr * l_dr
        gradients = accumulate_gradients(gradients, l_dr)
        
        # 6. Normal Guided Loss (L_N)
        # Uses reference normals from monocular estimator
        l_n = 0
        for i, (img, R_depth) in enumerate(zip(images, rendered_depths)):
            # Reference normals from monocular normal estimator
            N_ref = normal_net(img)
            # Surface normals from depth
            N_surf = normals_from_depth(R_depth)
            # Rendered normals from Gaussian rotation
            N_rendered = rendered_normals[i]
            
            # L_N = L1(N_ref, N_surf) + 0.1 * L1(N_ref, N_rendered) + TV(N_ref, N_surf)
            term1 = l1_loss(N_ref, N_surf)
            term2 = 0.1 * l1_loss(N_ref, N_rendered)
            term3 = total_variation(N_ref, N_surf)
            
            l_n += term1 + term2 + term3
        l_n = l_n / len(images)
        total_loss += lambda_n * l_n
        gradients = accumulate_gradients(gradients, l_n)
        
        # 7. Diffusion Guidance Loss (L_SDS)
        # Only active after iteration 10000
        if iteration > sds_start_iter:
            l_sds = 0
            for _ in range(num_novel_views):
                # Sample novel viewpoint
                cam_novel = sample_novel_viewpoint(cameras)
                
                # Render geometry
                R_geo = render_2dgs_geometry(G, cam_novel)
                
                # Forward diffusion
                t = random.uniform(0.02, 0.98)
                epsilon = random_normal_like(R_geo)
                alpha_t, sigma_t = get_diffusion_coefficients(t)
                Z_j = alpha_t * R_geo + sigma_t * epsilon
                
                # Predict noise
                epsilon_pred = diffusion_model.predict_noise(Z_j, t)
                
                # SDS Loss
                w_t = get_sds_weight(t)
                loss_term = w_t * squared_norm(epsilon_pred - epsilon)
                l_sds += loss_term
            l_sds = l_sds / num_novel_views
            total_loss += lambda_sds * l_sds
            gradients = accumulate_gradients(gradients, l_sds)
            
        # 8. Outlier Removal Loss (L_BCE)
        if alpha_masks is not None:
            l_bce = 0
            for i, (R_img, alpha_mask_gt) in enumerate(zip(rendered_images, alpha_masks)):
                # Extract alpha channel from rendered image
                alpha_rendered = extract_alpha(R_img)
                l_bce += binary_cross_entropy(alpha_rendered, alpha_mask_gt)
            l_bce = l_bce / len(images)
            total_loss += lambda_bce * l_bce
            gradients = accumulate_gradients(gradients, l_bce)
            
        # Optimization Step
        update_2dgs_parameters(G, gradients)
        
        # Periodic Floater Removal
        if iteration % floater_interval == 0:
            G = knn_floater_removal(G)
            
    return G
```

### Edge Cases and Pitfalls
*   **Early Stage SDS Ineffectiveness:** SDS loss is ineffective when the shape is far from the target. It is disabled for the first 10,000 iterations to prevent instability [S1].
*   **Sparse View Overfitting:** Standard 2DGS losses overfit to captured viewpoints, causing novel-view artifacts. The addition of depth, normal, and diffusion priors mitigates this [S1].
*   **MDE Hallucinations:** Monocular depth estimators may hallucinate or have systematic biases. The use of PCC (relative scale/order) rather than absolute L1 loss helps mitigate this, along with the diffusion prior [S1].
*   **Missing Alpha Masks:** If ground-truth alpha masks are not provided, the BCE loss is skipped, and the model assumes the scene includes the background [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** The optimization runs for 16,000 iterations. The inclusion of SDS loss (after iteration 10,000) adds significant computational cost due to the forward diffusion and noise prediction steps for 100 novel viewpoints per iteration [S1].
*   **Memory:** Memory usage is dominated by the 2DGS primitives and the storage of the pre-trained models (MDE, Normal Estimator, Diffusion Model) in memory [S1].

### Measured Performance
*   **Hardware:** Not specified in the sources [S1].
*   **Resolution:** Not specified in the sources [S1].
*   **Scene:** Tested on sparse multi-view captures (4-16 views) [S1].
*   **Scaling:** The method is designed for sparse views, where traditional methods fail. It scales with the number of Gaussians and the complexity of the diffusion model [S1].

### Strengths and Limitations
*   **Strengths:** Robust geometry reconstruction from sparse views; prevents novel-view artifacts; leverages strong priors from diffusion models for generalization [S1].
*   **Limitations:** Computationally expensive due to diffusion guidance; relies on the quality of pre-trained monocular estimators and diffusion models; requires known camera parameters [S1].

### When to Use
*   Use when reconstructing geometry from a sparse set of RGB images (4-16 views) where standard NeRF or 3DGS methods fail due to overfitting or artifacts [S1].
*   Use when high-fidelity geometry is required for subsequent material and lighting estimation (Stage II) [S1].

### When Not to Use
*   Do not use if dense input views are available, as standard 2DGS may suffice and the additional priors add unnecessary complexity [S1].
*   Do not use if computational resources are limited, as SDS guidance is expensive [S1].

## Background

### Origin
GAINS (Gaussian-based Inverse Rendering from Sparse Multi-view Captures) was introduced by P. Noras et al. in the paper "GAINS: Gaussian-Based Inverse Rendering from Sparse Multi-view Captures" [S1].

### History and Development
The method builds on 2D Gaussian Splatting (2DGS) [8], which was chosen over 3DGS for its more accurate geometry reconstruction, essential for material estimation [S1]. The pipeline adapts 2DGS for inverse rendering by splitting optimization into two stages: Stage I for geometry and Stage II for material and lighting [S1]. Stage I specifically addresses the challenges of sparse views by incorporating priors from latent diffusion models and monocular depth/normal estimation models [S1].

### Where it is Used
GAINS is used for robust and accurate geometry and material property reconstruction from sparse RGB images under known camera intrinsics and extrinsics [S1].

## Sources

[S1] GAINS: Gaussian-Based Inverse Rendering from Sparse Multi-view Captures, https://doi.org/10.1007/978-3-032-37602-2_18, fetched 2026-10-29
