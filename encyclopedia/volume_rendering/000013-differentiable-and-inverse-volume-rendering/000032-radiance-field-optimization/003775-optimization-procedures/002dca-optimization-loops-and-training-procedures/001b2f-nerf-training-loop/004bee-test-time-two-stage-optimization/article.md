# Test-Time Two-Stage Optimization
Also called: Test-time pose and appearance optimization

## Overview
Test-time two-stage optimization is a post-training refinement procedure for unconstrained Neural Radiance Fields (NeRF) that corrects inaccurate camera poses and adapts appearance embeddings to handle varying lighting and transient objects in test images [S1, unofficial]. It distinguishes itself from prior methods by decoupling pose and appearance updates into two sequential stages: first optimizing poses while freezing network weights, then optimizing appearance embeddings while freezing poses, thereby avoiding the local minima and information leakage issues inherent in joint optimization [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A pre-trained NeRF model (coarse and fine MLPs), ground-truth poses for training images, estimated poses for test images, and a set of unconstrained test images.
*   **Outputs:** Refined camera poses for test images and optimized appearance embeddings, resulting in high-fidelity renderings of the test views.

**Data Structures**
*   **Pose Parameters:** A tensor of shape $(N_{test}, 4, 4)$ representing the $4 \times 4$ transformation matrices for each test image [S1, unofficial].
*   **Appearance Embeddings:** A learnable tensor of shape $(N_{test}, D)$, where $D$ is the embedding dimension (typically 256 or 384 depending on the specific architecture variant, e.g., UP-NeRF uses 384 for the feature field input) [S1, unofficial].
*   **Ray Sampling:** 1024 rays per iteration, with 128 points sampled for the coarse model and 128 points for the fine model per ray [S1, unofficial].

**Numbered Steps**

1.  **Stage 1: Pose Optimization**
    1.  **Freeze Network:** Freeze all MLP weights (coarse and fine) and appearance embeddings.
    2.  **Initialize Poses:** Compute an alignment matrix using Procrustes analysis between the trained poses of the training images and their ground-truth poses. Apply this alignment matrix to the ground-truth poses of the test images to initialize the test pose parameters [S1, unofficial].
    3.  **Initialize Embeddings:** Initialize appearance embeddings (e.g., to zero or random) as they are required for the forward pass but will be frozen during this stage.
    4.  **Optimize:** Optimize the pose parameters for 20,000 iterations using the test images. The learning rate is $1 \times 10^{-4}$ for poses and $5 \times 10^{-3}$ for appearance embeddings (which remain frozen in the gradient update but are part of the parameter set definition) [S1, unofficial].
    5.  **Loss:** Minimize the photometric reconstruction loss (e.g., MSE) between rendered and ground-truth test images.

2.  **Stage 2: Appearance Optimization**
    1.  **Freeze Poses:** Freeze the refined pose parameters obtained from Stage 1.
    2.  **Re-initialize Embeddings:** Re-initialize the appearance embeddings to prevent carry-over bias [S1, unofficial].
    3.  **Split Data:** Divide the test images into two halves: a training set (left half) and an evaluation set (right half) to prevent information leakage [S1, unofficial].
    4.  **Optimize:** Optimize the appearance embeddings for 20 epochs using only the training set (left half). The learning rate is $1 \times 10^{-1}$ with no decay [S1, unofficial].
    5.  **Evaluate:** Compute metrics (PSNR, SSIM, LPIPS) only on the evaluation set (right half) to report the best performance during the optimization process [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_two_stage_optimization(model, train_poses_gt, test_poses_gt, test_images):
    """
    model: Pre-trained NeRF with coarse and fine MLPs.
    train_poses_gt: Ground truth poses for training images (N_train, 4, 4).
    test_poses_gt: Ground truth poses for test images (N_test, 4, 4).
    test_images: Unconstrained test images (N_test, H, W, 3).
    """
    
    # --- Stage 1: Pose Optimization ---
    
    # 1. Freeze network weights
    for param in model.parameters():
        param.requires_grad = False
    
    # Initialize pose parameters for test images
    # Note: In implementation, these are usually stored in a separate parameter group
    test_poses = torch.nn.Parameter(test_poses_gt.clone().detach())
    
    # Initialize appearance embeddings (dummy values for forward pass)
    # Assuming embedding dimension D is known from model architecture
    D = 384 
    appearance_embeds = torch.nn.Parameter(torch.zeros(len(test_images), D))
    
    # 2. Procrustes Alignment for Initialization
    # Compute alignment matrix A such that A * train_poses_gt ≈ model_trained_poses
    # This step assumes 'model_trained_poses' are accessible from the trained model's state
    # or are the poses used during training.
    # Alignment Matrix Calculation (Procrustes Analysis)
    # Let P_train be trained poses, G_train be GT train poses.
    # Find rotation/translation R, t that minimizes || R*P_train + t - G_train ||
    # Apply this transformation to GT test poses
    alignment_matrix = compute_procrustes_alignment(model.trained_poses, train_poses_gt)
    test_poses.data = apply_alignment(test_poses_gt, alignment_matrix)
    
    # 3. Optimization Loop for Poses
    optimizer_pose = torch.optim.Adam([test_poses, appearance_embeds], lr=1e-4)
    # Note: appearance_embeds lr is set but gradients are zeroed or it is frozen in step logic
    # The source specifies lr 5e-3 for appearance embeddings in pose opt, but they are frozen.
    # We follow the source's parameter list strictly.
    
    for iteration in range(20000):
        optimizer_pose.zero_grad()
        
        # Sample 1024 rays from test images
        rays = sample_rays(test_images, num_rays=1024)
        
        # Render using current test_poses and frozen model
        # appearance_embeds are passed but gradients are not computed for them if strictly frozen
        # However, source says "freeze all parameters except pose... and appearance embedding"
        # This implies appearance embeddings ARE trainable in Stage 1? 
        # Re-reading S1: "freeze all the parameters except the pose parameter and appearance embedding"
        # Then: "optimize them [poses and appearance embeddings] with test images"
        # BUT later: "appearance embeddings are re-initialized with the pose parameters frozen"
        # This implies Stage 1 optimizes BOTH poses and appearance embeddings?
        # Let's re-read carefully: "To optimize poses, we freeze all the parameters except the pose parameter and appearance embedding... Then, appearance embeddings are re-initialized with the pose parameters frozen, and optimized again"
        # This suggests Stage 1 optimizes Poses AND Appearance Embeds. Stage 2 optimizes Appearance Embeds ONLY.
        
        # Correction based on strict reading of S1:
        # Stage 1: Optimize Poses AND Appearance Embeds.
        # Stage 2: Optimize Appearance Embeds (Poses Frozen).
        
        # Re-defining Stage 1 Optimizer
        optimizer_stage1 = torch.optim.Adam([test_poses, appearance_embeds], lr=1e-4) 
        # Source says: "learning rates are 5e-3 for appearance embeddings and 1e-4 for pose parameters"
        # We need separate param groups for different LRs
        optimizer_stage1 = torch.optim.Adam([
            {'params': test_poses, 'lr': 1e-4},
            {'params': appearance_embeds, 'lr': 5e-3}
        ])
        
        for iteration in range(20000):
            optimizer_stage1.zero_grad()
            rays = sample_rays(test_images, num_rays=1024)
            rendered_colors = model.forward(rays, test_poses, appearance_embeds)
            loss = compute_loss(rendered_colors, rays.gt_colors)
            loss.backward()
            optimizer_stage1.step()
            
    # --- Stage 2: Appearance Optimization ---
    
    # 1. Freeze Poses
    test_poses.requires_grad = False
    
    # 2. Re-initialize Appearance Embeddings
    appearance_embeds = torch.nn.Parameter(torch.zeros(len(test_images), D))
    
    # 3. Split Images
    mid = len(test_images) // 2
    train_images = test_images[:mid]
    eval_images = test_images[mid:]
    
    # 4. Optimization Loop for Appearance
    optimizer_stage2 = torch.optim.Adam([appearance_embeds], lr=1e-1)
    
    best_psnr = -float('inf')
    best_psnr_val = -float('inf')
    
    for epoch in range(20):
        # Train on left half
        optimizer_stage2.zero_grad()
        rays_train = sample_rays(train_images, num_rays=1024)
        rendered_train = model.forward(rays_train, test_poses, appearance_embeds)
        loss_train = compute_loss(rendered_train, rays_train.gt_colors)
        loss_train.backward()
        optimizer_stage2.step()
        
        # Evaluate on right half (no grad)
        with torch.no_grad():
            rays_eval = sample_rays(eval_images, num_rays=1024)
            rendered_eval = model.forward(rays_eval, test_poses, appearance_embeds)
            psnr_val = compute_psnr(rendered_eval, rays_eval.gt_colors)
            
            if psnr_val > best_psnr_val:
                best_psnr_val = psnr_val
                # Save best appearance embeddings
                best_embeds = appearance_embeds.data.clone()
                
    # Return best appearance embeddings and final poses
    return test_poses, best_embeds

def compute_procrustes_alignment(P, G):
    """
    Computes the alignment matrix from trained poses P to GT poses G.
    Simplified placeholder for Procrustes analysis.
    """
    # Center points
    P_centered = P - P.mean(dim=0, keepdim=True)
    G_centered = G - G.mean(dim=0, keepdim=True)
    
    # Covariance matrix
    H = P_centered.T @ G_centered
    
    # SVD
    U, S, Vt = torch.linalg.svd(H)
    
    # Rotation
    R = Vt.T @ U.T
    
    # Check for reflection
    if torch.linalg.det(R) < 0:
        Vt[-1, :] *= -1
        R = Vt.T @ U.T
        
    # Translation
    t = G.mean(dim=0) - R @ P.mean(dim=0)
    
    # Construct 4x4 matrix
    A = torch.eye(4)
    A[:3, :3] = R
    A[:3, 3] = t
    return A

def apply_alignment(GT_poses, A):
    """Applies alignment matrix A to GT poses."""
    # Homogeneous coordinates
    GT_homo = torch.cat([GT_poses[:, :3, :], torch.ones_like(GT_poses[:, 3:, :])], dim=1)
    Aligned = A @ GT_homo
    return Aligned[:, :3, :] # Return 3x4 or keep 4x4 depending on model expectation
```

**Parameters**
*   **Iterations:** 20,000 for Stage 1 (Pose Optimization) [S1, unofficial].
*   **Epochs:** 20 for Stage 2 (Appearance Optimization) [S1, unofficial].
*   **Learning Rates:**
    *   Stage 1: $1 \times 10^{-4}$ for poses, $5 \times 10^{-3}$ for appearance embeddings [S1, unofficial].
    *   Stage 2: $1 \times 10^{-1}$ for appearance embeddings, no decay [S1, unofficial].
*   **Sampling:** 1024 rays per iteration, 128 points for coarse model, 128 points for fine model [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Information Leakage:** Evaluating on the same images used for training appearance embeddings leads to overfitting. The method strictly splits test images into left (train) and right (eval) halves [S1, unofficial].
*   **Local Minima:** Without a candidate head or proper feature field initialization, pose estimation may fall into local minima. The method relies on the pre-trained feature field to guide pose optimization [S1, unofficial].
*   **Precision:** Procrustes analysis assumes a rigid transformation. If the scene has non-rigid changes, this initialization may be suboptimal [G].

## Operating characteristics
*   **Time Complexity:** The method adds significant inference-time cost. Stage 1 requires 20,000 iterations of ray marching and backpropagation through the pose parameters. Stage 2 requires 20 epochs of optimization over appearance embeddings. This is substantially slower than standard inference.
*   **Memory Complexity:** Requires storing gradients for pose parameters and appearance embeddings. The ray sampling is reduced to 1024 rays (from 2048 in training) to manage memory and speed [S1, unofficial].
*   **Strengths:** Effectively handles unconstrained image collections with varying lighting and transient objects by decoupling pose and appearance errors. It outperforms methods that only optimize poses (BARF) or only appearance (NeRF-W) in complex scenarios [S1, unofficial].
*   **Limitations:** Computationally expensive. Requires a pre-trained model with good initial pose estimates (via Procrustes). Sensitive to the split of test images for evaluation.
*   **When to Use:** When rendering unconstrained test images where camera poses are inaccurate or unknown, and lighting/appearance varies significantly from training data.
*   **When Not to Use:** For constrained datasets with accurate ground-truth poses and consistent lighting, where standard inference suffices.

## Background
Test-time two-stage optimization was introduced in the context of unconstrained Neural Radiance Fields, specifically in the UP-NeRF framework [S1, unofficial]. It addresses the limitations of earlier methods like BARF (which only optimizes poses) and NeRF-W (which only optimizes appearance with given poses) by combining both optimizations in a staged manner to handle the complexities of real-world, unconstrained image collections [S1, unofficial]. The method was developed to improve reconstruction quality in scenarios with transient objects and varying illumination.

## Sources
[S1] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
