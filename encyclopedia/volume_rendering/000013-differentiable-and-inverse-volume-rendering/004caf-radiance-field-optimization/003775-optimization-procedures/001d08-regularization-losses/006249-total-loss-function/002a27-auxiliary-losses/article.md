# Auxiliary Losses

Also called: Background entropy loss, Per-point RGB loss

## Overview
Auxiliary losses are regularization terms added to the total training objective in differentiable volume rendering to improve rendering quality, prevent overfitting, and enforce physical plausibility in particle trajectories [S2]. They distinguish themselves from the primary photometric reconstruction loss by supervising intermediate representations, such as per-point color or background probability distributions, rather than only the final rendered image [S1, S2]. This method builds on [Total Loss Function](4177-total-loss-function.md) by providing specific components for stability and structure [S1, S2].

## How it works
The method combines two primary categories of auxiliary losses: per-point supervision and background/entropy regularization [S1]. In dynamic particle-based methods, additional motion and feature continuity losses are applied [S2].

**Inputs and Outputs:**
*   **Inputs:** Per-point radiance/color $C_i$, per-point opacity $\alpha_i$, feature field $F$, particle positions $P$, and camera parameters [S1, S2].
*   **Outputs:** Scalar loss values $L_{ptrgb}$, $L_{bg}$, $L_{tvf}$, $L_{tvm}$ to be summed into the total loss [S2].

**Data Structures:**
*   **Per-point Color:** A tensor of shape $(N, 3)$ containing RGB values for each sampled point or particle $i \in \{1, \dots, N\}$ [S1, S2].
*   **Background Probability:** A scalar representing the estimated probability mass assigned to the background (empty space) versus foreground [S1].

**Numbered Steps:**

1.  **Compute Per-Point RGB Loss ($L_{ptrgb}$):**
    Supervise the color emission of each query point directly, rather than the accumulated color [S1].
    $$ L_{ptrgb} = \frac{1}{N} \sum_{i=1}^{N} \| C_i - C_{gt, i} \|_1 $$
    Where $C_i$ is the predicted color at point $i$, and $C_{gt, i}$ is the ground truth color [G]. This term is weighted by $w_1$ [S2].

2.  **Compute Background Entropy Loss ($L_{bg}$):**
    Enforce that the rendered background probability concentrates on either the background or foreground, reducing ambiguity in empty space [S1].
    $$ L_{bg} = -P_{bg} \log(P_{bg}) - (1-P_{bg}) \log(1-P_{bg}) $$
    Where $P_{bg}$ is the background probability [G]. This term is weighted by $w_2$ [S2].

3.  **Compute Total Variation Loss for Feature Field ($L_{tvf}$):**
    Enhance the continuity of the feature field by penalizing large gradients between adjacent voxels or particles [S2].
    $$ L_{tvf} = \sum_{i} \| \nabla F_i \|_1 $$
    Weighted by $w_3$ [S2].

4.  **Compute Motion Regularization Loss ($L_{tvm}$):**
    Encourage particles to preserve local rigidity, reducing spurious trajectories in empty areas [S2].
    $$ L_{tvm} = \sum_{i,j \in \text{neighbors}} \| (P_i - P_j) - (P_i^t - P_j^t) \|_2 $$
    Where $P_i$ are particle positions at the current time and $P_i^t$ are positions at the previous time [G]. Weighted by $w_4$ [S2].

**Parameters with Typical Values:**
*   $w_1$ (for $L_{ptrgb}$): Follows common settings from [38], [5] (specific value not given in sources) [S2].
*   $w_2$ (for $L_{bg}$): Follows common settings from [38], [5] [S2].
*   $w_3$ (for $L_{tvf}$): $0.01$ [S2].
*   $w_4$ (for $L_{tvm}$): $0.01$ [S2].

**Edge Cases and Pitfalls:**
*   **Weight Sensitivity:** Using larger weights for $L_{tvf}$ and $L_{tvm}$ (e.g., $0.1$) hinders model learning [S2].
*   **Lower Bound:** Using smaller weights ($0.001$) yields lower PSNR ($32.73$–$33.84$) compared to $0.01$ ($34.01$–$34.06$) [S2].
*   **Monocular Limitations:** These losses are particularly effective in preventing overfitting in monocular videos with limited multi-view constraints [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_auxiliary_losses(per_point_colors, per_point_colors_gt,
                             background_probability, feature_field,
                             particle_positions, previous_positions):
    # 1. Per-point RGB Loss
    ptrgb_loss = sum(abs(c - cg) for c, cg in zip(per_point_colors, per_point_colors_gt)) / len(per_point_colors)
    w1 = 1.0  # [G] Base weight from common settings [S2]
    loss_ptrgb = w1 * ptrgb_loss

    # 2. Background Entropy Loss
    eps = 1e-7
    p = max(eps, min(1 - eps, background_probability))
    entropy = -(p * log(p) + (1 - p) * log(1 - p))
    loss_bg = abs(entropy)  # [G] Minimize entropy by taking absolute value
    w2 = 1.0  # [G] Base weight from common settings [S2]
    loss_bg_term = w2 * loss_bg

    # 3. Total Variation Loss for Feature Field
    tv_loss = 0.0
    if len(feature_field.shape) == 4:  # Grid: (N, D, H, W)
        for n in range(feature_field.shape[0]):
            for d in range(feature_field.shape[1]):
                for h in range(feature_field.shape[2] - 1):
                    for w in range(feature_field.shape[3] - 1):
                        tv_loss += abs(feature_field[n, d, h+1, w] - feature_field[n, d, h, w])
                        tv_loss += abs(feature_field[n, d, h, w+1] - feature_field[n, d, h, w])
    w3 = 0.01
    loss_tvf = w3 * tv_loss

    # 4. Motion Regularization Loss
    loss_tvm = 0.0
    if previous_positions is not None:
        for i in range(len(particle_positions)):
            for j in range(i + 1, len(particle_positions)):
                disp_curr = [particle_positions[i][k] - particle_positions[j][k] for k in range(3)]
                disp_prev = [previous_positions[i][k] - previous_positions[j][k] for k in range(3)]
                loss_tvm += sum(abs(d - dp) for d, dp in zip(disp_curr, disp_prev))
    w4 = 0.01
    loss_tvm_term = w4 * loss_tvm

    return loss_ptrgb + loss_bg_term + loss_tvf + loss_tvm_term
```

## Subtopics

### [Entropy Loss for Floater Cleaning](14625-entropy-loss-for-floater-cleaning.md)
Entropy loss is a regularization technique used in differentiable volume rendering (specifically 3D Gaussian Splatting) to encourage Gaussian opacities to converge to binary values of either 0 or 1. By forcing opacities to be binary, the method enables the automatic removal of "floaters"—small, noisy, or irrelevant Gaussians—during the training pruning pipeline. It distinguishes itself from sparsity-based masking losses by operating directly on the opacity parameter to enforce binary states rather than volume-based elimination.

## Operating characteristics
*   **Time Complexity:** The addition of auxiliary losses increases training time slightly due to extra gradient computations; for example, in Direct Voxel Grid Optimization, training time increased from 6.2 to 7.3 minutes for a specific configuration [S1].
*   **Memory Complexity:** Negligible increase, as it primarily involves scalar loss terms and small intermediate tensors [G].
*   **Measured Performance:** In Direct Voxel Grid Optimization, adding both auxiliary losses yielded a PSNR of 35.54, compared to 35.37 (none) or 35.53 (one loss) [S1]. In Dynamic Appearance Particle Neural Radiance Field, the full loss configuration yielded an average PSNR of 34.06 on the D-NeRF dataset; removing $L_{tvm}$ dropped PSNR to 33.79, and removing $L_{tvf}$ dropped it to 33.98 [S2].
*   **Strengths:** Prevents overfitting, especially in monocular videos; cleans up floaters; enforces physical plausibility in dynamic scenes [S1, S2].
*   **Limitations:** Sensitive to weight hyperparameters; requires careful tuning to avoid hindering learning [S2].
*   **When to Use:** When dealing with monocular videos, dynamic scenes, or when floaters/spurious trajectories are observed [S1, S2].
*   **When Not to Use:** When computational resources are extremely limited and marginal PSNR gains are not worth the training overhead; when multi-view constraints are strong enough to prevent overfitting without auxiliary terms [G].

## Background
Auxiliary losses in volume rendering originated from the need to stabilize training and improve generalization in neural radiance fields (NeRF) and 3D Gaussian Splatting [S1, S2]. The use of per-point RGB and background entropy losses was formalized in Direct Voxel Grid Optimization [S1]. The use of feature continuity and motion regularization losses was formalized in Dynamic Appearance Particle Neural Radiance Field [S2]. These methods build on earlier regularization techniques in neural rendering [G].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] Dynamic Appearance Particle Neural Radiance Field, https://doi.org/10.1109/tcsvt.2025.3540792, fetched 2026-11-05
