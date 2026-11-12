# Principled Uncertainty Pruning

Also called: Fisher Information-based pruning score, PUP 3D-GS, PUP 3D-GS Pruning Score & Pipeline, Principled Uncertainty Pruning for 3D Gaussian Splatting

## Overview
Principled Uncertainty Pruning (PUP) is a post-hoc method for compressing pretrained 3D Gaussian Splatting (3D-GS) models by removing Gaussians with high uncertainty in their spatial parameters. It solves the problem of visual fidelity degradation at high compression ratios by using a principled sensitivity score derived from the Fisher Information matrix of the reconstruction error. Unlike heuristic methods or those relying on color visibility, PUP isolates the uncertainty of mean and scale parameters, enabling a multi-round prune-refine pipeline that can remove up to 90% of Gaussians while preserving fine details and accelerating rendering.

## How it works
**Inputs:** A converged 3D-GS model $\mathcal{G}$ (set of $N$ Gaussians), a set of training views $\mathcal{P}_{gt}$ with ground truth images $I_{gt}$, and camera poses $\Phi$.
**Outputs:** A pruned set of Gaussians $\mathcal{G}'$ and a fine-tuned model.

**Data Structures:**
*   **Gaussian Parameters:** Each Gaussian $g_k$ has parameters $\theta_k = [\mu_k, \sigma_k]$, where $\mu_k \in \mathbb{R}^3$ is the mean (location) and $\sigma_k \in \mathbb{R}^3$ is the scale. Rotations and RGB colors are excluded from the sensitivity calculation [S1].
*   **Fisher Matrix:** For each Gaussian, a $6 \times 6$ block-diagonal Fisher Information matrix $F_k$ is computed, corresponding to the 6 spatial parameters (3 for mean, 3 for scale) [S1].
*   **Pruning Score:** A scalar score $s_k = \log \det(F_k)$ for each Gaussian [S1].

**Numbered Steps:**

1.  **Compute L2 Loss:** Calculate the total L2 reconstruction error over all training views:
    $$L_2 = \frac{1}{2} \sum_{\phi \in \mathcal{P}_{gt}} \|I_{\mathcal{G}}(\phi) - I_{gt}\|_2^2$$
    where $I_{\mathcal{G}}(\phi)$ is the rendered image for pose $\phi$ [S1].

2.  **Approximate Hessian via Fisher Information:**
    The exact Hessian of the L2 loss is:
    $$\nabla_G^2 L_2 = \sum_{\phi \in \mathcal{P}_{gt}} \nabla_G I_G(\phi) \nabla_G I_G(\phi)^T + (I_G(\phi) - I_{gt}) \nabla_G^2 I_G(\phi)$$
    Since the model is converged, the residual term $(I_G(\phi) - I_{gt})$ approaches zero, causing the second-order term to vanish. The Hessian is approximated by the Fisher Information matrix:
    $$\nabla_G^2 L_2 \approx \sum_{\phi \in \mathcal{P}_{gt}} \nabla_G I_G(\phi) \nabla_G I_G(\phi)^T$$
    This approximation relies on the assumption that the L1 residual is small [S1].

3.  **Compute Block-wise Fisher Matrix:**
    Compute the Fisher Information matrix for each Gaussian $g_k$ using only its mean and scale parameters. This results in a $6 \times 6$ matrix $F_k$.
    *   *Note:* Including rotation parameters expands this to $10 \times 10$, increasing memory usage by $2.78\times$ and causing out-of-memory errors on 16GB GPUs. Including RGB parameters degrades quality as it correlates with visibility rather than error sensitivity [S1].

4.  **Calculate Pruning Score:**
    For each Gaussian, compute the sensitivity score as the log determinant of its Fisher matrix:
    $$s_k = \log \det(F_k)$$
    This score represents the entropy of the second-order Laplace approximation of the posterior distribution for that Gaussian's spatial parameters [S1].

5.  **Compute Sensitivity over Patches (Optional Optimization):**
    To reduce computation, the sensitivity score can be computed over image patches of size $2\times2$, $4\times4$, or $8\times8$ pixels. The default and recommended patch size is $4\times4$ [S1].

6.  **Multi-Round Prune-Refine Pipeline:**
    Execute the following rounds until the target pruning percentage is reached:
    *   **Prune:** Remove the Gaussians with the highest pruning scores $s_k$.
        *   *Round 1:* Prune 80% of Gaussians [S1].
        *   *Round 2:* Prune 50% of the *remaining* Gaussians [S1].
    *   **Refine:** Fine-tune the remaining Gaussians for a set number of iterations (e.g., 5,000 steps per round) to recover quality and reduce the residual [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def principled_uncertainty_pruning(gaussians, training_views, num_rounds=2, 
                                   prune_pct_round1=0.8, prune_pct_round2=0.5, 
                                   fine_tune_iters=5000):
    """
    gaussians: List of Gaussian objects with attributes .mean, .scale, .render()
    training_views: List of (pose, ground_truth_image) tuples
    """
    current_gaussians = gaussians
    
    for round_idx in range(num_rounds):
        # Step 1 & 2: Compute Fisher Information Matrix for each Gaussian
        # We approximate the Hessian of the L2 loss using the Fisher approximation
        # F_k = sum over views of (gradient of rendered image w.r.t spatial params)^2
        
        scores = []
        for g in current_gaussians:
            # Initialize 6x6 Fisher matrix for mean (3) and scale (3)
            fisher_matrix = zeros((6, 6))
            
            for pose, gt_img in training_views:
                # Render current Gaussian set to get image
                # Note: In practice, gradients are computed via automatic differentiation
                # or analytical gradients of the splatting function
                rendered_img = render(current_gaussians, pose)
                
                # Compute gradient of rendered image w.r.t g.mean and g.scale
                # This is a vector of size 6 (3 for mean, 3 for scale)
                # Let grad_img be the gradient of the L2 loss w.r.t the image pixels
                # We need the Jacobian of the rendering function w.r.t spatial params
                # J = d(rendered_img) / d(g.spatial_params)
                
                # Approximation: Use backpropagation to get d(L2) / d(g.spatial_params)
                # L2 = 0.5 * sum((rendered_img - gt_img)^2)
                # grad_spatial = d(L2)/d(g.spatial_params)
                
                grad_spatial = compute_gradient_spatial(g, pose, gt_img)
                
                # Outer product to accumulate Fisher Information
                fisher_matrix += outer_product(grad_spatial, grad_spatial)
            
            # Step 3 & 4: Compute Pruning Score
            # Log determinant of the Fisher matrix
            score = log_det(fisher_matrix)
            scores.append(score)
        
        # Step 5: Sort Gaussians by score (highest uncertainty first)
        indexed_scores = list(enumerate(scores))
        indexed_scores.sort(key=lambda x: x[1], reverse=True)
        
        # Step 6: Prune
        if round_idx == 0:
            num_to_prune = int(len(current_gaussians) * prune_pct_round1)
        else:
            num_to_prune = int(len(current_gaussians) * prune_pct_round2)
            
        # Identify indices to keep
        keep_indices = [idx for idx, _ in indexed_scores[num_to_prune:]]
        
        # Remove pruned Gaussians
        current_gaussians = [current_gaussians[i] for i in keep_indices]
        
        # Step 7: Refine (Fine-tune)
        # This step involves running the standard 3D-GS optimization loop
        # on the remaining Gaussians for fine_tune_iters steps.
        # The optimizer updates mean, scale, opacity, rotation, and SH coefficients.
        current_gaussians = fine_tune(current_gaussians, training_views, fine_tune_iters)
        
    return current_gaussians
```

**Parameters:**
*   **Pruning Percentages:** 80% in Round 1, 50% in Round 2 [S1].
*   **Fine-tuning Iterations:** 5,000 steps per round [S1].
*   **Patch Size:** $4\times4$ pixels (default) [S1].

**Edge Cases and Pitfalls:**
*   **Convergence Requirement:** The method assumes the scene is fully converged. If the L1 residual is large, the Fisher approximation is inaccurate, and pruning too aggressively before convergence degrades quality [S1].
*   **Memory Constraints:** Including rotation parameters increases the Fisher matrix size to $10\times10$, requiring $2.78\times$ more memory. This can cause out-of-memory errors on 16GB GPUs (e.g., RTX A4000) [S1].
*   **RGB Parameters:** Using RGB color parameters for the sensitivity score degrades image quality because it correlates with visibility rather than reconstruction error sensitivity [S1].

## Operating characteristics
*   **Time Complexity:** The computation of the Fisher matrix scales with the number of Gaussians $N$ and the number of training views. The memory requirement scales linearly with $N$, specifically proportional to $N \times 36$ bytes (for the $6\times6$ matrix) [S1].
*   **Measured Performance:** On the Deep Blending playroom scene, PUP prunes from 2.65M to 0.265M Gaussians, accelerating rendering from 76.65 FPS to 318.06 FPS (4.15x speed-up) [S1]. On average across Mip-NeRF 360, Tanks & Temples, and Deep Blending, it increases rendering speed by 3.56x after pruning 90% of Gaussians [S1].
*   **Strengths:** Preserves fine details and foreground information better than heuristic methods like LightGaussian. Achieves higher image quality metrics (PSNR, SSIM, LPIPS) at ~90% pruning [S1].
*   **Limitations:** Requires a converged model. Memory usage for the Fisher matrix can be high for very large scenes, though it fits on 16GB GPUs with the $6\times6$ approximation [S1].

## Background
Principled Uncertainty Pruning (PUP) was introduced by Alex Hanson, Allen Tu, Vasu Singla, Mayuka Jayawardhana, Matthias Zwicker, and Tom Goldstein from the University of Maryland, College Park. It was published in 2025 at the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR) [S1]. The method builds on the concept of using Fisher Information for uncertainty estimation in neural networks and applies it specifically to the spatial parameters of 3D Gaussian Splatting. It is designed as a post-hoc pipeline, meaning it can be applied to any pretrained 3D-GS model without modifying the training process [S1].

## Sources
[S1] PUP 3D-GS: Principled Uncertainty Pruning for 3D Gaussian Splatting, https://doi.org/10.1109/CVPR52734.2025.00558, fetched 2026-10-29
