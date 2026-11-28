# NeRF Restarting Approach
Also called: Joint Pose-NeRF Refinement with Restart, Pose-Constrained NeRF Restart
## Overview
The NeRF Restarting Approach is an optimization protocol for jointly refining camera poses and neural radiance fields (NeRFs) from sparse, noisy initial poses. It solves the problem of artifacts persisting in the scene representation due to the coupling of pose and geometry optimization. Unlike [Staged Training](12984-staged-training.md), which freezes poses after initial convergence but retains the learned network weights, this method re-initializes the network from scratch after pose convergence, thereby removing artifacts learned during the pose optimization phase [S1].
## How it works
The method consists of three sequential phases.

**Inputs:**
- A set of $N$ training images with noisy initial camera poses $P_0 = \{T_1, \dots, T_N\}$.
- A NeRF architecture comprising a coarse MLP and a fine MLP.

**Outputs:**
- Optimized camera poses $P^*$.
- A high-fidelity NeRF model.

**Data Structures:**
- **Camera Pose:** A $4 \times 4$ transformation matrix $T \in \mathbb{R}^{4 \times 4}$ representing the world-to-camera transformation.
- **NeRF:** Two Multi-Layer Perceptrons (MLPs), $M_{coarse}$ and $M_{fine}$, mapping 3D coordinates and viewing directions to volume density and color.

**Steps:**

1. **Joint Optimization Phase:**
   Initialize $M_{coarse}$ and $M_{fine}$ randomly. Jointly optimize the parameters of both MLPs and the camera poses $P$ by minimizing the photometric rendering loss $\mathcal{L}_{photo}$ between rendered and ground-truth images. Continue this phase until the camera pose estimates converge (i.e., pose updates fall below a threshold or validation loss plateaus) [S1].

2. **Freezing and Re-initialization:**
   Let $P^*$ be the converged pose estimates from Phase 1. Freeze $P^*$ such that they are no longer trainable. Re-initialize the weights of both $M_{coarse}$ and $M_{fine}$ to random values, discarding the weights learned in Phase 1 [S1].

3. **Fine-Tuning Phase:**
   Train the re-initialized $M_{coarse}$ and $M_{fine}$ from scratch using the fixed poses $P^*$. Optimize the network parameters to minimize the rendering loss, effectively learning the scene geometry and appearance without the interference of pose uncertainty [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_restarting_approach(images, noisy_poses, num_iterations_phase1, num_iterations_phase2):
    # Phase 1: Joint Optimization
    mlp_coarse = initialize_random_mlp()
    mlp_fine = initialize_random_mlp()
    poses = noisy_poses.clone()
    poses.requires_grad = True
    
    for step in range(num_iterations_phase1):
        # Render images using current poses and MLPs
        rendered_images = render_neRF(images, poses, mlp_coarse, mlp_fine)
        # Compute photometric loss
        loss = compute_photometric_loss(rendered_images, images)
        # Backpropagate to update poses and MLPs
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # Check for pose convergence (implementation detail: [G])
        if check_pose_convergence(poses):
            break
            
    # Phase 2: Freezing and Re-initialization
    optimized_poses = poses.detach()
    optimized_poses.requires_grad = False
    
    # Re-initialize MLPs from scratch
    mlp_coarse = initialize_random_mlp()
    mlp_fine = initialize_random_mlp()
    
    # Phase 3: Fine-Tuning with Fixed Poses
    for step in range(num_iterations_phase2):
        # Render images using fixed optimized poses
        rendered_images = render_neRF(images, optimized_poses, mlp_coarse, mlp_fine)
        # Compute photometric loss
        loss = compute_photometric_loss(rendered_images, images)
        # Backpropagate to update only MLPs
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return optimized_poses, mlp_coarse, mlp_fine
```

**Edge Cases and Pitfalls:**
- **Convergence Criteria:** Determining when poses have "converged" in Phase 1 is critical. Premature stopping may leave poses inaccurate, while excessive training may overfit the network to noisy poses before restart [S1].
- **Re-initialization Cost:** The method requires two full training cycles, doubling the computational cost compared to single-pass methods [S1].

## Subtopics

### [Staged Training](12984-staged-training.md)
Staged Training is a two-phase optimization protocol for joint camera pose and neural radiance field (NeRF) reconstruction from sparse, noisy initial poses.

## Operating characteristics
- **Performance:** On the DTU dataset with 3 views and 15% initial additive Gaussian noise, the restarting approach achieves a PSNR of 17.80 (19.07 masked), SSIM of 0.72 (0.83 masked), LPIPS of 0.25 (0.12 masked), and Depth Error (DE) of 1.84 [S1].
- **Comparison:** It outperforms [Staged Training](12984-staged-training.md) by removing artifacts learned during the pose optimization phase that persist in the staged approach [S1].
- **Complexity:** The method doubles the training time compared to a single optimization pass, as it requires a full re-training of the network after pose convergence [S1].

## Background
The NeRF Restarting Approach was introduced in the context of sparse and noisy pose estimation for NeRFs. It was identified as the best alternative in prior work [51] and demonstrated to yield superior results compared to staged training protocols in ablation studies [S1]. The method addresses the limitation where joint optimization leads to a coupled solution where network artifacts mimic pose errors or vice versa.

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
