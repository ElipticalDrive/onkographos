# Progressive Training

Also called: Coarse-to-Fine Hash Training, Incremental layer training, Layered model training, Level Activation Schedule, Multi-level Supervision Training, Progressive Training Strategy, Progressive Training for Hash Encodings

## Overview
Progressive training is a strategy for optimizing multi-resolution hash encodings that gradually activates higher resolution levels over the course of training [S1, S3]. It solves the problem of training instability inherent in hash-based neural fields: using only low-resolution grids leads to underfitting (smooth geometry lacking details), while using all high-resolution grids from the start causes overfitting to specific camera views, resulting in noise, artifacts, and view inconsistency [S1, S5]. By starting with coarse levels and incrementally introducing finer details, the method ensures view consistency and efficient convergence [S1, S5]. It differs from simultaneous multi-resolution training by freezing higher-resolution grids until their activation step and applying multi-rate-distortion supervision [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $L$ multi-resolution hash grids (typically $L=6$), a total number of training iterations $maxiter$, an activation interval $active\_iter$, and initial loss weights $\lambda_1, \lambda_2, \alpha_l$ [S3].
*   **Outputs:** Trained residual feature grids $R_t$ and reconstructed features $\hat{F}_t$ for the target frame or scene [S3].

**Data Structures:**
*   **Residual Feature Grids ($R_t^l$):** For each resolution level $l \in \{1, \dots, L\}$, a grid storing residual features that are recursively combined with previous frame features [S3].
*   **Reference Frame Buffer:** Stores reconstructed features $\hat{F}_t^{l-1}$ from previous frames or iterations to compute residuals [S3].
*   **Bandwidth Parameter ($\lambda$):** An integer scalar controlling the active resolution bandwidth [S1].

**Parameters:**
*   $L$: Number of resolution levels. Optimal value is 6 [S3].
*   $\lambda$: Initial value 2. Increases by 1 every 2.5% of total training steps [S1].
*   $\alpha_l$: MSE loss weight. $\alpha_6 = 1$, all other $\alpha_l = 0.25$ [S3].
*   $\lambda_1$: Rate loss weight. Set to $0.000001$ [S3].
*   $\lambda_2$: Regularization loss weight. Set to $0.000001$ [S3].
*   $active\_iter$: Iteration step determining activation timing. Set to 2,500 in HPC [S3].
*   $maxiter$: Total iterations. Set to 40,000 in HPC [S3].

**Algorithm:**
The method operates in two primary variants: the bandwidth-modulated approach (NeuS2) and the iterative residual activation approach (HPC) [S1, S3]. The following pseudocode implements the HPC residual activation loop, which is the most detailed implementation provided in the sources [S3].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_training(maxiter, active_iter, L):
    # 1. Initialize residual feature grids R[1..L]
    R = [ResidualGrid() for _ in range(L + 1)]  # 1-based indexing
    # 2. Initialize loss weights
    alpha = [0.0] + [0.25] * (L - 1) + [1.0]  # alpha[6] = 1, others 0.25
    lambda1, lambda2 = 0.000001, 0.000001
    # 3. Reference frame buffer
    F_hat_prev = [None] * (L + 1)
    # 4. Freeze higher resolution grids initially
    for l in range(2, L + 1):
        R[l].requires_grad = False
    R[1].requires_grad = True

    for it in range(1, maxiter + 1):
        # NeuS2 variant: Bandwidth update (every 2.5% of steps)
        if it % (maxiter // 40) == 0:
            lambda_bw += 1

        for l in range(1, L):
            # Activate next level at multiples of active_iter
            if it == l * active_iter:
                R[l + 1].requires_grad = True
                begin_entropy_estimation(R[l + 1])
            # Deactivate lower levels towards the end
            if it == maxiter - l * active_iter:
                R[L - l + 1].requires_grad = False
                if is_keyframe():
                    lambda1, lambda2 = 0, 0

        # Feature reconstruction
        F_hat_curr = [None] * (L + 1)
        for l in range(1, L + 1):
            F_hat_curr[l] = R[l] if l == 1 else F_hat_prev[l - 1] + R[l]

        # Loss calculation and optimization
        L_prev = 0
        for l in range(1, L + 1):
            if R[l].requires_grad:
                L_mse = sum((c(r) - F_hat_curr[l]) ** 2 for r in samples)
                L_reg = sum(abs(x) for x in R[l].flatten())
                L_rate = compute_rate(R[l])
                L_l = L_prev + alpha[l] * L_mse + lambda1 * L_rate + lambda2 * L_reg
                L_l.backward()
                update_parameters()
                L_prev = L_l

        F_hat_prev = F_hat_curr
```

**Edge Cases and Pitfalls:**
*   **Underfitting:** Using only low-resolution grids results in smooth geometry lacking high-frequency details [S1].
*   **Overfitting:** Using high-resolution grids from the start induces noise and artifacts, and causes the strong capacity of the grid to overfit predefined camera views, leading to inconsistent geometry across different views [S1, S5].
*   **View Consistency:** Starting with high-resolution grids produces inconsistent results across views; coarse-to-fine training mitigates this [S5].

## Subtopics

### Progressive Levels of Details (Coarse-to-Fine Optimization)
A variant used in Neuralangelo that progressively activates finer hash grids as the numerical gradient step size $\epsilon$ decreases [S6]. It avoids the need for fine grids to "unlearn" coarse features by linking grid activation to the smoothing effect of numerical gradients [S6].
*   **Steps:** 1. Initialize with a subset of coarse hash grids (4 for DTU, 8 for Tanks and Temples) [S6]. 2. Exponentially decrease step size $\epsilon$ [S6]. 3. Enable a new hash resolution every 5000 iterations when $\epsilon$ equals the spatial size of that resolution [S6].
*   **Parameters:** Hash grid resolution spans $2^5$ to $2^{11}$ with 16 levels [S6]. Max hash entries per resolution is $2^{22}$ [S6]. Channel size is 8 [S6]. Weight decay is applied to all parameters to prevent single-resolution features from dominating [S6].
*   **When to choose:** When using numerical gradients for surface normal computation and requiring high-fidelity surface reconstruction without auxiliary data [S6].

### Hierarchical Compensation Mechanism
A strategy that divides training into three levels, progressively increasing the resolution of tri-plane feature maps and context grids to capture fine scene details [S7].
*   **Steps:** Refine scene representation using structural features, then merge refined features $f_h = \sum f_{hi}$ from each level [S7].
*   **When to choose:** For large-scale unbounded scenes where structural similarity and global coherence are critical, such as in SplatCo [S7].

## Operating characteristics
Training time is distributed across levels, with lower computational cost per iteration in early stages due to frozen higher-resolution grids [S3]. However, total training time exceeds single-resolution training due to sequential optimization [S1]. Memory usage scales with the number of active levels, keeping peak memory lower than simultaneous multi-resolution training while still initializing all residual grids [S3]. The method achieves faster convergence and higher reconstruction quality, yielding improved Chamfer Distance and PSNR on datasets like DTU compared to static multi-resolution baselines [S4]. It prevents overfitting to camera views and ensures view consistency, but requires careful tuning of activation intervals and loss weights [S3]. It is best suited for hash-based neural fields on consumer GPUs where view consistency is critical, but should be avoided when training time is strictly prohibitive or scenes are trivially simple [S1, S5].

## Background
Progressive training for hash encodings originated in fast neural implicit surface reconstruction and volumetric video coding. The bandwidth-modulated approach was introduced in NeuS2 (2023) to improve training convergence and speed for multi-view reconstruction [S1]. The iterative residual activation approach with multi-rate-distortion loss was proposed in HPC (2024) for hierarchical progressive coding of volumetric video, leveraging temporal coherence [S3]. The concept of coarse-to-fine optimization using numerical gradient step sizes was adopted in Neuralangelo (2023) for high-fidelity surface reconstruction [S6]. The strategy addresses the empirical observation that static multi-resolution training suffers from underfitting at low levels and overfitting at high levels [S1, S5].

## Sources
[S1] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S2] L3GS: Layered 3D Gaussian Splats for Efficient 3D Scene Delivery, https://doi.org/10.1145/3680207.3723485, fetched 2026-10-27
[S3] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
[S4] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
[S5] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S6] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S7] SplatCo: Structure-View Collaborative Gaussian Splatting for Detail-Preserving Rendering of Large-Scale Unbounded Scenes, https://doi.org/10.48550/arXiv.2505.17951, fetched 2026-10-27
