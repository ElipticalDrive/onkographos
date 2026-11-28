# SplattingAvatar

Also called: SplattingAvatar: Realistic Real-Time Human Avatars with Mesh-Embedded Gaussian Splatting

## Overview
SplattingAvatar is a real-time human avatar rendering method that embeds 3D Gaussians directly onto a driving mesh (FLAME or NHA) in mesh tangent space [S2, unofficial]. It solves the problem of noisy deformation and ambiguous motion control found in nearest-triangle sampling methods (like INSTA) by using trainable embeddings for motion control, enabling high-fidelity rendering of complex geometries like eyes and hair [S2, unofficial]. It differs from geometry-focused methods like FLARE by prioritizing photometric quality and rendering efficiency over accurate material reconstruction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-angle video of a subject; pre-fitted driving mesh (FLAME or NHA) [S2, unofficial].
*   **Outputs:** Animatable 3D Gaussian avatar; Unity-compatible render with >300 FPS [S2, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Position, scale, rotation, color, opacity, and trainable embeddings [S2, unofficial].
*   **Mesh Association:** Each Gaussian is initialized per mesh triangle in tangent space [S1, unofficial][S2, unofficial].
*   **Rendering Primitive:** One quad primitive per Gaussian [S2, unofficial].

**Training Optimization:**
1.  **Initialization:** Initialize one Gaussian per mesh triangle in tangent space [S1, unofficial][S2, unofficial].
2.  **Iteration Loop:** Run for 30,000 iterations [S2, unofficial].
3.  **Densification/Pruning:** Starting from iteration 600, perform densify and prune every 100 iterations [S2, unofficial].
4.  **Opacity Reset:** Every 3000 iterations, reset the opacity of all Gaussians to zero to remove redundant Gaussians [S2, unofficial].
5.  **Walking on Triangle:** After densifying and pruning, perform the "walking on triangle" step to ensure Gaussians remain aligned with the mesh surface [S2, unofficial].
6.  **Termination:** Densify, prune, and opacity-reset processes stop at iteration 15,000 [S2, unofficial].

**Parameters:**
*   $\lambda_l$ (L1 loss weight) = 0.01 [S2, unofficial].
*   $\lambda_s$ (SSIM loss weight) = 1.0 [S2, unofficial].
*   $T_s$ (Scale threshold) = 10.0 [S2, unofficial].
*   $T_r$ (Rotation threshold) = 0.008 [S2, unofficial].

**Pitfalls:**
*   **Nearest-Triangle Sampling:** Using nearest-triangle sampling for deformation (as in INSTA) causes unstable sampling in canonical space and strong noise in complex geometries like the eye region [S2, unofficial].
*   **Disabling Walking on Triangle:** Disabling this mechanism and clipping UV values causes Gaussians to stick and pile up on mesh triangle boundaries, degrading PSNR, SSIM, and LPIPS [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# SplattingAvatar Training Loop
def train_splatting_avatar(mesh, video_frames, iterations=30000):
    # Initialize Gaussians per mesh triangle in tangent space
    gaussians = initialize_gaussians_per_triangle(mesh)
    
    for iter in range(1, iterations + 1):
        # Get current pose from video
        pose = get_pose(video_frames, iter)
        
        # Render using Unity-compatible quad primitives
        # Embeddings are used for motion control in compute shaders
        rendered_image = render_gaussians(gaussians, pose)
        
        # Calculate loss
        # lambda_l = 0.01, lambda_s = 1.0
        loss = 0.01 * l1_loss(rendered_image, ground_truth) + 1.0 * ssim_loss(rendered_image, ground_truth)
        
        # Backpropagate and update embeddings and Gaussian attributes
        update_parameters(gaussians, loss)
        
        # Densify and Prune every 100 iterations starting from 600
        if iter >= 600 and iter % 100 == 0:
            densify_and_prune(gaussians)
            # Walking on Triangle step
            walking_on_triangle(gaussians, mesh)
            
        # Opacity Reset every 3000 iterations
        if iter % 3000 == 0:
            reset_opacity(gaussians)
            
        # Stop densify/prune/reset after 15000 iterations
        if iter > 15000:
            disable_densify_prune_reset()
            
    return gaussians
```

## Subtopics

### GauHuman
Redesigns densification primitives for human dynamics [S1, unofficial][S2, unofficial]. [Link](9198-gauhuman.md)

### GaussianBody
Focuses on body avatar reconstruction [S2, unofficial]. [Link](9209-gaussianbody.md)

### WildGaussians
Handles dynamic scenes with wild Gaussians [S2, unofficial]. [Link](6064-wildgaussians.md)

## Operating characteristics
*   **Performance:** Over 300 FPS on NVIDIA RTX 3090 GPU [S2, unofficial].
*   **Quality:** On head avatar dataset, Ours+FLAME achieves PSNR 28.19, SSIM 0.931, LPIPS 0.063; Ours+NHA achieves PSNR 28.86, SSIM 0.931, LPIPS 0.060 [S2, unofficial].
*   **Comparison:** Outperforms FLARE in photometric quality and rendering efficiency, though FLARE focuses more on accurate geometry and materials [S2, unofficial].
*   **Limitations:** Does not focus on accurate geometry and materials reconstruction compared to mesh-based approaches like FLARE [S2, unofficial].

## Background
SplattingAvatar was introduced as a method for realistic real-time human avatars using mesh-embedded Gaussian Splatting [S2, unofficial]. It builds on the SMPL-X rig and embeds Gaussians in mesh tangent space [S1, unofficial]. It addresses issues in previous methods like INSTA by avoiding nearest-triangle sampling noise [S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
