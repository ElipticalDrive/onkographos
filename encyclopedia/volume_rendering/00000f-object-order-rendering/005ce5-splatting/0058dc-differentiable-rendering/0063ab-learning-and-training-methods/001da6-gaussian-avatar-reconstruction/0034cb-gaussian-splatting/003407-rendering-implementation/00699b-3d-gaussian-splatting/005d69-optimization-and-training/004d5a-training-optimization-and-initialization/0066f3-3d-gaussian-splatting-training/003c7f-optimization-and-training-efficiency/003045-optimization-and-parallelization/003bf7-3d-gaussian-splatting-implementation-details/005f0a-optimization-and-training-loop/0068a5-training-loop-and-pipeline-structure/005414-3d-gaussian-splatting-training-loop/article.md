# 3D Gaussian Splatting Training Loop

Also called: 3DGS Optimization, Render-Compare-Correct Loop

## Overview
The 3D Gaussian Splatting (3DGS) training loop is a stochastic gradient descent procedure that minimizes a photometric loss between rendered images and ground-truth photographs to optimize the parameters of a 3D Gaussian point cloud [S2, unofficial]. It distinguishes itself from random search or Monte Carlo methods by computing explicit gradients via a differentiable rasterizer, allowing millions of parameters to be corrected simultaneously and purposefully [S2, unofficial]. The method operates on a fixed iteration budget (typically 30,000 steps) rather than a convergence threshold, periodically densifying and pruning Gaussians to adapt scene density [S2, unofficial][S1].

## How it works
**Inputs:** A set of training images with known camera poses (from Structure-from-Motion), and an initial set of 3D Gaussians (initialized from SfM points or a template model like SMPL) [S2, unofficial][S1].
**Outputs:** Optimized Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients) and potentially updated topology (split/pruned Gaussians) [S2, unofficial][S1].

**Data Structures:**
*   **Gaussian Set:** A list of $N$ Gaussians. Each Gaussian $i$ stores:
    *   Position $\mathbf{p}_i \in \mathbb{R}^3$
    *   Covariance $\Sigma_i$ (or rotation $\mathbf{q}_i$ and scale $\mathbf{s}_i$)
    *   Opacity $\alpha_i \in [0, 1]$
    *   Spherical Harmonics (SH) coefficients $\mathbf{c}_{i, l}$ for color representation
*   **Optimizer State:** Adam optimizer state (first and second moment estimates) for each parameter group [S1].

**Numbered Steps:**
1.  **Sampling:** Randomly select one training image and its corresponding camera pose from the dataset [S2, unofficial].
2.  **Transformation:** Transform canonical Gaussian parameters to the observation space defined by the selected camera pose. If using a parametric avatar (e.g., Human Gaussian Splatting), apply skinning weights from a template (like SMPL) to deform the Gaussians [S1].
3.  **Rendering:** Project the transformed Gaussians to 2D and alpha-blend them using the differentiable Gaussian rasterizer to produce a rendered image $\hat{I}$ [S1][S2, unofficial].
4.  **Loss Computation:** Compute the per-image error $\mathcal{L}_{img}$ by comparing $\hat{I}$ to the ground-truth image $I_{gt}$ [S2, unofficial]. The loss is a weighted sum:
    $$ \mathcal{L}_{img} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM} $$
    where $\mathcal{L}_{L1}$ is the L1 loss (mean absolute difference) and $\mathcal{L}_{D-SSIM}$ is the differentiable Structural Similarity index loss [S2, unofficial][S1]. The weight $\lambda$ is typically 0.2 [S2, unofficial][S1].
5.  **Backpropagation:** Compute gradients $\nabla \mathcal{L}_{img}$ with respect to all Gaussian parameters (position, covariance, opacity, SH coefficients) using automatic differentiation through the rasterizer [S2, unofficial][S1].
6.  **Update:** Update the Gaussian parameters using the Adam optimizer with $\beta_1 = 0.9$ and $\beta_2 = 0.99$ [S1][S2, unofficial].
7.  **Densification/Pruning:** Periodically (e.g., every $k$ iterations), evaluate gradient magnitudes and opacity thresholds to split high-gradient Gaussians or prune low-opacity ones, adapting the density of the representation [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(gaussians, training_views, num_iterations=30000, lambda_ss=0.2):
    optimizer = Adam(gaussians.parameters(), lr=1e-4, betas=(0.9, 0.99))
    
    for iteration in range(num_iterations):
        # 1. Sampling
        view = random.choice(training_views)
        camera_pose = view.pose
        gt_image = view.image
        
        # 2. Transformation (if parametric)
        # For standard 3DGS, this step is implicit in the rasterizer's view transform
        # For avatar 3DGS, apply skinning:
        # gaussians.transform_to_observation_space(camera_pose, skinning_weights)
        
        # 3. Rendering
        rendered_image = gaussian_rasterizer(gaussians, camera_pose)
        
        # 4. Loss Computation
        l1_loss = l1_loss_function(rendered_image, gt_image)
        ssim_loss = d_ssim_loss(rendered_image, gt_image)
        loss = (1 - lambda_ss) * l1_loss + lambda_ss * ssim_loss
        
        # 5. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        
        # 6. Update
        optimizer.step()
        
        # 7. Densification/Pruning (periodic)
        if iteration % densification_interval == 0:
            gaussians.densify_and_prune(iteration)
            
    return gaussians
```

**Edge Cases and Pitfalls:**
*   **Precision:** The differentiable rasterizer must handle depth sorting and alpha blending correctly to ensure valid gradients [S1].
*   **Boundaries:** Background pixels should be handled explicitly (e.g., setting background to black in ground truth) to avoid misleading gradients [S1].
*   **Degenerate Input:** If the initial SfM point cloud is sparse, the initial render may be poor, requiring careful initialization or densification strategies [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Each iteration involves a forward pass through the rasterizer ($O(N)$ where $N$ is the number of Gaussians) and a backward pass. The total time is proportional to the number of iterations (typically 30,000) [S2, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians and their parameters (position, covariance, opacity, SH coefficients). Densification increases memory usage over time [S1][S2, unofficial].
*   **Measured Performance:** Training is performed on GPUs using PyTorch. Evaluation metrics include PSNR, SSIM, and LPIPS on held-out views [S1].
*   **Strengths:** Efficient gradient-based optimization allows for high-quality reconstruction from sparse inputs. The fixed iteration budget provides predictable training time [S2, unofficial].
*   **Limitations:** The method is sensitive to the initial SfM reconstruction quality. Densification heuristics can lead to over-parameterization if not tuned correctly [S1][S2, unofficial].
*   **When to use:** Suitable for static or dynamic scene reconstruction where high-fidelity novel view synthesis is required [S1][S2, unofficial].
*   **When not to use:** Not suitable for real-time training on resource-constrained devices due to the iterative nature and memory requirements [S2, unofficial].

## Background
The 3D Gaussian Splatting training loop was introduced in the original 3DGS paper [S1] and popularized by subsequent tutorials and implementations [S2, unofficial]. It builds on the differentiable rendering framework established by earlier work [S1]. The method has been extended to dynamic scenes, such as Human Gaussian Splatting, which integrates SMPL models for animatable avatars [S1].

## Sources
[S1] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
[S2] Gaussian Splats Part 2: From Photos to a Trained Model, https://alphapixeldev.com/gaussian-splats-part-2-creating-gaussian-splats/, fetched 2026-10-29 (unofficial)
