# CBARF

Also called: CBARF, Cascaded BA

## Overview
CBARF is a 3D reconstruction model designed to optimize imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones [S1]. It builds on [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) by introducing a coarse-to-fine optimization strategy with compact BA modules, addressing the overfitting and convergence issues of standard BA when initialized with highly inaccurate poses [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of images with initial imperfect camera poses $\hat{P}$, a NeRF backbone (typically BARF), and ground-truth poses $P_{gt}$ (for evaluation only).
- **Outputs:** Refined camera poses $Q$ and a reconstructed 3D scene representation.

**Data Structures:**
- **Camera Pose:** Represented as $P \in \mathbb{R}^{3 \times 4}$, defined as $P = [R_p \ t_p]$ where $R_p \in SO(3)$ and $t_p \in \mathbb{R}^3$ [S1]. The last row of the $4 \times 4$ transformation matrix is dropped [S1].
- **Noise/Refinement Parameters:** Represented in the Lie algebra $\mathfrak{se}(3)$ as $n, p \in \mathbb{R}^6$ [S1].

**Numbered Steps:**
1. **Pose Initialization:** Generate imperfect initial poses $\hat{P}$ by adding noise to reference poses $P$.
2. **Cascaded Optimization:** Perform optimization in three stages: Coarse, Recursive, and Fine. Each stage uses a compact BA module based on BARF.
3. **Learning Rate Scheduling:** Use exponential interpolation with a stage-specific modulation factor to bias learning towards the initial high value [S1].
4. **Evaluation Alignment:** Apply Procrustes analysis to align optimized poses with ground-truth before computing error metrics [S1].

**Formulas:**
- **Noise Generation:** Noise $n$ is drawn from a 6-dimensional normal distribution:
  $$ n = 0.35 \mathcal{N}(0, I) \in \mathbb{R}^6 $$
  This corresponds to an average deviation of $30.4^\circ$ in rotation and $0.56$ in translation [S1].
- **Imperfect Pose Construction:** Transform noise $n$ into a matrix $N = [R_n \ t_n]$ and compose with reference pose $P$:
  $$ \hat{P} = P N = [R_p R_n \ t_p + t_n] $$
  [S1]
- **Refined Pose Construction:** Optimize refinement parameters $p \in \mathfrak{se}(3)$, convert to $P_r = [R_r \ t_r]$, and compose with $\hat{P}$:
  $$ Q = \hat{P} P_r = [R_p R_n R_r \ t_p + t_n + t_r] $$
  [S1]
- **Learning Rate:** Calculated via exponential interpolation with modulation factor $\alpha$:
  $$ \text{lr}(t) = \text{lr}_{init} \cdot \alpha^{t/T} $$
  Where $\alpha$ is 10.0 (coarse), 3.0 (recursive), and 1.0 (fine) [S1].

**Parameters:**
- **Noise Coefficient:** 0.35 (compared to 0.15 in BARF) [S1].
- **Iterations:** 20k for coarse and recursive stages; 200k for fine stage [S1].
- **Image Resolution:** 400×400 (half of the 800×800 training resolution) during pose optimization [S1].
- **Modulation Factors:** 10.0 (coarse), 3.0 (recursive), 1.0 (fine) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cbarf_optimize(images, initial_poses_gt, num_iterations_coarse=20000, 
                   num_iterations_recursive=20000, num_iterations_fine=200000,
                   noise_coeff=0.35, modulation_factors=[10.0, 3.0, 1.0]):
    """
    Optimizes camera poses using Cascaded Bundle-Adjusting NeRF.
    """
    # Step 1: Initialize Imperfect Poses
    # Generate noise n in se(3)
    noise = noise_coeff * np.random.randn(6)
    # Convert noise to transformation matrix N
    N = se3_to_matrix(noise)
    # Compose with ground truth to get imperfect poses P_hat
    # Note: initial_poses_gt is in R^{3x4} format [R|t]
    P_hat = compose_poses(initial_poses_gt, N)
    
    current_poses = P_hat
    
    # Step 2: Coarse Stage
    # Use modulation factor 10.0
    current_poses = run_compact_ba_module(
        images, current_poses, 
        iterations=num_iterations_coarse, 
        modulation_factor=modulation_factors[0],
        resolution=(400, 400)
    )
    
    # Step 3: Recursive Stage
    # Use modulation factor 3.0
    current_poses = run_compact_ba_module(
        images, current_poses, 
        iterations=num_iterations_recursive, 
        modulation_factor=modulation_factors[1],
        resolution=(400, 400)
    )
    
    # Step 4: Fine Stage
    # Use modulation factor 1.0
    current_poses = run_compact_ba_module(
        images, current_poses, 
        iterations=num_iterations_fine, 
        modulation_factor=modulation_factors[2],
        resolution=(400, 400)
    )
    
    return current_poses

def run_compact_ba_module(images, poses, iterations, modulation_factor, resolution):
    """
    Runs a single compact BA module using BARF backbone.
    """
    # Initialize BARF network with current poses
    nerf = BARF_Backbone(poses)
    
    # Learning rate schedule with modulation
    # lr(t) = lr_init * modulation_factor^(t/total_iterations)
    # This biases learning towards the initial high value
    
    for t in range(iterations):
        # Sample rays based on current poses
        rays = generate_rays(images, poses, resolution)
        
        # Render colors using volume rendering
        rendered_colors = nerf.render(rays)
        
        # Compute photometric loss
        loss = photometric_loss(rendered_colors, images)
        
        # Backpropagate to update network weights and pose refinement parameters
        gradients = compute_gradients(loss, nerf)
        update_parameters(nerf, gradients)
        
        # Update poses based on refined parameters
        poses = update_poses_from_refinement(poses, nerf.refinement_params)
        
    return poses

def se3_to_matrix(vec_6):
    """
    Converts 6D vector in se(3) to 3x4 transformation matrix [R|t].
    """
    # Extract rotation vector and translation vector
    omega = vec_6[:3]
    v = vec_6[3:]
    
    # Convert rotation vector to rotation matrix using Rodrigues' formula
    R = rodrigues(omega)
    
    # Convert twist vector to translation
    # Note: Exact conversion depends on Lie algebra definition, 
    # typically involves integrating the twist or using exponential map
    t = exponential_map_se3(vec_6)[:3] 
    
    return np.hstack([R, t.reshape(3, 1)])

def compose_poses(P1, P2):
    """
    Composes two 3x4 pose matrices.
    P1 = [R1|t1], P2 = [R2|t2]
    Result = [R1*R2 | t1 + R1*t2]
    """
    R1 = P1[:, :3]
    t1 = P1[:, 3:]
    R2 = P2[:, :3]
    t2 = P2[:, 3:]
    
    R_new = R1 @ R2
    t_new = t1 + R1 @ t2
    
    return np.hstack([R_new, t_new])
```

**Edge Cases and Pitfalls:**
- **Overfitting:** Standard BA may overfit before completing optimization if noise is high; CBARF mitigates this via coarse-to-fine stages [S1].
- **Coordinate Misalignment:** Optimized poses may have a global offset relative to ground-truth; Procrustes analysis is required for evaluation [S1].
- **Incomplete Poses:** If 10% of poses are missing, they are assigned an initial pose $P_{ini}$ and jointly optimized with the training set [S1].

## Operating characteristics
- **Time Complexity:** The cascaded structure increases total iterations (20k + 20k + 200k = 240k per module) compared to single-stage BA, but the coarse-to-fine strategy improves convergence reliability [S1].
- **Memory Complexity:** Uses half resolution (400×400) during pose optimization to improve efficiency [S1].
- **Strengths:** Successfully optimizes poses with high initial error (30.4° rotation, 0.56 translation) where BARF fails [S1].
- **Limitations:** Requires careful tuning of modulation factors and iteration counts for each stage [S1].

## Background
CBARF was proposed in 2024 by authors publishing in IEEE Transactions on Multimedia [S1]. It extends the BARF method [S1] by addressing the significance of camera pose initialization for bundle-adjustment performance. The method introduces a cascaded BA structure and a neighbor-replacement strategy to rectify erroneous poses [S1].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
