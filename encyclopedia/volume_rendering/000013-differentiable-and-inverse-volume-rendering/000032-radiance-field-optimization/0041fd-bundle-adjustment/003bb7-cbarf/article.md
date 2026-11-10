# CBARF

Also called: CBARF, Cascaded BA

## Overview
CBARF is a 3D reconstruction method that optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones [S1]. It builds on [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) by introducing a coarse-to-fine optimization strategy with distinct modulation factors for each stage, allowing it to handle larger initial pose errors (e.g., 30.4° rotation deviation) where standard BARF may overfit [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of images with imperfect camera poses $\hat{P}$, a NeRF backbone (BARF), and scene geometry parameters.
- **Outputs:** Refined camera poses $Q$, a trained NeRF model, and optionally synthetic images via a Voxel Grid Module (DVGO) [S1].

**Data Structures:**
- **Camera Pose:** Represented as $P \in \mathbb{R}^{3 \times 4}$, defined as $P = [R_p \ t_p]$ where $R_p \in SO(3)$ and $t_p \in \mathbb{R}^3$ [S1]. The last row of the standard $4 \times 4$ projection matrix is dropped [S1].
- **Noise/Refinement Parameters:** Represented in the Lie algebra $\mathfrak{se}(3)$ as $n \in \mathbb{R}^6$ [S1].
- **Transform Matrix:** Derived from $\mathfrak{se}(3)$ parameters, represented as $N = [R_n \ t_n]$ [S1].

**Numbered Steps:**

1.  **Simulate Imperfect Poses (Initialization):**
    Generate noise $n \in \mathfrak{se}(3)$ from a 6-dimensional random normal distribution:
    $$ n = 0.35 \mathcal{N}(0, I) \in \mathbb{R}^6 $$
    [S1]
    Transform $n$ into a camera transform matrix $N = [R_n \ t_n]$ [S1]. Compose this with the reference pose $P = [R_p \ t_p]$ to obtain the imperfect pose $\hat{P}$:
    $$ \hat{P} = [R_p R_n \ t_p + t_n] $$
    [S1]
    *Note:* The coefficient 0.35 corresponds to an average deviation of 30.4° in rotation and 0.56 in translation [S1].

2.  **Define Refined Pose Composition:**
    Optimize camera poses by training camera refine parameters $p \in \mathfrak{se}(3)$. Convert $p$ into a transform matrix $P_r = [R_r \ t_r]$ [S1]. Compose $P_r$ with $\hat{P}$ to obtain the refined pose $Q$:
    $$ Q = [R_p R_n R_r \ t_p + t_n + t_r] $$
    [S1]

3.  **Cascaded Optimization Structure:**
    Use a coarse-to-fine structure with compact BA modules, using the BARF network as the backbone [S1]. The process consists of three stages: Coarse, Recursive, and Fine [S1].

4.  **Learning Rate Schedule:**
    Calculate the learning rate using exponential interpolation with a modulation factor $\alpha$ to bias learning towards the initial high value [S1]. The modulation factors for the three stages are:
    - Coarse stage: $\alpha = 10.0$
    - Recursive stage: $\alpha = 3.0$
    - Fine stage: $\alpha = 1.0$
    [S1]
    A factor of 1.0 corresponds to the original exponential learning rate used in BARF [S1].

5.  **Handling Incomplete Poses (Optional):**
    For incomplete camera poses, randomly drop 10% of poses to form a test set $F$, assigning an initial pose $P_{ini}$ to $F$. The remaining images form the training set $T$. Jointly optimize poses for both $T$ and $F$ during training [S1].

6.  **Evaluation Alignment:**
    Use Procrustes analysis to align optimized poses with ground-truth before evaluation to remove global offset [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def generate_noise(coefficient=0.35):
    """
    Generate noise n in se(3) from a 6D normal distribution.
    Returns n as a 6D vector.
    """
    n = coefficient * np.random.normal(0, 1, 6)
    return n

def se3_to_transform(n):
    """
    Convert se(3) vector n (6D) to a 3x4 transform matrix [R t].
    Note: The exact exponential map implementation depends on the library.
    This is a placeholder for the mathematical operation described in [S1].
    """
    # Implementation detail: Convert 6D vector to rotation matrix R and translation t
    # using the exponential map from so(3) and translation components.
    # Since the source does not provide the specific exponential map code,
    # we assume a standard library function or mathematical implementation exists.
    # For the purpose of this article, we denote the result as [R_n, t_n].
    R_n, t_n = exp_map_se3(n) 
    return np.hstack([R_n, t_n.reshape(3, 1)])

def compose_poses(P_ref, N_noise):
    """
    Compose reference pose P_ref [3x4] with noise transform N_noise [3x4].
    P_ref = [R_p, t_p]
    N_noise = [R_n, t_n]
    Result = [R_p @ R_n, t_p + t_n]
    """
    R_p = P_ref[:, :3]
    t_p = P_ref[:, 3:]
    R_n = N_noise[:, :3]
    t_n = N_noise[:, 3:]
    
    R_new = R_p @ R_n
    t_new = t_p + t_n
    return np.hstack([R_new, t_new])

def optimize_cbarf(images, initial_imperfect_poses, stages_config):
    """
    Main optimization loop for CBARF.
    stages_config: List of dicts with 'iterations', 'modulation_factor', 'resolution'
    """
    current_poses = initial_imperfect_poses
    nerf_model = BARF_Backbone()
    
    for stage in stages_config:
        iterations = stage['iterations']
        modulation = stage['modulation_factor']
        resolution = stage['resolution']
        
        # Setup learning rate with modulation
        # LR_schedule = initial_lr * (modulation_factor) * exp_decay
        # The source states modulation biases learning towards initial high value.
        
        for step in range(iterations):
            # Sample a batch of rays/images at the specified resolution
            rays, targets = sample_rays(images, current_poses, resolution)
            
            # Forward pass through NeRF
            predicted_colors = nerf_model(rays, current_poses)
            
            # Compute loss (e.g., MSE)
            loss = compute_loss(predicted_colors, targets)
            
            # Backward pass
            # Gradients are computed for both NeRF parameters and camera refine parameters p
            loss.backward()
            
            # Update parameters
            optimizer.step()
            optimizer.zero_grad()
            
            # Decay learning rate based on exponential interpolation with modulation
            update_learning_rate(optimizer, modulation, step, total_steps=iterations)
            
        # After stage, current_poses are updated by the refined parameters p
        # Q = [R_p R_n R_r t_p + t_n + t_r]
        # The optimizer updates p, which is converted to P_r and composed.
        current_poses = update_poses_with_refinement(current_poses, nerf_model.camera_params)
        
    return current_poses, nerf_model

def procrustes_align(optimized_poses, ground_truth_poses):
    """
    Align optimized poses with ground-truth using Procrustes analysis.
    """
    # Standard Procrustes analysis to find global rotation and translation
    # that minimizes the difference between the two sets of poses.
    # This removes global offset for evaluation.
    aligned_poses = procrustes(optimized_poses, ground_truth_poses)
    return aligned_poses
```

**Parameters:**
- **Noise Coefficient:** 0.35 (for simulation) [S1].
- **Iterations:** 20k for Coarse and Recursive stages; 200k for Fine stage [S1].
- **Image Resolution:** 400×400 during pose optimization (half of 800×800 training resolution) [S1].
- **Modulation Factors:** 10.0 (Coarse), 3.0 (Recursive), 1.0 (Fine) [S1].

**Edge Cases and Pitfalls:**
- **Overfitting:** Standard BARF may encounter overfitting before completing optimization with large initial errors; CBARF mitigates this via the cascaded structure [S1].
- **Global Offset:** Optimized poses may have a misalignment with the ground-truth coordinate system. Procrustes analysis is required for evaluation but does not correct the pose during optimization [S1].
- **Incomplete Poses:** Randomly dropping 10% of poses requires joint optimization of training and test sets, using rendering quality as an indirect metric for pose error in the absence of ground-truth for the test set [S1].

## Operating characteristics
- **Time Complexity:** The method uses a coarse-to-fine structure. The total iterations are 240k per scene (20k + 20k + 200k) [S1].
- **Memory Complexity:** Uses the BARF backbone, which stores the NeRF MLP and camera parameters. Image resolution is reduced to 400×400 during optimization to improve efficiency [S1].
- **Performance:** Successfully optimizes poses with 30.4° rotation deviation, whereas BARF fails [S1]. Rendering quality is comparable to ground-truth images [S1].
- **Strengths:** Handles large initial pose errors; progressive refinement prevents overfitting.
- **Limitations:** Requires careful tuning of modulation factors; Procrustes alignment is needed for evaluation, implying the absolute pose is not directly optimized to a global frame without post-processing [S1].

## Background
CBARF was proposed in 2024 by authors publishing in the IEEE Transactions on Multimedia [S1]. It addresses the limitations of [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) in handling imperfect camera poses by introducing a cascaded BA structure [S1]. The method is used for 3D reconstruction from imperfect camera poses, including scenarios with incomplete pose data [S1].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
