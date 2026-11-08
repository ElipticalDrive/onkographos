# CBARF

Also called: CBARF, Cascaded BA

## Overview
CBARF is a 3D reconstruction model that optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones [S1]. It addresses the sensitivity of joint NeRF optimization to initialization by decomposing the refinement process into coarse, recursive, and fine stages, allowing the model to converge from large initial errors where standard methods like BARF might overfit or diverge [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images with initial imperfect camera poses $\hat{P}$, a NeRF backbone (typically BARF), and training hyperparameters.
*   **Outputs:** Refined camera poses $Q$, optimized NeRF weights, and synthesized images.

**Data Structures:**
*   **Camera Pose:** Represented as $P \in \mathbb{R}^{3 \times 4}$, defined as $P = [R_p \ t_p]$ where $R_p \in SO(3)$ and $t_p \in \mathbb{R}^3$ [S1]. The last row of the $4 \times 4$ transformation matrix is dropped [S1].
*   **Lie Algebra Noise:** Perturbations are represented as $n \in \mathbb{R}^6$ from the Lie algebra $se(3)$ [S1].

**Algorithm Steps:**

1.  **Initialization (Simulating Imperfect Poses):**
    To train and evaluate robustness, initial poses are perturbed. Generate noise $n$ from a 6-dimensional normal distribution:
    $$ n = \sigma N(0, I) \in \mathbb{R}^6 $$
    where $\sigma$ is the noise coefficient (0.35 in the source implementation) [S1]. Transform $n$ into a camera transform matrix $N = [R_n \ t_n]$ using the exponential map of $se(3)$ [S1]. Compose with the reference pose $P$ to get imperfect poses $\hat{P}$:
    $$ \hat{P} = [R_p \ R_n \ t_p + t_n] $$
    [S1].

2.  **Cascaded Optimization Structure:**
    The optimization proceeds through three stages: Coarse, Recursive, and Fine [S1]. Each stage uses a "compact BA module" based on the BARF network [S1].

3.  **Pose Refinement Loop (per stage):**
    For each stage, optimize camera refine parameters $p \in se(3)$ [S1].
    *   Convert $p$ to a transform matrix $P_r = [R_r \ t_r]$ [S1].
    *   Update the current pose estimate. If the current pose is $\hat{P}$, the refined pose $Q$ is:
        $$ Q = [R_p \ R_n \ R_r \ t_p + t_n + t_r] $$
        [S1].
    *   Compute the photometric loss $\mathcal{L}_{photo}$ between rendered and input images.
    *   Backpropagate to update NeRF weights $\Theta$ and camera parameters $p$ [S1].

4.  **Learning Rate Schedule:**
    Use exponential interpolation with a modulation factor $\alpha$ to bias the learning rate towards the initial high value [S1]. The learning rate $\eta(t)$ at iteration $t$ is modulated by $\alpha$:
    $$ \eta(t) = \eta_{start} \cdot (\eta_{end}/\eta_{start})^{t/T} \cdot \alpha + \text{base\_decay} $$
    *(Note: The source describes the modulation factor as determining the degree of deviation from the initial value, with $\alpha=1.0$ corresponding to the original exponential rate [S1]. Exact formula implementation details for the modulation are not explicitly provided beyond the factor values [S1].)*
    Typical modulation factors: $\alpha = 10.0$ (Coarse), $\alpha = 3.0$ (Recursive), $\alpha = 1.0$ (Fine) [S1].

5.  **Stage Transitions:**
    *   **Coarse Stage:** Iterations = 20k [S1].
    *   **Recursive Stage:** Iterations = 20k [S1].
    *   **Fine Stage:** Iterations = 200k [S1].
    *   Image resolution during pose optimization is reduced to 400×400 (half of the 800×800 training resolution) to improve efficiency [S1].

6.  **Handling Incomplete Poses (Optional):**
    If 10% of poses are missing, assign an initial pose $P_{ini}$ to them. Jointly optimize poses for the known set $T$ and the missing set $F$ [S1].

7.  **Evaluation Alignment:**
    Apply Procrustes analysis to align optimized poses with ground-truth before evaluation to remove global coordinate system misalignment [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def se3_to_matrix(n):
    """Convert 6D Lie algebra vector to 4x4 SE3 matrix, then drop last row to 3x4."""
    # Implementation of exp_map from se3 to so3 and translation
    # Returns R, t
    R, t = exp_se3(n) 
    return np.hstack([R, t.reshape(3, 1)])

def exp_se3(n):
    """Placeholder for Lie algebra exponential map."""
    # n is 6D: [omega, v]
    # Returns rotation matrix R and translation vector t
    pass

def cbarf_optimize(images, initial_poses_P, noise_coeff=0.35, stages=[('coarse', 20000, 10.0), 
                                                                       ('recursive', 20000, 3.0), 
                                                                       ('fine', 200000, 1.0)]):
    """
    CBARF Optimization Loop.
    """
    # 1. Initialize Imperfect Poses
    # Generate noise n ~ N(0, I)
    noise = noise_coeff * np.random.randn(6)
    N = se3_to_matrix(noise)
    
    # Compose with reference pose P to get P_hat
    # P is 3x4 [Rp tp]
    # N is 3x4 [Rn tn]
    # P_hat = [Rp Rn tp + tn]
    Rp = initial_poses_P[:, :3]
    tp = initial_poses_P[:, 3:]
    Rn = N[:, :3]
    tn = N[:, 3:]
    
    P_hat = np.hstack([Rp @ Rn, tp + tn])
    
    current_poses = P_hat
    nerf_model = BARF_Backbone() # Initialize NeRF
    
    # 2. Cascaded Stages
    for stage_name, iterations, modulation_factor in stages:
        print(f"Starting {stage_name} stage")
        
        # Learning rate setup with modulation
        # The source states modulation factor biases learning towards initial high value.
        # Exact formula not given, assuming standard exponential decay modified by factor.
        lr_scheduler = ExponentialLR(modulation_factor=modulation_factor)
        
        for step in range(iterations):
            # 3. Pose Refinement
            # Optimize camera refine parameters p in se3
            p = np.random.randn(6) # In practice, this is a learnable parameter updated by gradient descent
            
            # Convert p to matrix Pr
            Pr = se3_to_matrix(p)
            
            # Compose with current poses
            # Q = [Rp Rn Rr tp + tn + tr]
            # current_poses is [Rp Rn tp + tn]
            # Pr is [Rr tr]
            # We need to multiply the rotation part and add translation
            
            # Extract current R and t from current_poses
            # current_poses = [R_curr t_curr]
            R_curr = current_poses[:, :3]
            t_curr = current_poses[:, 3:]
            
            Rr = Pr[:, :3]
            tr = Pr[:, 3:]
            
            # Update poses
            new_R = R_curr @ Rr
            new_t = t_curr + tr
            current_poses = np.hstack([new_R, new_t])
            
            # 4. Render and Loss
            # Render images using current_poses and nerf_model
            rendered_images = render(nerf_model, current_poses, images.shape)
            
            # Photometric loss
            loss = np.mean((rendered_images - images) ** 2)
            
            # 5. Backpropagate
            # Update nerf_model weights and p (camera parameters)
            # Note: p is updated via optimizer.step()
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            # Update learning rate
            lr_scheduler.step()
            
    # 6. Post-processing for evaluation
    # Procrustes alignment is done externally before metric calculation
    return current_poses, nerf_model
```

**Edge Cases and Pitfalls:**
*   **Initialization Sensitivity:** Poor initial poses can lead to local minima. The cascaded structure mitigates this by starting with coarse updates [S1].
*   **Coordinate Misalignment:** Optimized poses may have a different global coordinate system than ground truth. Procrustes analysis is required for evaluation [S1].
*   **Missing Poses:** If poses are missing, they must be initialized and jointly optimized, which requires careful handling of the loss gradient flow for those specific rays [S1].

## Operating characteristics
*   **Time Complexity:** The method is iterative. The coarse and recursive stages take 20k iterations each, while the fine stage takes 200k iterations [S1]. Total iterations are significantly higher than single-stage BA methods.
*   **Memory Complexity:** Uses half resolution (400×400) during pose optimization to improve efficiency, reducing memory footprint compared to full-resolution training [S1].
*   **Performance:** Successfully optimizes poses with large initial errors (noise coefficient 0.35, ~30.4° rotation deviation) where BARF overfits [S1]. Rendering quality (PSNR, SSIM, LPIPS) is comparable to ground-truth pose rendering [S1].
*   **Strengths:** Robust to large initial pose errors; progressive refinement prevents overfitting; handles incomplete poses.
*   **Limitations:** Computationally expensive due to multiple stages and high iteration counts in the fine stage; requires careful tuning of modulation factors [S1].

## Background
CBARF (Cascaded Bundle-Adjusting Neural Radiance Fields) was proposed to address the limitations of standard Bundle-Adjusting NeRF (BARF) when dealing with imperfect camera pose initialization [S1]. While BARF uses a coarse-to-fine frequency scheduling, CBARF introduces a cascaded BA structure with distinct coarse, recursive, and fine modules to progressively refine poses [S1]. The method was developed to effectively rectify erroneous poses that cannot be automatically optimized in standard BA processes [S1].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
