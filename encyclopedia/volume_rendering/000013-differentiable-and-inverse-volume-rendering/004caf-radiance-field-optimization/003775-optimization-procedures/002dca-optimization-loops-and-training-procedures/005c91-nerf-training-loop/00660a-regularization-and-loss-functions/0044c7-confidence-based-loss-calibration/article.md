# Confidence-Based Loss Calibration

Also called: ICON, Incremental CONfidence

## Overview
Confidence-Based Loss Calibration dynamically weights the optimization of NeRF parameters and camera poses using estimated reliability metrics, addressing the bas-relief ambiguity and registration failures in joint optimization [S1]. It sets itself apart from static regularization methods by using pose confidence to weight NeRF gradients and ray confidence to weight pose gradients, effectively decoupling error sources during training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current NeRF parameters $\Theta$, current pose estimates $\hat{P}$, input images $I$, momentum factor $\beta$, re-initialization threshold $\lambda$, and window size $K$.
*   **Outputs:** Calibrated loss values $L_{NeRF}$, $L_{Pose}$, and updated parameters $\Theta, \hat{P}$.

**Data Structures:**
*   **Pose Confidence ($\zeta_P$):** A scalar value per pose, maintained as an exponential moving average.
*   **Ray Confidence ($\zeta_r$):** A scalar value per ray, computed from the current step's consistency metrics.

**Steps:**
1.  **Initialize:** Set initial pose confidences $\zeta_P^{(0)}$ based on initial registration quality.
2.  **Sample Rays:** For each pose $P_i$, sample a set of rays $\{r_{ji}\}_{j=1}^B$.
3.  **Compute Ray Confidence ($\zeta_r$):** Calculate a confidence metric $\zeta_{r_{ji}}$ for each ray based on consistency (e.g., photometric or geometric consistency) [S1].
4.  **Update Pose Confidence ($\zeta_P$):** Update the exponential moving average for pose $P_i$:
    $$ \zeta_{P_i}^{(t)} = \beta \zeta_{P_i}^{(t-1)} + (1 - \beta) \frac{1}{B} \sum_{j=1}^B \zeta_{r_{ji}} $$
    The momentum $\beta$ is typically 0.9 [S1].
5.  **Compute Calibrated Losses:**
    *   **NeRF Loss:** Weight the per-ray loss by the *pose* confidence $\zeta_{P_i}$:
        $$ L_{NeRF}(\Theta|\hat{P}, I) = \frac{\sum_i \sum_j L(r_{ji}) \zeta_{P_i}}{\sum_i \zeta_{P_i}} $$
    *   **Pose Loss:** Weight the per-ray loss by the *ray* confidence $\zeta_{r_{ji}}$:
        $$ L_{Pose}(\hat{P}|\Theta, I) = \frac{\sum_i \sum_j L(r_{ji}) \zeta_{r_{ji}}}{\sum_i \sum_j \zeta_{r_{ji}}} $$
    *   **Total Loss:**
        $$ L_{all}(\Theta, \hat{P}|I) = L_{NeRF} + L_{Pose} + L_{conf} $$
        Where $L_{conf}$ is an additional confidence regularization term [S1].
6.  **Optimize:** Backpropagate gradients through $L_{all}$ to update $\Theta$ and $\hat{P}$.
7.  **Pose Re-initialization (Guard):** If a new pose $P_i$ fails to register (abrupt confidence drop), restart from the previous pose. Failure is declared if:
    $$ \zeta_{P_i} \le \text{mean}(\{\zeta_{P_j}\}_{j=i-K}^{i-1}) - \lambda \cdot \text{std}(\{\zeta_{P_j}\}_{j=i-K}^{i-1}) $$
    Typical values: $\lambda = 2$, $K = 10$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def confidence_calibrated_step(nerf_params, poses, images, beta=0.9, lam=2.0, K=10):
    L_neRF = 0.0
    L_pose = 0.0
    sum_pose_conf = 0.0
    sum_ray_conf = 0.0
    ray_losses = []
    ray_confs = []
    pose_confs = []

    for i, P_i in enumerate(poses):
        # 1. Sample rays for current pose
        rays = sample_rays(P_i, images[i])
        
        # 2. Compute per-ray loss and raw confidence
        current_ray_losses = []
        current_ray_confs = []
        for r in rays:
            # Render and compute basic loss L(r)
            loss_r = compute_rendering_loss(nerf_params, P_i, r, images[i])
            # Compute ray confidence (e.g., consistency metric)
            conf_r = compute_ray_confidence(nerf_params, P_i, r, images[i])
            
            current_ray_losses.append(loss_r)
            current_ray_confs.append(conf_r)
            
            # Accumulate for Pose Loss (weighted by ray confidence)
            L_pose += loss_r * conf_r
            sum_ray_conf += conf_r

        # 3. Update Pose Confidence (Exponential Moving Average)
        mean_ray_conf = sum(current_ray_confs) / len(current_ray_confs)
        prev_conf = pose_confs[i] if i < len(pose_confs) else mean_ray_conf
        new_conf = beta * prev_conf + (1 - beta) * mean_ray_conf
        pose_confs.append(new_conf)
        
        # 4. Accumulate for NeRF Loss (weighted by pose confidence)
        for loss_r in current_ray_losses:
            L_neRF += loss_r * new_conf
        sum_pose_conf += new_conf

    # 5. Normalize Losses
    if sum_pose_conf > 0:
        L_neRF /= sum_pose_conf
    if sum_ray_conf > 0:
        L_pose /= sum_ray_conf

    # 6. Pose Re-initialization Check
    if i >= K:
        window = pose_confs[i-K:i]
        mean_w = sum(window) / len(window)
        std_w = sqrt(sum((x - mean_w)**2 for x in window) / len(window))
        if new_conf < mean_w - lam * std_w:
            # Restart from previous pose
            poses[i] = poses[i-1]
            # Re-compute or reset confidence for this step
            pass

    # 7. Add Confidence Regularization (L_conf)
    L_conf = compute_confidence_regularization(pose_confs, ray_confs)
    
    total_loss = L_neRF + L_pose + L_conf
    return total_loss, nerf_params, poses
```

## Subtopics

### Learnable Confidence Combination
A variant that computes a robust confidence map by defining learnable weights $\omega_i$ for individual confidence metrics (e.g., RGB, SSIM, VGG, Depth, Flow) and jointly optimizing them during training [S2, unofficial]. The final confidence map is $\hat{C} = \sum_i \omega_i C_i$, subject to $\sum_i \omega_i = 1$ [S2, unofficial]. This allows the model to automatically focus on the most reliable confidence signal for a given scene or region [S2, unofficial].

## Operating characteristics
*   **Complexity:** Adds minimal overhead to the training loop, primarily involving the computation of ray/pose confidence metrics and simple averaging operations.
*   **Strengths:** Mitigates the bas-relief ambiguity by down-weighting ambiguous poses [S1]. Prevents error propagation from poorly registered images by re-initializing failed poses [S1].
*   **Limitations:** Requires careful tuning of hyperparameters $\beta$, $\lambda$, and $K$ [S1]. Relies on the quality of the underlying confidence metrics [S1][S2, unofficial].
*   **When to use:** Joint pose and NeRF optimization where initial pose estimates may be noisy or registration fails [S1].

## Background
Introduced in the ICON (Incremental CONfidence) paper [S1], this method draws inspiration from trial-and-error registration mechanisms in incremental SfM [S1]. It addresses the bas-relief ambiguity, a common issue in monocular NeRF optimization where depth and scale are coupled [S1].

## Sources
[S1] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
[S2] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
