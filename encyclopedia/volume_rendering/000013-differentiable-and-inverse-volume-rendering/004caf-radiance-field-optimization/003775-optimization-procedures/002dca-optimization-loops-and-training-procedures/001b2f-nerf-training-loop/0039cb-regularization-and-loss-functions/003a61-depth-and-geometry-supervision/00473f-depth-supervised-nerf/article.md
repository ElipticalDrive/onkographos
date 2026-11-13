# Depth-Supervised NeRF

Also called: DS-NeRF

## Overview
Depth-Supervised NeRF (DS-NeRF) is a training procedure that accelerates Neural Radiance Field optimization and improves quality under sparse-view conditions by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing [S1, S2][S3, unofficial]. Unlike methods that rely on data-driven priors from other scenes or semantic consistency, DS-NeRF uses the "free" depth estimates inherent to standard SfM pipelines (e.g., COLMAP) to enforce that the rendered ray termination distribution matches surface priors, thereby reducing overfitting to empty space and speeding up convergence [S1]. It is distinct from sibling methods like [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by explicitly modeling depth uncertainty as a Gaussian distribution and minimizing the Kullback-Leibler (KL) divergence between the rendered distribution and this noisy depth prior [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $\{I_j\}$ with known camera poses $\{P_j\}$ (estimated via SfM); a sparse 3D point cloud $\{X\}$ and visibility flags $\{X_j\}$ from the SfM solver; near and far clipping bounds $t_n, t_f$.
*   **Outputs:** A trained NeRF model (MLP mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$); rendered novel views.

**Data Structures:**
*   **Keypoint Data:** For each keypoint $x_i \in X$, store:
    *   3D position $\mathbf{x}_i \in \mathbb{R}^3$.
    *   Reprojection error variance $\hat{\sigma}_i^2 \in \mathbb{R}$, computed as the average reprojection error across all views where $x_i$ is detected [S1].
*   **Ray Data:**
    *   Standard training rays: origin $\mathbf{o}$, direction $\mathbf{d}$, pixel coordinates $(u,v)$.
    *   Keypoint rays: origin $\mathbf{o}_k$, direction $\mathbf{d}_k$ derived from keypoint projection.

**Numbered Steps:**

1.  **SfM Preprocessing:** Run an SfM solver (e.g., COLMAP) on the input images to estimate camera parameters and generate a sparse 3D point cloud $\{X\}$ with visibility information [S1][S3, unofficial].
2.  **Depth Estimation:** For each training image $I_j$ with camera $P_j$, identify visible keypoints $X_j \subset X$. For each visible keypoint $x_i \in X_j$, compute the estimated depth $D_{ij}$ by projecting $\mathbf{x}_i$ with $P_j$ and taking the re-projected z-value [S1].
3.  **Uncertainty Modeling:** Model the true depth of the first surface encountered by a ray as a random variable $D_{ij}$ normally distributed around the estimated depth $D_{ij}$ with variance $\hat{\sigma}_i^2$:
    $$ D_{ij} \sim \mathcal{N}(D_{ij}, \hat{\sigma}_i^2) $$
    [S1]
4.  **Ray Termination Distribution:** Compute the ray termination distribution $h_{ij}(t)$ from the NeRF rendering. This is defined as:
    $$ h(t) = T(t)\sigma(t) = \exp\left(-\int_0^t \sigma(s)ds\right)\sigma(t) $$
    where $\sigma(t)$ is the differential density at distance $t$ from the camera origin. $h(t)$ is a valid probability distribution over $t \in [t_n, t_f]$ assuming $\sigma(t) \ge 0$ and the far bound $t_f$ acts as an opaque wall [S1].
5.  **Depth Loss Calculation:** Minimize the KL divergence between the rendered ray distribution $h_{ij}(t)$ and the noisy depth distribution $\mathcal{N}(D_{ij}, \hat{\sigma}_i^2)$. The loss term $L_{Depth}$ is approximated as:
    $$ L_{Depth} = \mathbb{E}_{x_i \in X_j} \left[ -\log h(t) \exp \left( -\frac{(t - D_{ij})^2}{2\hat{\sigma}_i^2} \right) \right] dt \approx \mathbb{E}_{x_i \in X_j} \sum_k -\log h_k \exp \left( -\frac{(t_k - D_{ij})^2}{2\hat{\sigma}_i^2} \right) \Delta t_k $$
    where $t_k$ are sample points along the ray and $\Delta t_k$ is the step size [S1].
    *Note: An alternative MSE loss between rendered depth and $D_{ij}$ is possible but yields slightly more artifacts than KL divergence [S1][S3, unofficial].*
6.  **Keypoint Ray Generation:** To supervise depth and color simultaneously, generate rays corresponding to detected keypoints. In local camera coordinates, the ray direction is given by the image coordinates of the keypoint and $-1$ in the camera axis [S1].
7.  **RGB Interpolation:** Since image coordinates are continuous, interpolate RGB supervision values for the rays corresponding to detected keypoints from the ground truth image $I_j$ [S1].
8.  **Total Loss:** Combine the standard photometric color loss $L_{Color}$ and the depth loss:
    $$ L = L_{Color} + \lambda_D L_{Depth} $$
    where $\lambda_D$ is a hyper-parameter balancing the two terms [S1]. Typical training iterations range from 10k to 100k [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def compute_ray_termination_distribution(sigma_samples, delta_t):
    """
    Computes the ray termination distribution h(t) = T(t) * sigma(t).
    
    Args:
        sigma_samples: Tensor of shape (num_rays, num_samples), density values.
        delta_t: Tensor of shape (num_rays, num_samples), distance between samples.
        
    Returns:
        h: Tensor of shape (num_rays, num_samples), termination distribution weights.
        t: Tensor of shape (num_rays, num_samples), sample distances.
    """
    # Accumulated transmittance T(t) = exp(-sum(sigma * delta_t))
    # Note: delta_t is usually constant or pre-computed based on near/far bounds
    alpha = 1.0 - torch.exp(-sigma_samples * delta_t)
    transmittance = torch.cumprod(1.0 - alpha + 1e-10, dim=-1)
    # h(t) = T(t) * sigma(t) is not exactly correct for discrete sampling.
    # Correct discrete weight w_j = alpha_j * prod_{k<j} (1 - alpha_k)
    # However, the paper defines h(t) as the continuous density T(t)sigma(t).
    # For the loss, we use the log-probability of the distribution.
    # h_k approximates the probability mass at sample k.
    # Using the discrete approximation from the paper:
    # L_Depth ~ sum_k -log(h_k) * exp(-(t_k - D)^2 / 2s^2)
    
    # Reconstruct T(t) for the continuous definition h(t) = T(t)sigma(t)
    # T(t_k) = exp(-integral_0^t_k sigma(s) ds)
    # Approximate integral using cumulative sum of sigma * delta_t
    cum_sigma_delta = torch.cumsum(sigma_samples * delta_t, dim=-1)
    T = torch.exp(-cum_sigma_delta)
    h = T * sigma_samples
    
    # Normalize h to ensure it sums to 1 (optional but good for stability)
    # The paper assumes T(t_f) -> 0 (opaque wall), so sum(h) ~ 1.
    return h, cum_sigma_delta

def depth_supervised_nerf_loss(
    sigma_samples, 
    delta_t, 
    keypoint_depths, 
    keypoint_variances, 
    rgb_gt, 
    rgb_pred, 
    lambda_d=1.0
):
    """
    Computes the total loss for DS-NeRF.
    
    Args:
        sigma_samples: Tensor of shape (num_rays, num_samples), density values for rays.
        delta_t: Tensor of shape (num_rays, num_samples), step sizes.
        keypoint_depths: Tensor of shape (num_keypoint_rays), estimated depth D_ij.
        keypoint_variances: Tensor of shape (num_keypoint_rays), variance sigma_hat_i^2.
        rgb_gt: Tensor of shape (num_keypoint_rays, 3), ground truth RGB for keypoint rays.
        rgb_pred: Tensor of shape (num_keypoint_rays, 3), predicted RGB for keypoint rays.
        lambda_d: Float, weight for depth loss.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # 1. Color Loss for Keypoint Rays
    # Interpolate RGB for keypoint rays from ground truth image
    color_loss = F.mse_loss(rgb_pred, rgb_gt)
    
    # 2. Depth Loss for Keypoint Rays
    # Compute termination distribution h(t)
    h, _ = compute_ray_termination_distribution(sigma_samples, delta_t)
    
    # Get sample distances t_k for keypoint rays
    # Assuming delta_t is uniform or pre-computed for the specific rays
    # t_k = cumulative sum of delta_t
    t_samples = torch.cumsum(delta_t, dim=-1)
    
    # Compute the exponent term: -(t_k - D_ij)^2 / (2 * sigma_hat_i^2)
    # keypoint_depths shape: (num_keypoint_rays, 1) for broadcasting
    # keypoint_variances shape: (num_keypoint_rays, 1)
    exponent = -((t_samples - keypoint_depths.unsqueeze(1)) ** 2) / (2 * keypoint_variances.unsqueeze(1))
    
    # Compute log(h_k)
    # Add small epsilon to avoid log(0)
    log_h = torch.log(h + 1e-10)
    
    # Compute the integrand: -log(h_k) * exp(exponent)
    # Approximate integral by summing over samples
    depth_loss_per_ray = torch.sum(-log_h * torch.exp(exponent) * delta_t, dim=-1)
    
    # Average over keypoint rays
    depth_loss = torch.mean(depth_loss_per_ray)
    
    # 3. Total Loss
    total_loss = color_loss + lambda_d * depth_loss
    
    return total_loss
```

**Parameters:**
*   $\lambda_D$: Depth loss weight. Empirically set to 1 [S4].
*   $t_n, t_f$: Near and far clipping planes. $t_f$ is often treated as an opaque wall to ensure $\int_0^\infty \sigma(s)ds = \infty$ [S1].
*   Training Iterations: 10,000 to 100,000 [S3, unofficial].
*   Image Resolution: 400 × 300 (typical for experiments) [S1].
*   Keypoint Sparseness: Performance degrades with fewer keypoints but remains superior to baseline NeRF [S1].

**Edge Cases and Pitfalls:**
*   **Poor SfM:** Optimization fails if SfM produces inaccurate depth estimates or poor camera poses [S1].
*   **Gaussian Assumption:** The method assumes a Gaussian distribution models keypoint uncertainty, which may not hold for depth from non-SfM sources [S1].
*   **Domain Gap:** Methods pre-trained on specific datasets (e.g., DTU) may struggle on others (e.g., NeRF Real) due to domain differences, though DS-NeRF itself is general [S1].
*   **MVSNeRF Compatibility:** Adding depth supervision did not improve MVSNeRF because it already has a geometry prior from PSV-initialization [S1].

## Operating characteristics
*   **Time Complexity:** Training iterations are reduced by 2-3x compared to standard NeRF to reach the same PSNR [S1]. Wall-clock time per iteration is comparable (~362.4 ms vs ~359.8 ms on RTX A5000) [S1].
*   **Memory Complexity:** No significant additional memory overhead compared to standard NeRF, as it reuses SfM outputs and interpolates RGB for keypoint rays [S1].
*   **Performance:** On a 5-view setup, DS-NeRF achieves NeRF's peak test PSNR approximately 13 hours faster [S1]. With 2 views, DS-NeRF significantly outperforms NeRF, which often fails to converge to high quality [S1].
*   **Strengths:** Accelerates training, improves quality with few views, compatible with other NeRF variants (e.g., pixelNeRF, IBRNet), and generalizes to any scene where SfM succeeds [S1].
*   **Limitations:** Relies on the quality of SfM keypoints; poor depth estimates lead to optimization failure; assumes Gaussian uncertainty [S1].

## Background
*   **Origin:** Introduced by Deng et al. in 2022 [S1].
*   **History:** Developed to address the failure mode of NeRF fitting incorrect geometries with insufficient input views by leveraging "free" depth supervision from SfM pipelines [S1].
*   **Usage:** Applied to LLFF, NeRF Real, Redwood-3dscan, and DTU datasets [S1]. Compatible with depth from scanned sensors and RGB-D outputs [S1].

## Sources
[S1] Depth-supervised NeRF: Fewer Views and Faster Training for Free, https://doi.org/10.1109/cvpr52688.2022.01254, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S4] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
