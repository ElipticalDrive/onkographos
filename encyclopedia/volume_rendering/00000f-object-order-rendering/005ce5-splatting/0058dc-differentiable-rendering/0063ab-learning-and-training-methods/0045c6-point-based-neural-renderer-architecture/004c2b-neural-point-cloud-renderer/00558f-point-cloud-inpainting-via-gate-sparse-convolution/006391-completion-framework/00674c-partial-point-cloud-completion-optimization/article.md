# Partial Point Cloud Completion Optimization

Also called: Shape completion, free-space optimization

## Overview
This method optimizes an object-level latent code to reconstruct a complete 3D shape from a partial point cloud, using a pre-trained generative prior (ObjectNet) to infer missing geometry, followed by a patch-level refinement stage to recover high-frequency details [S1]. It distinguishes itself from global implicit field methods by decoupling the coarse shape completion (via the object prior) from local detail recovery (via patch latent codes), allowing for generalizable completion without per-scene network training [S1].

## How it works
**Inputs:**
*   Partial point cloud derived from depth maps at fixed and per-scene random viewpoints [S1].
*   Pre-trained ObjectNet (trained on the target category) and PatchNet (pre-trained on a different category, kept fixed) [S1].
*   Initial object latent code estimate [S1].

**Outputs:**
*   Optimized object latent code $z_{obj}$ and patch latent codes $z_{patch}$ representing the completed 3D shape [S1].

**Data Structures:**
*   **Object Latent Code ($z_{obj}$):** A vector sampled from a multivariate Gaussian fitted to object latent codes obtained during ObjectNet training [S1].
*   **Patch Latent Codes ($z_{patch}$):** Intermediate variables representing local patches of the shape, regressed during optimization [S1].
*   **Extrinsics:** Camera parameters associated with the input depth maps [S1].

**Formulas and Losses:**
1.  **Reconstruction Loss:** Measures the discrepancy between the rendered output and the input partial point cloud [S1].
2.  **L2 Regularizer:** Applied to the latent codes to prevent overfitting [S1].
3.  **Coverage Loss:** Ensures the generated shape covers the visible region of the input [S1].
4.  **Free-Space Loss:** Encourages the network to regress positive values for samples located between the surface and the camera. This is applied to 30% of the sampled points [S1].
5.  **Extrinsics Losses:** Explicitly excluded during optimization to avoid detrimental effects on patches outside the partial point cloud [S1].

**Algorithm Steps:**
1.  **Initialization:** Compute the average of the object latent codes obtained during ObjectNet training. Initialize $z_{obj}$ with this average [S1].
2.  **Global Optimization (600 iterations):**
    *   Sample 8,000 points per iteration [S1].
    *   Optimize $z_{obj}$ and intermediate variables (including $z_{patch}$ and extrinsics) to minimize the sum of Reconstruction Loss, L2 Regularizer, Coverage Loss, and Free-Space Loss [S1].
    *   Update learning rate: Start at $\eta = 0.01$. Halve $\eta$ every 200 iterations [S1].
3.  **Local Refinement (Last 100 iterations):**
    *   Fix the extrinsics [S1].
    *   Optimize only the patch latent codes $z_{patch}$ [S1].
    *   Use a learning rate of $\eta = 0.001$ [S1].
    *   This step integrates details not captured by the object-level prior [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partial_point_cloud_completion(partial_point_cloud, object_net, patch_net):
    # 1. Initialization
    # Fit multivariate Gaussian to object latent codes from training
    # Sample initial z_obj from the fitted Gaussian, or use the mean
    z_obj = mean(object_latent_codes_from_training)
    
    # Initialize patch latent codes and extrinsics as intermediate variables
    z_patch = initialize_patch_latents()
    extrinsics = initialize_extrinsics()
    
    # 2. Global Optimization
    learning_rate = 0.01
    num_global_iters = 600
    lr_decay_step = 200
    
    for i in range(num_global_iters):
        # Decay learning rate
        if i > 0 and i % lr_decay_step == 0:
            learning_rate /= 2.0
            
        # Sample 8k points
        samples = sample_points(count=8000)
        
        # Compute losses
        # Reconstruction loss between rendered output and partial_point_cloud
        loss_recon = compute_reconstruction_loss(samples, z_obj, z_patch, extrinsics, object_net, patch_net)
        
        # L2 regularizer on latent codes
        loss_reg = l2_regularizer(z_obj, z_patch)
        
        # Coverage loss
        loss_cov = compute_coverage_loss(samples, z_obj, z_patch, extrinsics)
        
        # Free-space loss (30% of samples)
        free_space_ratio = 0.3
        free_space_samples = select_random_subset(samples, free_space_ratio)
        loss_free = compute_free_space_loss(free_space_samples, z_obj, z_patch, extrinsics)
        
        # Total loss (exclude extrinsics losses)
        total_loss = loss_recon + loss_reg + loss_cov + loss_free
        
        # Backpropagate and update z_obj, z_patch, extrinsics
        update_variables(z_obj, z_patch, extrinsics, total_loss, learning_rate)
        
    # 3. Local Refinement
    # Fix extrinsics
    extrinsics.requires_grad = False
    
    refinement_lr = 0.001
    num_refinement_iters = 100
    
    for i in range(num_refinement_iters):
        # Sample points (implementation detail: reuse or resample)
        samples = sample_points(count=8000)
        
        # Compute losses
        loss_recon = compute_reconstruction_loss(samples, z_obj, z_patch, extrinsics, object_net, patch_net)
        loss_reg = l2_regularizer(z_obj, z_patch)
        loss_cov = compute_coverage_loss(samples, z_obj, z_patch, extrinsics)
        
        free_space_samples = select_random_subset(samples, free_space_ratio)
        loss_free = compute_free_space_loss(free_space_samples, z_obj, z_patch, extrinsics)
        
        total_loss = loss_recon + loss_reg + loss_cov + loss_free
        
        # Backpropagate and update ONLY z_patch
        update_variables(z_patch, total_loss, refinement_lr)
        
    return z_obj, z_patch
```

**Edge Cases and Pitfalls:**
*   **Extrinsics Losses:** Including extrinsics losses during optimization has a detrimental effect on patches outside the partial point cloud; they must be excluded [S1].
*   **PatchNet Pre-training:** PatchNet is pre-trained on a different object category and kept fixed, making the task more challenging than if it were category-specific [S1].
*   **Baseline Comparison:** Baselines use versions where the eight final layers are pre-trained on all categories and fine-tuned on the target shape category [S1].

## Operating characteristics
*   **Performance:** On the ShapeNet dataset, the method achieves higher F-scores than baselines (DeepSDF and other baselines) across sofas and airplanes, both fixed and random viewpoints. For example, on sofas (fixed viewpoint), it achieves an accuracy of 0.091 and F-score of 48.1, improving to 0.053 accuracy and 52.4 F-score after refinement (Note: The source text lists "0.052 / 53.6" for "ours+refined" in the notes, but Table 5 in the source text shows "0.053 / 52.4" for sofas random and "0.052 / 53.6" is not explicitly in the provided Table 5 snippet for sofas fixed. The notes say "sofas fixed 0.091 / 48.1 (ours), 0.052 / 53.6 (ours+refined)". Table 5 snippet shows "ours" as 0.091/48.1. The refined values are not fully visible in the snippet but the notes provide them. We rely on the notes for the refined values).
*   **Complexity:** The method requires 600 iterations of global optimization and 100 iterations of local refinement, using 8,000 samples per iteration [S1].
*   **Strengths:** The two-stage approach allows the object prior to handle global shape completion while patch-level refinement captures local details [S1].
*   **Limitations:** The method relies on a pre-trained ObjectNet and PatchNet, limiting its applicability to categories covered by these networks [S1].

## Background
The method is part of the PatchNets framework, which introduces patch-based generalizable deep implicit 3D shape representations [S1]. It builds upon the Completion Framework, which uses two encoders for existing and missing parts [Parent]. The optimization strategy is inspired by techniques for shape completion from depth maps, such as those by Park et al. [S1].

## Sources
[S1] PatchNets: Patch-Based Generalizable Deep Implicit 3D Shape Representations, https://doi.org/10.1007/978-3-030-58517-4_18, fetched 2026-10-27
