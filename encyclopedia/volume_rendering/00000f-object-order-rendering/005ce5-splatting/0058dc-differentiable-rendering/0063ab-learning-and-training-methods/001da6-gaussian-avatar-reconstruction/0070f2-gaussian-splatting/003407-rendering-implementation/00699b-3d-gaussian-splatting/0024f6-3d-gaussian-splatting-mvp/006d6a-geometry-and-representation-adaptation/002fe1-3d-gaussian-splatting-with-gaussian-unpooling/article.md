# 3D Gaussian Splatting with Gaussian Unpooling

Also called: FSGS

## Overview
FSGS (First Sparse Gaussian Splatting) is a method for reconstructing 3D scenes from sparse view inputs, addressing the challenge of limited camera poses where standard 3D Gaussian Splatting fails due to insufficient geometric constraints [S1]. It distinguishes itself from sibling methods like [3D Gaussian Splatting with Mesh UV Parameterization](8046-3d-gaussian-splatting-with-mesh-uv-parameterization.md) and [H-2DGS (Height-Field 2D Gaussian Splatting)](21287-h-2dgs-height-field-2d-gaussian-splatting.md) by dynamically expanding the Gaussian representation via unpooling from sparse Structure-from-Motion (SfM) seeds, rather than relying on static surface parameterizations or height fields [S1].

## How it works
**Inputs:**
*   Sparse set of input images with known camera poses (typically derived from SfM).
*   A pre-trained 2D monocular depth estimation network.

**Outputs:**
*   A set of 3D Gaussians $\{G_i\}$ optimized to represent the scene geometry and appearance.

**Data Structures:**
*   **Gaussian Set:** A list of $N$ Gaussians. Each Gaussian $G_i$ is defined by mean $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (or rotation $R_i$ and scale $S_i$), opacity $\alpha_i \in [0,1]$, and spherical harmonics coefficients for color.
*   **SfM Points:** A sparse set of 3D points $\{P_j\}$ with associated image observations, serving as the initialization seeds.

**Steps:**
1.  **Initialization:** Initialize the Gaussian set using the sparse 3D points obtained from Structure-from-Motion (SfM) on the input images [S1]. These points serve as the initial means $\mu_i$ and provide initial covariance estimates based on local point cloud density or fixed small scales.
2.  **Gaussian Unpooling:** Identify and expand the sparse Gaussians to cover the scene volume. This involves "unpooling" existing Gaussians, which typically entails duplicating or splitting Gaussians in regions where the initial sparse representation is insufficient to capture fine details or occluded surfaces. The exact mathematical operation for unpooling is not detailed in the provided sources, but it serves to densify the representation from the sparse SfM seeds [S1].
3.  **Depth Supervision:** Render depth images from the current Gaussian set using differentiable volume rendering. Pass the input images through the pre-trained 2D depth estimation network to obtain predicted depth maps. Compute a loss between the rendered depth and the predicted depth [S1].
4.  **Optimization:** Optimize the Gaussian parameters (means, covariances, opacities, colors) using a combined loss function:
    $$ \mathcal{L} = \mathcal{L}_{color} + \lambda \mathcal{L}_{depth} $$
    where $\mathcal{L}_{color}$ is the standard photometric loss (e.g., L1 or LPIPS) between rendered and input images, and $\mathcal{L}_{depth}$ is the consistency loss between rendered and predicted depth maps [S1]. The parameter $\lambda$ balances the two terms.
5.  **Pruning:** Remove Gaussians with incorrect depth values or low opacity to maintain efficiency and reduce noise [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_fsgs(input_images, camera_poses, depth_net):
    # Step 1: Initialize from SfM
    sfm_points = run_sfm(input_images, camera_poses)
    gaussians = initialize_gaussians_from_sfm(sfm_points)
    
    # Step 2 & 4: Optimization Loop
    for step in range(num_steps):
        # Sample a random input image
        img_idx = random_choice(len(input_images))
        img = input_images[img_idx]
        cam = camera_poses[img_idx]
        
        # Render current Gaussians
        rendered_img, rendered_depth = render(gaussians, cam)
        
        # Get predicted depth from pre-trained network
        predicted_depth = depth_net(img)
        
        # Compute losses
        loss_color = compute_color_loss(rendered_img, img)
        loss_depth = compute_depth_loss(rendered_depth, predicted_depth)
        
        # Combined loss
        loss = loss_color + lambda_depth * loss_depth
        
        # Backpropagate and update Gaussians
        update_gaussians(gaussians, loss)
        
        # Step 5: Prune invalid Gaussians
        gaussians = prune_gaussians(gaussians, rendered_depth, predicted_depth)
        
        # Step 2: Unpool if necessary (implementation detail not specified)
        if should_unpool(gaussians):
            gaussians = unpool_gaussians(gaussians)
            
    return gaussians
```

**Parameters:**
*   $\lambda$: Weight for depth loss. Not given in the sources.
*   `num_steps`: Number of optimization iterations. Not given in the sources.

**Edge Cases:**
*   **Sparse Input:** If the SfM reconstruction is too sparse or fails, initialization may be poor. The method relies on the robustness of the pre-trained depth network to fill gaps [S1].
*   **Depth Ambiguity:** Monocular depth estimation can be ambiguous for textureless or reflective surfaces. The depth loss may introduce artifacts if not balanced correctly with photometric loss [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and optimization steps. Unpooling increases the number of Gaussians, potentially increasing rendering and optimization time compared to standard 3DGS.
*   **Memory Complexity:** Scales with the number of Gaussians. Unpooling increases memory usage.
*   **Strengths:** Enables high-quality reconstruction from very few views (sparse input) where standard 3DGS fails. Leverages strong priors from pre-trained 2D depth networks.
*   **Limitations:** Relies on the accuracy of the pre-trained depth network. Performance may degrade if the depth predictions are significantly incorrect. Unpooling strategy is critical and not fully detailed, potentially leading to over-smoothing or noise if not tuned.
*   **When to use:** Sparse-view reconstruction scenarios.
*   **When not to use:** Dense-view scenarios where standard 3DGS is sufficient and faster.

## Background
FSGS was introduced as the first method to explore reconstructing 3D scenes from sparse view input using Gaussian Splatting [S1]. It builds on the core differentiable rasterization pipeline of 3D Gaussian Splatting [3410-3d-gaussian-splatting-mvp.md] but modifies the representation to handle sparse data by initializing from SfM and using depth supervision [S1]. It is part of a family of methods including SparseGS, CoherentGS, and DNGaussian that target sparse-view reconstruction [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
