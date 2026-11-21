# Test-Time Two-Stage Optimization

Also called: Test-time pose and appearance optimization

## Overview
Test-Time Two-Stage Optimization is a post-training evaluation procedure that first optimizes camera poses and then optimizes appearance embeddings to handle unconstrained image collections with varying lighting and transient objects [S1, unofficial]. It distinguishes itself from sibling methods by decoupling pose refinement and appearance adaptation into two sequential stages, addressing the limitations of methods that optimize only poses (e.g., BARF) or only appearance (e.g., NeRF-W) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF model (weights $\Theta$), a set of test images $\{I_i\}$, and initial camera pose estimates $P_i$ for the test images [S1, unofficial].
*   **Outputs:** Refined camera poses $P'_i$ and optimized appearance embeddings $A_i$ for the test images [S1, unofficial].

**Data Structures:**
*   **Pose Parameters:** For each test image $i$, a 6-dimensional vector representing rotation and translation, or a $4 \times 4$ transformation matrix [G].
*   **Appearance Embeddings:** A learnable vector $a_i$ per test image, concatenated with the positional encoding of the 3D point before the density/color MLPs [G].
*   **NeRF Weights:** Fixed during optimization [S1, unofficial].

**Numbered Steps:**

1.  **Stage 1: Pose Optimization**
    *   **Initialization:** Initialize appearance embeddings $A_i$ randomly [S1, unofficial]. Initialize test pose parameters $P_i$ using Procrustes analysis: compute the alignment matrix $M$ between the trained poses of training images and their ground-truth poses, then apply $M$ to the ground-truth test poses [S1, unofficial].
    *   **Freezing:** Freeze all NeRF network weights $\Theta$.
    *   **Optimization:** Optimize $P_i$ and $A_i$ jointly using the test images.
    *   **Sampling:** Sample 1024 rays per iteration. For each ray, sample 128 points for the coarse model and 128 points for the fine model [S1, unofficial].
    *   **Duration:** 20,000 iterations [S1, unofficial].
    *   **Learning Rates:** $1 \times 10^{-4}$ for pose parameters $P_i$; $5 \times 10^{-3}$ for appearance embeddings $A_i$ [S1, unofficial]. No learning rate decay is applied [S1, unofficial].

2.  **Stage 2: Appearance Optimization**
    *   **Freezing:** Freeze the refined pose parameters $P'_i$ from Stage 1 [S1, unofficial].
    *   **Re-initialization:** Re-initialize the appearance embeddings $A_i$ [S1, unofficial].
    *   **Optimization:** Optimize only $A_i$ using the test images.
    *   **Data Splitting:** To prevent information leakage, optimize on the left half of the test images and evaluate only on the right half [S1, unofficial].
    *   **Duration:** 20 epochs [S1, unofficial].
    *   **Learning Rates:** $1 \times 10^{-1}$ for appearance embeddings $A_i$ [S1, unofficial]. No learning rate decay is applied [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_two_stage_optimization(nerf_model, test_images, initial_poses, train_trained_poses, train_gt_poses):
    """
    nerf_model: Pre-trained NeRF with fixed weights.
    test_images: List of test images [I_1, ..., I_N].
    initial_poses: Initial pose estimates for test images.
    train_trained_poses: Trained poses of training images.
    train_gt_poses: Ground-truth poses of training images.
    """
    N = len(test_images)
    
    # Stage 1: Pose Optimization
    # Initialize appearance embeddings randomly
    appearance_embeddings = [torch.randn(1, 256) for _ in test_images]
    
    # Initialize poses using Procrustes analysis
    # Compute alignment matrix M between train_trained_poses and train_gt_poses
    M = compute_procrustes_alignment(train_trained_poses, train_gt_poses)
    poses = apply_alignment(initial_poses, M)
    
    # Optimize poses and appearance embeddings for 20,000 iterations
    for iteration in range(20000):
        # Sample 1024 rays
        rays = sample_rays(test_images, poses, num_rays=1024)
        
        # Render using coarse and fine models (128 points each)
        rendered_colors = nerf_model.render(rays, poses, appearance_embeddings)
        
        # Compute loss against ground truth
        loss = compute_loss(rendered_colors, rays.gt_colors)
        
        # Backpropagate to update poses and appearance embeddings
        # LR_pose = 1e-4, LR_appearance = 5e-3
        update_parameters(poses, appearance_embeddings, loss, 
                          lr_pose=1e-4, lr_appearance=5e-3)
    
    refined_poses = poses
    
    # Stage 2: Appearance Optimization
    # Freeze refined_poses
    # Re-initialize appearance embeddings
    appearance_embeddings = [torch.randn(1, 256) for _ in test_images]
    
    # Split test images into left and right halves
    left_images = test_images[:N//2]
    right_images = test_images[N//2:]
    
    # Optimize appearance embeddings for 20 epochs
    for epoch in range(20):
        # Use only left half for training
        rays = sample_rays(left_images, refined_poses, num_rays=1024)
        rendered_colors = nerf_model.render(rays, refined_poses, appearance_embeddings)
        loss = compute_loss(rendered_colors, rays.gt_colors)
        
        # LR_appearance = 1e-1
        update_parameters(appearance_embeddings, loss, lr_appearance=1e-1)
        
        # Evaluate on right half to prevent information leakage
        eval_rays = sample_rays(right_images, refined_poses, num_rays=1024)
        eval_rendered = nerf_model.render(eval_rays, refined_poses, appearance_embeddings)
        eval_loss = compute_loss(eval_rendered, eval_rays.gt_colors)
        
    return refined_poses, appearance_embeddings

def compute_procrustes_alignment(A, B):
    # Standard Procrustes alignment: returns rotation/translation matrix M
    # such that B ≈ M @ A. Implemented via SVD of covariance matrix.
    return standard_procrustes(A, B) # [G]

def apply_alignment(poses, M):
    return M @ poses # [G]
```

**Edge Cases and Pitfalls:**
*   **Information Leakage:** Optimizing on the same images used for evaluation can lead to overfitting. The method mitigates this by splitting test images into training (left half) and evaluation (right half) sets during Stage 2 [S1, unofficial].
*   **Local Minima:** Without proper initialization, pose optimization may fall into local minima. Procrustes analysis provides a robust initialization based on training data alignment [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The optimization is performed at test time, adding latency. Stage 1 requires 20,000 iterations, and Stage 2 requires 20 epochs. The number of sampled rays is reduced to 1024 per iteration (from 2048 in training) to speed up computation [S1, unofficial].
*   **Memory Complexity:** Requires storing pose parameters and appearance embeddings for all test images. The NeRF weights are frozen, so no additional memory for model weights is needed [G].
*   **Strengths:** Handles unconstrained image collections with varying lighting and transient objects effectively. Decoupling pose and appearance optimization improves convergence stability [S1, unofficial].
*   **Limitations:** Computationally expensive due to the two-stage optimization process. Requires a pre-trained NeRF model. Performance depends on the quality of the initial pose estimates [G].
*   **When to Use:** When dealing with unconstrained image collections where camera poses are not known or are inaccurate, and lighting conditions vary significantly [S1, unofficial].
*   **When Not to Use:** When computational resources are limited, or when camera poses are already accurate and lighting is consistent [G].

## Background
Test-Time Two-Stage Optimization was introduced in the context of UP-NeRF (Unconstrained Pose-Prior-Free Neural Radiance Fields) [S1, unofficial]. It addresses the limitations of previous methods like BARF, which only optimizes poses, and NeRF-W, which only optimizes appearance given fixed poses [S1, unofficial].

## Sources
[S1] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
