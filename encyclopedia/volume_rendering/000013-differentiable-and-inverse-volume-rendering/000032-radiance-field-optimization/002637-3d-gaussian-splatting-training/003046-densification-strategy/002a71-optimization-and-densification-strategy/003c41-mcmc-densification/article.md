# MCMC Densification

Also called: Stochastic Gradient Langevin Dynamic (SGLD) Densification

## Overview
MCMC Densification is a Bayesian densification strategy for 3D Gaussian Splatting that reformulates the adaptive density control process as a Stochastic Gradient Langevin Dynamic (SGLD) update rule [S1, unofficial]. Unlike the deterministic splitting and cloning heuristics of standard 3DGS, this method treats the set of 3D Gaussians as Markov Chain Monte Carlo (MCMC) samples, adding noise to center locations to explore the solution space [S2, unofficial]. It is distinguished by its ability to reduce reliance on the initial point cloud and allow for specifying the target number of Gaussians prior to optimization [S2, unofficial].

## How it works
**Inputs:**
*   Current set of 3D Gaussians $\mathcal{G}$, where each Gaussian $n$ has a mean $\mu_n$, covariance, opacity, and spherical harmonics coefficients.
*   A batch of training images $\mathcal{I}$ with known camera poses.
*   Hyperparameters: learning rate $\lambda_{lr}$, noise scale $\lambda_{noise}$, and noise term $\epsilon$.

**Outputs:**
*   Updated set of 3D Gaussians $\mathcal{G}'$ with modified parameters, particularly the center locations $\mu$.

**Data Structures:**
*   The Gaussian attributes are stored in contiguous arrays or structures of arrays (SoA) for efficient GPU access. The mean $\mu_n$ is a 3D vector. The noise $\epsilon$ is sampled from a standard normal distribution $\mathcal{N}(0, I)$ for each dimension of the mean.

**Steps:**
1.  **Rasterization and Loss:** Render the current Gaussian set $\mathcal{G}$ to obtain the predicted image. Compute the total loss $\mathcal{L}_{\text{total}}$ (e.g., L1 + SSIM) against the ground truth image.
2.  **Gradient Computation:** Compute the gradient of the loss with respect to the Gaussian parameters $\mathcal{G}$, specifically focusing on the gradients with respect to the mean locations $\nabla_{\mathcal{G}} \mathcal{L}_{\text{total}}$.
3.  **SGLD Update:** Update the Gaussian parameters using the SGLD rule. The update for the Gaussian set $\mathcal{G}$ is defined as:
    $$ \mathcal{G} \leftarrow \mathcal{G} - \lambda_{\text{lr}} \cdot \nabla_{\mathcal{G}} \mathbf{E}_{\mathcal{I}} [\mathcal{L}_{\text{total}} (\mathcal{G}; \mathcal{I})] + \lambda_{\text{noise}} \cdot \epsilon \quad (1) $$
    where:
    *   $\lambda_{lr}$ is the learning rate hyperparameter for the gradient descent step [S1, unofficial].
    *   $\nabla_{\mathcal{G}} \mathbf{E}_{\mathcal{I}} [\mathcal{L}_{\text{total}} (\mathcal{G}; \mathcal{I})]$ is the expected gradient of the loss over the image distribution [S1, unofficial].
    *   $\lambda_{noise}$ is the noise scale hyperparameter applied to the center locations $\mu$ [S1, unofficial].
    *   $\epsilon$ is the noise term added to the update, sampled from $\mathcal{N}(0, I)$ [S1, unofficial].
4.  **Re-localization:** Instead of explicit splitting or cloning, the noise injection effectively performs a re-localization scheme that tries to preserve sample probability [S2, unofficial]. Gaussians may be pruned or regenerated based on their contribution to the loss and the target density, though the core mechanism is the stochastic update [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcmc_densification_step(gaussians, images, camera_poses, lambda_lr, lambda_noise):
    """
    Performs one step of MCMC densification for 3D Gaussian Splatting.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mu: 3D mean position (tensor)
            - other attributes (covariance, opacity, etc.)
        images: Batch of ground truth RGB images.
        camera_poses: Camera pose matrices for the batch.
        lambda_lr: Learning rate for gradient descent.
        lambda_noise: Scale of the noise added to means.
        
    Returns:
        Updated gaussians list.
    """
    # 1. Rasterize and compute loss
    # Render the current Gaussians using the rasterizer
    rendered_images = rasterize(gaussians, camera_poses)
    
    # Compute total loss (e.g., L1 + SSIM)
    loss = compute_loss(rendered_images, images)
    
    # 2. Compute gradients
    # Compute gradients with respect to Gaussian parameters
    # Specifically, we need gradients for the mean positions mu
    gradients = compute_gradients(loss, gaussians)
    
    # 3. Sample noise
    # Sample noise epsilon from standard normal distribution for each Gaussian mean
    # Shape: (N, 3) where N is the number of Gaussians
    epsilon = torch.randn_like(gaussians.mu)
    
    # 4. Update Gaussians using SGLD rule
    # G <- G - lambda_lr * grad + lambda_noise * epsilon
    # Update mean positions
    new_mu = gaussians.mu - lambda_lr * gradients.mu + lambda_noise * epsilon
    
    # Update other parameters using standard gradient descent (or SGLD if specified)
    # Note: The source formula (1) applies the noise specifically to the center locations
    # Other parameters like covariance and opacity are typically updated via standard SGD
    # unless specified otherwise. We assume standard gradient descent for non-mean params.
    new_covariance = gaussians.covariance - lambda_lr * gradients.covariance
    new_opacity = gaussians.opacity - lambda_lr * gradients.opacity
    
    # Create new Gaussian objects with updated parameters
    updated_gaussians = []
    for i in range(len(gaussians)):
        new_gaussian = Gaussian(
            mu=new_mu[i],
            covariance=new_covariance[i],
            opacity=new_opacity[i],
            # Copy other attributes like SH coefficients
            sh_coefficients=gaussians[i].sh_coefficients
        )
        updated_gaussians.append(new_gaussian)
        
    return updated_gaussians
```

**Parameters:**
*   $\lambda_{lr}$: Learning rate. Typical values are not specified in the sources, but it follows standard SGD learning rate schedules [S1, unofficial].
*   $\lambda_{noise}$: Noise scale. Controls the magnitude of the stochastic perturbation. Typical values are not specified in the sources [S1, unofficial].
*   $\epsilon$: Noise term. Sampled from $\mathcal{N}(0, I)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise Magnitude:** If $\lambda_{noise}$ is too large, the Gaussians may diverge from the optimal solution. If too small, the method may behave like standard SGD and fail to explore the space effectively [S1, unofficial].
*   **Initial Point Cloud:** While less reliant than standard 3DGS, the initial distribution still affects convergence [S2, unofficial].
*   **Target Number of Gaussians:** The method allows specifying the target number of Gaussians, but the exact mechanism for enforcing this count (e.g., pruning thresholds) is not detailed in the core formula [S2, unofficial].

## Subtopics

### Fast MCMC Densification
An optimized implementation of the MCMC densification approach from 3DGS-MCMC [39], which treats 3D Gaussians as MCMC samples and uses SGLD updates [S2, unofficial]. It fuses the noise injection step into a single CUDA kernel to reduce the training time bottleneck [S2, unofficial]. On the Mip-NeRF360 dataset, the optimized version achieves 28.00 PSNR, 6m17s training time, and 7.2GiB VRAM, compared to the original 27.83 PSNR, 27m22s training time, and 8.9GiB VRAM [S2, unofficial]. The target number of Gaussians for the nine Mip-NeRF360 scenes (alphabetical order) are: 6131954, 1244819, 1222956, 3636448, 5834784, 1852335, 1593376, 4961797, 3783761 [S2, unofficial]. It is significantly faster and uses less VRAM than the original implementation while achieving similar or slightly better quality [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The original MCMC densification implementation has higher training time and VRAM usage compared to standard 3DGS due to the noise injection and re-localization steps [S2, unofficial].
*   The optimized "Fast MCMC Densification" reduces training time by approximately 4x (6m17s vs 27m22s) and VRAM usage by approximately 19% (7.2GiB vs 8.9GiB) on the Mip-NeRF360 dataset [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Reduces reliance on the initial point cloud, allows specifying the number of Gaussians prior to optimization, and leads to improved rendering quality [S2, unofficial].
*   **Limitations:** The original implementation is slower and uses more VRAM than standard 3DGS [S2, unofficial]. The hyperparameters $\lambda_{lr}$ and $\lambda_{noise}$ require tuning [S1, unofficial].

**When to Use:**
*   Use when the initial point cloud is sparse or poor quality, or when a specific target number of Gaussians is required [S2, unofficial].
*   Use the optimized version when training time and VRAM are constraints [S2, unofficial].

## Background
MCMC Densification was proposed by Kheradmand et al. (2024) in the context of 3DGS-MCMC [S1, unofficial]. It adopts a Bayesian perspective to reformulate Gaussian Splatting densification as a Stochastic Gradient Langevin Dynamic (SGLD) update rule [S1, unofficial]. The approach is motivated by the advantages of MCMC sampling in exploring the solution space and avoiding local minima associated with deterministic densification strategies [S2, unofficial].

## Sources
[S1] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S2] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
