# Principled Uncertainty Pruning

Also called: Fisher Information-based pruning score, PUP 3D-GS, PUP 3D-GS Pruning Score & Pipeline, Principled Uncertainty Pruning for 3D Gaussian Splatting

## Overview
Principled Uncertainty Pruning (PUP) is a post-hoc method for compressing pretrained 3D Gaussian Splatting (3D-GS) models by removing Gaussians with high uncertainty in their spatial parameters [S1]. It solves the problem of visual fidelity degradation at high compression ratios by using a principled sensitivity score derived from the Fisher Information matrix of the reconstruction error [S1]. Unlike heuristic methods that rely on opacity or gradient magnitude, PUP quantifies the entropy of the posterior distribution over each Gaussian's position and scale, allowing for aggressive pruning (up to 90%) while preserving fine details and foreground information [S1].

## How it works
PUP operates as a multi-round prune-refine pipeline applied to a converged 3D-GS model [S1]. It does not modify the training pipeline but instead evaluates a sensitivity score for each Gaussian to determine which ones to remove [S1].

### Inputs and Outputs
*   **Input:** A converged 3D-GS model $\mathcal{G} = \{g_1, \dots, g_N\}$, where each Gaussian $g_k$ has mean $\mu_k$, scale $s_k$, rotation $q_k$, opacity $\alpha_k$, and spherical harmonic coefficients $\mathbf{c}_k$ [S1]. A set of ground truth training images $\mathcal{I}$ with corresponding camera poses $\Phi$ [S1].
*   **Output:** A pruned set of Gaussians $\mathcal{G}'$ with reduced cardinality [S1].

### Data Structures
The method computes a $6 \times 6$ Fisher Information matrix block for each Gaussian, corresponding to its 3D mean position ($\mu_x, \mu_y, \mu_z$) and 3D scale parameters ($s_x, s_y, s_z$) [S1].
*   **Fisher Matrix ($F_k$):** A $6 \times 6$ symmetric positive semi-definite matrix for Gaussian $g_k$ [S1].
*   **Pruning Score ($S_k$):** A scalar value derived from $F_k$ [S1].

### Parameters
*   **Patch Size:** The sensitivity score is computed over image patches. Default is $4 \times 4$ pixels. Alternatives $2 \times 2$ and $8 \times 8$ are also supported [S1].
*   **Pruning Percentages:** Round 1 prunes 80% of Gaussians [S1]. Round 2 prunes 50% of the *remaining* Gaussians [S1]. Total pruning is ~90% [S1].
*   **Fine-tuning Iterations:** 5,000 iterations per round, totaling 10,000 iterations [S1].

### Method Steps

1.  **Compute L2 Loss:** Calculate the L2 reconstruction error over the training views:
    $$L_2 = \frac{1}{2} \sum_{\phi \in \Phi} \|I_{\mathcal{G}}(\phi) - I_{gt}(\phi)\|_2^2$$
    where $I_{\mathcal{G}}(\phi)$ is the rendered image for pose $\phi$ using Gaussians $\mathcal{G}$, and $I_{gt}(\phi)$ is the ground truth image [S1].

2.  **Approximate Hessian via Fisher Information:**
    The exact Hessian of the L2 loss with respect to Gaussian parameters $\mathcal{G}$ is:
    $$\nabla_{\mathcal{G}}^2 L_2 = \sum_{\phi \in \Phi} \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi) \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi)^T + (I_{\mathcal{G}}(\phi) - I_{gt}(\phi)) \nabla_{\mathcal{G}}^2 I_{\mathcal{G}}(\phi)$$
    Since the model is converged, the residual term $(I_{\mathcal{G}}(\phi) - I_{gt}(\phi))$ approaches zero, causing the second-order term to vanish [S1]. The Hessian is approximated by the Fisher Information matrix:
    $$F \approx \sum_{\phi \in \Phi} \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi) \nabla_{\mathcal{G}} I_{\mathcal{G}}(\phi)^T$$
    [S1].

3.  **Compute Block-wise Fisher Matrix:**
    To reduce memory usage and focus on geometric uncertainty, restrict the Fisher matrix to the spatial mean and scale parameters only (6 parameters per Gaussian) [S1]. Exclude rotation and RGB color parameters [S1].
    *   For each Gaussian $g_k$, compute the $6 \times 6$ block $F_k$ of the global Fisher matrix.
    *   The memory requirement scales linearly with the number of Gaussians $N$, specifically $N \times 36$ elements [S1].

4.  **Calculate Pruning Score:**
    The pruning score $S_k$ for Gaussian $g_k$ is the log determinant of its Fisher block:
    $$S_k = \log \det(F_k)$$
    This score represents the entropy of the second-order Laplace approximation of the posterior distribution for that Gaussian [S1]. Higher entropy indicates higher uncertainty, making it a candidate for pruning [S1].

5.  **Multi-Round Prune-Refine Pipeline:**
    *   **Round 1:**
        1.  Sort all Gaussians by $S_k$ in descending order.
        2.  Remove the top 80% of Gaussians (highest uncertainty) [S1].
        3.  Fine-tune the remaining Gaussians for 5,000 iterations using standard 3D-GS optimization to recover quality [S1].
    *   **Round 2:**
        1.  Recompute the Fisher matrices and pruning scores for the remaining Gaussians [S1].
        2.  Remove the top 50% of the remaining Gaussians [S1].
        3.  Fine-tune for another 5,000 iterations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def principled_uncertainty_pruning(gaussians, training_images, training_poses, 
                                   patch_size=4, round1_prune_pct=0.8, 
                                   round2_prune_pct=0.5, fine_tune_iters=5000):
    """
    Performs Principled Uncertainty Pruning on a 3D-GS model.
    """
    def extract_patches(tensor, patch_size):
        # tensor shape: (H, W, C, P) where P=6 params
        # Returns shape: (num_patches, patch_size*patch_size*C, P)
        H, W, C, P = tensor.shape
        patches = tensor.reshape(H // patch_size, patch_size, W // patch_size, patch_size, C, P)
        patches = patches.transpose(0, 2, 1, 3, 4, 5).reshape(-1, patch_size * patch_size * C, P)
        return patches

    def compute_fisher_block(g, poses, images, patch_size):
        n_params = 6
        fisher_block = np.zeros((n_params, n_params))
        for pose, gt_img in zip(poses, images):
            # Requires differentiable rasterizer backward pass to compute dI_G/d_params
            grad_img_params = compute_gradient_of_rendered_image_wrt_spatial_params(g, pose)
            grad_patches = extract_patches(grad_img_params, patch_size)
            for patch_grad in grad_patches:
                fisher_block += patch_grad.T @ patch_grad
        return fisher_block

    scores = []
    for g in gaussians:
        F_k = compute_fisher_block(g, training_poses, training_images, patch_size)
        epsilon = 1e-6
        F_k_reg = F_k + epsilon * np.eye(6)
        try:
            log_det = np.log(np.linalg.det(F_k_reg))
        except np.linalg.LinAlgError:
            log_det = -np.inf
        scores.append(log_det)
    scores = np.array(scores)

    # Round 1
    n_gaussians = len(gaussians)
    n_prune_round1 = int(n_gaussians * round1_prune_pct)
    indices_to_prune_round1 = np.argsort(scores)[-n_prune_round1:]
    active_mask = np.ones(n_gaussians, dtype=bool)
    active_mask[indices_to_prune_round1] = False
    active_gaussians = [g for i, g in enumerate(gaussians) if active_mask[i]]
    # Standard optimizer loop (e.g., Adam) on remaining Gaussians for fine_tune_iters steps
    active_gaussians = fine_tune_gaussians(active_gaussians, training_poses, training_images, fine_tune_iters)

    # Round 2
    active_scores = []
    for g in active_gaussians:
        F_k = compute_fisher_block(g, training_poses, training_images, patch_size)
        F_k_reg = F_k + epsilon * np.eye(6)
        try:
            log_det = np.log(np.linalg.det(F_k_reg))
        except np.linalg.LinAlgError:
            log_det = -np.inf
        active_scores.append(log_det)
    active_scores = np.array(active_scores)
    n_active = len(active_gaussians)
    n_prune_round2 = int(n_active * round2_prune_pct)
    indices_to_prune_round2 = np.argsort(active_scores)[-n_prune_round2:]
    active_mask_round2 = np.ones(n_active, dtype=bool)
    active_mask_round2[indices_to_prune_round2] = False
    final_gaussians = [g for i, g in enumerate(active_gaussians) if active_mask_round2[i]]
    final_gaussians = fine_tune_gaussians(final_gaussians, training_poses, training_images, fine_tune_iters)
    return final_gaussians

def compute_gradient_of_rendered_image_wrt_spatial_params(g, pose):
    # Requires differentiable rasterizer backward pass to compute dI_G/d_params
    # Returns array of shape (H, W, 3, 6)
    pass

def fine_tune_gaussians(gaussians, poses, images, iterations):
    # Standard optimizer loop (e.g., Adam) on remaining Gaussians for iterations steps
    pass
```

### Edge Cases and Pitfalls
*   **Convergence Requirement:** The method assumes the scene is fully converged. The Fisher approximation relies on the residual $(I_G - I_{gt})$ being small. Pruning before convergence degrades the approximation and final quality [S1].
*   **Memory Constraints:** Including rotation parameters expands the Fisher matrix to $10 \times 10$, increasing memory usage by 2.78×. This can cause out-of-memory errors on 16GB GPUs (e.g., RTX A4000). Excluding rotations is critical for standard hardware [S1].
*   **RGB Parameters:** Using RGB color parameters for the sensitivity score degrades image quality. The RGB score correlates with visibility rather than reconstruction error sensitivity [S1].
*   **Patch Size:** While $4 \times 4$ is the default, $2 \times 2$ and $8 \times 8$ patches produce similar results, though $4 \times 4$ yields slightly better metrics [S1].

## Operating characteristics
*   **Time Complexity:** The dominant cost is computing the Fisher matrices, which scales linearly with the number of Gaussians $N$ and the number of training views. The fine-tuning steps add additional time proportional to the number of iterations [S1].
*   **Memory Complexity:** Memory usage for Fisher matrices is proportional to $N \times 36$ elements (for the $6 \times 6$ block) [S1]. This fits within 16GB GPUs for typical scene sizes [S1].
*   **Performance:** On the Mip-NeRF 360 dataset, PUP achieves an average PSNR of 26.67, SSIM of 0.7862, and LPIPS of 0.2719 after 90% pruning, with an average FPS of 204.81 on an RTX A4000 [S1]. On the Deep Blending playroom scene, it accelerates rendering from 76.65 FPS to 318.06 FPS (4.15× speed-up) [S1].
*   **Strengths:** Preserves fine details and foreground information better than heuristic methods like LightGaussian. Achieves higher compression ratios (up to 90%) with less quality degradation [S1].
*   **Limitations:** Requires a converged model. Not suitable for online or incremental training scenarios where the model is not yet stable [S1].

## Background
Principled Uncertainty Pruning (PUP) was introduced by Alex Hanson, Allen Tu, Vasu Singla, Mayuka Jayawardhana, Matthias Zwicker, and Tom Goldstein from the University of Maryland, College Park [S1]. The method addresses the limitations of previous pruning techniques that relied on heuristics, which often failed to preserve visual fidelity at high compression ratios [S1]. PUP builds on the concept of Fisher Information as a measure of parameter sensitivity, adapting it to the specific structure of 3D Gaussian Splatting [S1].

## Sources
[S1] PUP 3D-GS: Principled Uncertainty Pruning for 3D Gaussian Splatting, https://doi.org/10.1109/CVPR52734.2025.00558, fetched 2026-10-29
