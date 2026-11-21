# MCMC Densification

Also called: Stochastic Gradient Langevin Dynamic (SGLD) Densification

## Overview
MCMC Densification is a Bayesian strategy that reformulates the densification of 3D Gaussian primitives as a Stochastic Gradient Langevin Dynamic (SGLD) update rule [S1, unofficial]. Unlike standard adaptive density control which relies on heuristic thresholds for cloning and pruning based on gradient magnitude, this method treats the set of 3D Gaussians as Markov Chain Monte Carlo (MCMC) samples [S2, unofficial]. It adds controlled noise to the center locations of Gaussians during optimization, allowing the representation to explore the parameter space more effectively [S1, unofficial]. This approach reduces reliance on the initial point cloud initialization and allows the target number of Gaussians to be specified prior to optimization [S2, unofficial].

## How it works
**Inputs:**
*   Current set of 3D Gaussians $\mathcal{G}$, where each Gaussian $g_i$ is defined by mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients.
*   Learning rate hyperparameter $\lambda_{lr}$ [S1, unofficial].
*   Noise scale hyperparameter $\lambda_{noise}$ [S1, unofficial].
*   Noise term $\epsilon$ sampled from a standard normal distribution [S1, unofficial].
*   Camera views $\mathcal{I}$ and corresponding ground truth images.

**Outputs:**
*   Updated set of 3D Gaussians $\mathcal{G}'$ with modified parameters, particularly the center locations $\mu$.

**Data Structures:**
*   The Gaussian parameters are stored in standard 3DGS format. The key modification is the application of noise to the mean vector $\mu$ after the gradient descent step.

**Steps:**
1.  **Rasterization and Loss:** Render the current Gaussian set $\mathcal{G}$ to obtain RGB images and alpha masks. Compute the total loss $\mathcal{L}_{\text{total}}$ (e.g., L1 + D-SSIM) against ground truth images [S1, unofficial].
2.  **Gradient Computation:** Compute the gradient of the loss with respect to the Gaussian parameters $\mathcal{G}$ [S1, unofficial].
3.  **SGLD Update:** Update the Gaussian parameters using the SGLD rule. The update rule for the Gaussian set $\mathcal{G}$ is:

    $$ \mathcal{G} \leftarrow \mathcal{G} - \lambda_{\text{lr}} \cdot \nabla_{\mathcal{G}} \mathbf{E}_{\mathcal{I}} [\mathcal{L}_{\text{total}} (\mathcal{G}; \mathcal{I})] + \lambda_{\text{noise}} \cdot \epsilon \quad (1) $$

    Where:
    *   $\lambda_{lr}$ is the learning rate [S1, unofficial].
    *   $\nabla_{\mathcal{G}} \mathbf{E}_{\mathcal{I}} [\mathcal{L}_{\text{total}} (\mathcal{G}; \mathcal{I})]$ is the expected gradient of the total loss over the image set [S1, unofficial].
    *   $\lambda_{noise}$ controls the magnitude of the injected noise [S1, unofficial].
    *   $\epsilon$ is the noise term [S1, unofficial].

4.  **Noise Application:** The noise term $\lambda_{\text{noise}} \cdot \epsilon$ is specifically applied to the center locations $\mu$ of the Gaussians [S1, unofficial]. This perturbs the positions, encouraging the Gaussians to move towards regions of high probability density in the data distribution, effectively performing densification through re-localization rather than explicit cloning [S2, unofficial].
5.  **Regularization:** Additional regularization terms may be added to the loss function to stabilize training [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcmc_densification_step(gaussians, cameras, gt_images, lambda_lr, lambda_noise):
    """
    Performs one step of MCMC densification.
    
    Args:
        gaussians: List of Gaussian objects with attributes .mu, .cov, .opacity, .sh.
        cameras: List of camera objects.
        gt_images: Ground truth RGB images.
        lambda_lr: Learning rate for gradient descent.
        lambda_noise: Scale factor for the Langevin noise.
        
    Returns:
        Updated gaussians list.
    """
    # 1. Rasterize and compute loss
    # Render current gaussians to get predicted images
    pred_images = rasterize(gaussians, cameras)
    
    # Compute total loss (e.g., L1 + SSIM)
    loss = compute_loss(pred_images, gt_images)
    
    # 2. Compute gradients
    # Compute gradients with respect to all Gaussian parameters
    grads = compute_gradients(loss, gaussians)
    
    # 3. Apply SGLD update
    for i, g in enumerate(gaussians):
        # Standard gradient descent step for all parameters
        g.mu = g.mu - lambda_lr * grads[i].mu
        g.cov = g.cov - lambda_lr * grads[i].cov
        g.opacity = g.opacity - lambda_lr * grads[i].opacity
        g.sh = g.sh - lambda_lr * grads[i].sh
        
        # Add noise to center locations (MCMC step)
        # epsilon is sampled from standard normal distribution N(0, 1)
        epsilon = sample_normal(g.mu.shape)
        g.mu = g.mu + lambda_noise * epsilon
        
    return gaussians
```

**Parameters:**
*   $\lambda_{lr}$: Learning rate. Typical values depend on the specific optimizer setup, but standard 3DGS learning rates apply [S1, unofficial].
*   $\lambda_{noise}$: Noise scale. This hyperparameter controls the exploration capability. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noise Magnitude:** If $\lambda_{noise}$ is too large, the Gaussians may diverge or fail to converge to a stable representation. If too small, the method degenerates to standard gradient descent without the benefits of MCMC sampling [S1, unofficial].
*   **Initial Point Cloud:** While MCMC densification reduces reliance on the initial point cloud, poor initialization can still lead to suboptimal local minima [S2, unofficial].
*   **Computational Cost:** The original implementation involves separate kernels for noise injection, which can be a bottleneck [S2, unofficial].

## Subtopics

### [Dynamic LoD Hierarchy Densification](10311-dynamic-lod-hierarchy-densification.md)
Dynamic LoD Hierarchy Densification is a strategy for maintaining and evolving a Level-of-Detail (LoD) hierarchy of 3D Gaussians continuously during the optimization phase of Gaussian Splatting training. It solves the problem of post-training hierarchy construction by allowing the hierarchy structure and Gaussian parameters to refine simultaneously as the scene representation evolves. Unlike methods that construct the hierarchy after training is complete, this approach supports stable, progressive refinement and restructuring without requiring a separate post-processing step. It builds on [MCMC Densification](9133-mcmc-densification.md) by applying similar spawning principles to hierarchical nodes.

### [HYDRA Densification](11088-hydra-densification.md)
HYDRA is a densification strategy for 3D Gaussian Splatting that decomposes complex, multi-lobed primitives into simpler constituents within the MCMC framework. It solves the problem of undersampled regions by subdividing primitives with high positional gradients while preserving the target distribution $P(g_{\text{new}}) \approx P(g_{\text{old}})$. It distinguishes itself from standard splitting by using a learned decomposition for large primitives, enabling generalizable planar primitives to operate effectively within MCMC.

### Fast MCMC Densification
Fast MCMC Densification is an optimized implementation of the MCMC densification approach from 3DGS-MCMC [39], which treats 3D Gaussians as MCMC samples and uses SGLD updates [S2, unofficial]. It fuses the noise injection step into a single CUDA kernel to reduce the training time bottleneck [S2, unofficial]. On the Mip-NeRF360 dataset, the optimized version achieves 28.00 PSNR, 6m17s training time, and 7.2GiB VRAM, compared to the original 27.83 PSNR, 27m22s training time, and 8.9GiB VRAM [S2, unofficial]. The target number of Gaussians for the nine Mip-NeRF360 scenes (alphabetical order) are: 6131954, 1244819, 1222956, 3636448, 5834784, 1852335, 1593376, 4961797, 3783761 [S2, unofficial]. The optimized version is significantly faster and uses less VRAM than the original implementation [39], while achieving similar or slightly better quality [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The original MCMC densification implementation has higher computational overhead due to separate kernels for noise injection, leading to longer training times and higher VRAM usage [S2, unofficial].
*   The optimized "Fast MCMC Densification" reduces training time significantly (from ~27m to ~6m on Mip-NeRF360) and reduces VRAM usage (from 8.9GiB to 7.2GiB) by fusing operations [S2, unofficial].

**Measured Performance:**
*   On the Mip-NeRF360 dataset, the optimized MCMC densification achieves a PSNR of 28.00, comparable to or slightly better than the original 27.83 PSNR [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Reduces reliance on the initial point cloud; allows specifying the number of Gaussians prior to optimization; leads to improved rendering quality [S2, unofficial].
*   **Limitations:** Requires careful tuning of the noise scale hyperparameter $\lambda_{noise}$; original implementation is computationally expensive [S1, unofficial][S2, unofficial].

**When to Use:**
*   Use when the initial point cloud is sparse or inaccurate, or when a specific target number of Gaussians is desired [S2, unofficial].
*   Use when rendering quality is prioritized over training speed, or when using the optimized implementation [S2, unofficial].

**When Not to Use:**
*   Use standard adaptive density control if training speed is critical and the initial point cloud is high quality [S2, unofficial].

## Background
MCMC Densification was proposed by Kheradmand et al. (2024) in the context of 3DGS-MCMC [S1, unofficial]. It adopts a Bayesian perspective to reformulate the densification strategy in Gaussian Splatting as a Stochastic Gradient Langevin Dynamic (SGLD) update rule [S1, unofficial]. This approach replaces the splitting, cloning, and pruning steps of adaptive density control with a re-localization scheme that preserves sample probability [S2, unofficial].

## Sources
[S1] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S2] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
