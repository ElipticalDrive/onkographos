# Optimization and Training Strategy

Also called: 3DGS-MCMC Training Strategy, MCMC-based Training with EMA and Regularization

## Overview
This strategy defines the optimization schedule, loss formulation, and regularization techniques for training radiance fields using 3D Gaussian Splatting with MCMC-based densification [S1]. It addresses the problem of overfitting to noise and geometric artifacts during the iterative refinement of Gaussian primitives [S1, S2]. It distinguishes itself from standard 3DGS by incorporating specific MCMC-derived regularization terms on opacity and scale, using cosine annealing for feature/MLP learning rates, applying Exponential Moving Average (EMA) to MLP weights, and executing a final refinement phase that freezes geometry [S1].

## How it works
The method optimizes a set of 3D Gaussian primitives $P$ to minimize a composite loss function while managing the stability of learned features and MLP weights.

**Inputs:**
- Initial set of Gaussian primitives (position, scale, rotation, opacity, features).
- Training images and camera poses.
- Hyperparameters: $\lambda$ (D-SSIM weight), $\lambda_\alpha$ (opacity reg weight), $\lambda_s$ (scale reg weight), $\gamma$ (EMA decay), $T_{refine}$ (refinement iterations, typically 3000) [S1].

**Outputs:**
- Optimized Gaussian primitives and associated MLP weights.

**Data Structures:**
- Primitives: Array of size $P$. Each primitive $i$ has fields: position $\mathbf{p}_i$, scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, opacity $\alpha_i$, feature vector $\mathbf{f}_i$.
- MLP Weights: $\theta$, filtered version $\bar{\theta}$ [S1].

**Formulas:**
1.  **Opacity Regularization:**
    $$ \mathcal{R}_\alpha = \frac{1}{P} \sum_{i=1}^P \alpha_i $$
    where $P$ is the number of primitives [S1].

2.  **Scale Regularization:**
    $$ \mathcal{R}_s = \frac{1}{P} \sum_{i=1}^P \|\mathbf{s}_i\|_1 $$
    where $\mathbf{s}_i$ is the scale vector of primitive $i$ [S1].

3.  **Total Loss:**
    $$ \mathcal{L} = (1 - \lambda) \mathcal{L}_{L_1} + \lambda \mathcal{L}_{\text{D-SSIM}} + \lambda_\alpha \mathcal{R}_\alpha + \lambda_s \mathcal{R}_s $$
    where $\mathcal{L}_{L_1}$ is the L1 photometric loss, $\mathcal{L}_{\text{D-SSIM}}$ is the differentiable SSIM loss, and $\lambda, \lambda_\alpha, \lambda_s$ are scalar weights [S1].

4.  **EMA Update for MLP Weights:**
    $$ \bar{\theta}_t \leftarrow \gamma \bar{\theta}_{t-1} + (1 - \gamma) \theta_t $$
    where $\bar{\theta}_t$ is the filtered weight at step $t$, $\theta_t$ is the raw updated weight, and $\gamma$ is the decay factor [S1].

**Steps:**
1.  **Initialization:** Initialize Gaussian primitives and MLP weights. Set EMA weights $\bar{\theta}_0 = \theta_0$ [S1].
2.  **Main Training Loop:** For each training iteration $t$:
    a. Render the scene using the current Gaussian primitives and MLP weights.
    b. Compute the total loss $\mathcal{L}$ using the formula above [S1].
    c. Perform gradient descent to update primitive parameters (position, scale, rotation, opacity) and raw MLP weights $\theta_t$ [S1].
    d. Update the EMA weights $\bar{\theta}_t$ using the EMA formula [S1].
    e. Use a cosine annealing scheduler to determine the learning rates for feature vectors and MLP weights at step $t$ [S1].
    f. Apply the MCMC-based densification strategy (spawn/prune) to manage the set of primitives $P$ [S1].
3.  **Refinement Phase:** After the main training loop, execute the last $T_{refine}$ iterations (typically 3000) with the following modifications:
    a. Disable all regularization terms ($\mathcal{R}_\alpha, \mathcal{R}_s$) [S1].
    b. Freeze all primitive parameters (position, scale, rotation, opacity) [S1].
    c. Optimize only the feature vectors and MLP weights $\theta$ [S1].
    d. Continue using the cosine annealing scheduler for learning rates [S1].

**Parameters:**
- $\lambda$: Weight for D-SSIM loss. Not given in the sources.
- $\lambda_\alpha$: Weight for opacity regularization. Not given in the sources.
- $\lambda_s$: Weight for scale regularization. Not given in the sources.
- $\gamma$: EMA decay factor. Not given in the sources.
- $T_{refine}$: Number of refinement iterations. Typically 3000 [S1].

**Edge Cases and Pitfalls:**
- **Overfitting:** Without EMA and regularization, the model may overfit to individual frames or geometric artifacts [S1, S2].
- **Local Minima:** In unconditional generation contexts, failing to reset latent representations can cause the model to get stuck in local minima that overfit geometric artifacts [S2].
- **Precision:** Half-precision memory fetches on feature vectors are used during rasterization to reduce register pressure, which may introduce precision trade-offs [S1].

## Subtopics

### Latent Resetting
A training procedure used in unconditional 3D generation (e.g., SSDNeRF) to prevent overfitting to geometric artifacts by periodically resetting latent representations to their mean [S2]. It involves resetting triplane codes to their mean value halfway through the training schedule and doubling the total training length to compensate [S2]. Without this, the model risks getting stuck in local minima [S2]. On the SRN Cars dataset, this approach achieves an FID of 19.04±1.10 and a KID/10⁻³ of 8.28±0.60 [S2].

## Operating characteristics
- **Time Complexity:** The refinement phase adds a fixed cost of 3000 iterations focused only on features and MLP weights, which is relatively cheap compared to full primitive optimization [S1].
- **Memory Complexity:** Uses half-precision memory fetches for feature vectors to reduce register pressure, lowering memory bandwidth requirements during rasterization [S1].
- **Strengths:** Enhances robustness to noise and avoids overfitting to individual frames through EMA and regularization [S1]. The refinement step slightly improves color fidelity, particularly in large-scale scenes [S1].
- **Limitations:** Requires careful tuning of regularization weights and EMA decay. The densification strategy relies on the underlying 3DGS-MCMC method, which may have its own computational overhead [S1].
- **When to use:** Suitable for high-quality radiance-field novel-view synthesis, semantic scene reconstruction, and high-resolution image fitting where noise robustness and color fidelity are critical [S1].
- **When not to use:** Not applicable for unconditional 3D generation tasks that require latent resetting (use Latent Resetting variant instead) [S2].

## Background
The optimization strategy is based on the 3DGS-MCMC method, which reinterprets the Gaussian cloud as samples from an underlying probability distribution [S1]. The specific regularization terms and EMA application were introduced in "Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction" [S1]. The latent resetting variant was introduced in "Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction" for unconditional generation tasks [S2].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
