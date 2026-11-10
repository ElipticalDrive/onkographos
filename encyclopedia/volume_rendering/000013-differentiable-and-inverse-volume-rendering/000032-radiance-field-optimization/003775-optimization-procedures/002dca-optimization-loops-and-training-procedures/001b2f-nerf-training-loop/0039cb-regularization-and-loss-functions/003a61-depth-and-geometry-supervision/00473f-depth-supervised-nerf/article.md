# Depth-Supervised NeRF

Also called: DS-NeRF

## Overview
Depth-Supervised NeRF (DS-NeRF) is a training procedure that accelerates Neural Radiance Field optimization and improves quality with sparse input views by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing [S1]. Unlike methods that rely on data-driven priors from category-specific training data (e.g., pixelNeRF, metaNeRF) or semantic consistency, DS-NeRF only requires the existence of 3D keypoints, making it flexible and not dependent on training data distribution assumptions [S1]. It enforces that the ray termination distribution matches surface priors, reducing overfitting and speeding up training by 2-3x [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images $I_j$ with known camera poses $P_j$ (estimated via SfM/COLMAP), sparse 3D keypoints $X$ with visibility flags, and depth uncertainty estimates $\hat{\sigma}_i$ for each keypoint [S1].
*   **Outputs:** A trained NeRF model (MLP mapping 3D coordinates and viewing directions to density $\sigma$ and color $\mathbf{c}$) and rendered novel views [S1].

**Data Structures:**
*   **Keypoints:** A set $X = \{x_1, x_2, \dots\} \subset \mathbb{R}^3$ [S1].
*   **Visibility:** For each image $j$, a subset $X_j \subset X$ of visible keypoints [S1].
*   **Depth Estimates:** For each keypoint $x_i \in X_j$, a scalar depth $D_{ij}$ and variance $\hat{\sigma}_i^2$ [S1].
*   **Ray Sampling:** Standard NeRF sampling points $t_k$ along rays, with step size $\Delta t_k$ [S1].

**Numbered Steps:**

1.  **SfM Preprocessing:** Run an SfM solver (e.g., COLMAP) on the input images to estimate camera parameters $P_j$ and sparse 3D keypoints $X$ [S1][S2].
2.  **Depth Estimation:** For each image $I_j$ with camera $P_j$, identify visible keypoints $X_j$ [S1]. For each $x_i \in X_j$, compute the projected depth $D_{ij}$ by projecting $x_i$ with $P_j$ and taking the re-projected z-value [S1].
3.  **Uncertainty Calculation:** Compute the depth uncertainty $\hat{\sigma}_i$ for each keypoint $x_i$ as the average reprojection error across all views where the keypoint was detected [S1].
4.  **Ray Generation:** For each training iteration, allocate a portion of the total training rays to "keypoint rays" [S1]. Calculate ray origins in local camera coordinates from the image coordinates of detected keypoints, setting the camera axis coordinate to -1 [S1]. Shoot these rays into the scene [S1].
5.  **Simultaneous Supervision:** To train with both color and depth supervision in a single ray marching step, exploit the continuity of image coordinates to interpolate RGB supervision for rays corresponding to detected keypoints [S1]. Interpolate RGB supervision values for the rays corresponding to detected keypoints, since image coordinates are continuous and pixels are discrete samples [S1]. Simultaneously supervise RGB and depth for these rays within a single ray marching procedure [S1].
6.  **Ray Termination Distribution:** Compute the ray termination distribution $h_{ij}(t)$ from the NeRF rendering, defined as $h(t) = T(t)\sigma(t)$, where $T(t) = \exp(-\int_0^t \sigma(s)ds)$ [S1]. Note that $h(t)$ is a valid probability distribution if $\sigma(s) \ge 0$ and $\int_0^\infty \sigma(s)ds = \infty$ (often enforced by treating the far bound $t_f$ as an opaque wall) [S1].
7.  **Depth Loss Calculation:** Model the true depth of the first surface as a random variable $D_{ij} \sim \mathcal{N}(D_{ij}, \hat{\sigma}_i)$ [S1]. Minimize the KL divergence between the rendered ray distribution $h_{ij}(t)$ and the noisy depth distribution [S1]. The loss is approximated as:
    $$ L_{Depth} = \mathbb{E}_{x_i \in X_j} \sum_k -\log h_k \exp \left( -\frac{(t_k - D_{ij})^2}{2\hat{\sigma}_i^2} \right) \Delta t_k $$
    where $h_k = h(t_k)$ and $t_k$ are sample points along the ray [S1]. Alternatively, an MSE loss between the rendered depth distribution and the SFM-estimated depth can be used, though KL divergence yields slightly better values with fewer artifacts [S1].
8.  **Total Loss:** Combine color and depth losses:
    $$ L = L_{Color} + \lambda_D L_{Depth} $$
    where $L_{Color} = || \mathbf{C} - I ||_2^2$ and $\lambda_D$ is a hyperparameter [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_depth_loss(keypoint_rays, nerf_model, near_bound, far_bound, num_samples=128):
    """
    Computes the depth supervision loss for a batch of keypoint rays.
    
    Args:
        keypoint_rays: List of dicts with keys 'origin', 'direction', 'gt_color', 'gt_depth', 'depth_sigma'.
        nerf_model: Standard NeRF MLP callable that takes (points, direction) and returns (sigma, color).
        near_bound: t_n, near clipping plane.
        far_bound: t_f, far clipping plane (treated as opaque wall).
        num_samples: Number of uniform samples per ray.
        
    Returns:
        depth_loss: Scalar float representing the KL divergence loss.
    """
    total_loss = 0.0
    num_rays = len(keypoint_rays)
    step = (far_bound - near_bound) / num_samples

    for ray in keypoint_rays:
        origin = ray['origin']
        direction = ray['direction']
        gt_color = ray['gt_color']
        D_ij = ray['gt_depth']
        sigma_i = ray['depth_sigma']

        # 1. Sample points along the ray (uniform + jitter)
        t_vals = [near_bound + (i + 0.5 + random.uniform(0, 1)) * step for i in range(num_samples)]

        # 2. Query NeRF for density and color at sample points
        points = [[origin[d] + t_vals[k] * direction[d] for d in range(3)] for k in range(num_samples)]
        sigma_raw, color = nerf_model(points, direction)
        # Enforce non-negativity for density [G]
        sigma = [max(0.0, s) for s in sigma_raw]

        # 3. Compute transmittance and weights
        # alpha_k = 1 - exp(-sigma_k * delta_t)
        alpha = [1.0 - math.exp(-sigma[k] * step) for k in range(num_samples)]
        # T_k = product_{j<k} (1 - alpha_j)
        transmittance = [1.0]
        for k in range(num_samples):
            transmittance.append(transmittance[-1] * (1.0 - alpha[k] + 1e-10))
        transmittance = transmittance[:-1]

        # Weights w_k = alpha_k * T_k
        weights = [alpha[k] * transmittance[k] for k in range(num_samples)]

        # 4. Rendered Color (for color supervision)
        rendered_color = [sum(weights[k] * color[k][c] for k in range(num_samples)) for c in range(3)]

        # 5. Rendered Depth Distribution h(t)
        # h(t) = T(t) * sigma(t). Discrete approximation: h_k = T_k * sigma_k
        h_k = [transmittance[k] * sigma[k] for k in range(num_samples)]

        # 6. Compute Depth Loss
        # L_Depth = sum_k -log(h_k) * exp( - (t_k - D_ij)^2 / (2 * sigma_i^2) ) * delta_t_k
        loss_term = 0.0
        for k in range(num_samples):
            exponent = -((t_vals[k] - D_ij)**2) / (2 * sigma_i**2)
            exponent = max(exponent, -100.0) # Clip for numerical stability
            term = -math.log(h_k[k] + 1e-10) * math.exp(exponent) * step
            loss_term += term
        total_loss += loss_term

    return total_loss / num_rays
```

**Parameters:**
*   $\lambda_D$: Hyperparameter balancing color and depth supervision. Not given in the sources.
*   $t_n, t_f$: Near and far clipping planes. $t_f$ is often treated as an opaque wall [S1].
*   Training iterations: O(10k - 100k) [S2].
*   Image resolution: 400 × 300 for training and evaluation [S1].

**Edge Cases and Pitfalls:**
*   **Poor SfM:** Optimization fails if SfM produces poor depth estimates or if depth measurements are inaccurate [S1].
*   **Gaussian Assumption:** The method assumes a Gaussian distribution models keypoint location uncertainty, which may not hold for depth derived from non-SfM sources [S1].
*   **Domain Gap:** Methods pre-trained on DTU (like pixelNeRF-DTU) struggle on NeRF Real data due to domain differences, but DS-NeRF does not suffer from this as it requires no pre-training [S1].
*   **MVSNeRF Compatibility:** Adding depth supervision did not improve MVSNeRF because it already has an existing geometry prior handled by its PSV-initialization [S1].

## Operating characteristics
*   **Time Complexity:** DS-NeRF reaches a target test PSNR in 2-3x fewer training iterations than standard NeRF [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On a single RTX A5000 GPU, a DS-NeRF training iteration takes ~362.4 ms, compared to ~359.8 ms for standard NeRF [S1]. In a 5-view setup, DS-NeRF achieves NeRF's peak test PSNR approximately 13 hours faster [S1].
*   **Scaling:** Performance gains are magnified at lower view counts [S1]. For example, in a 5-view setup, DS-NeRF achieves NeRF's peak test PSNR much faster, and with 2 views, NeRF fails to match DS-NeRF's performance [S1].
*   **Strengths:** Accelerates training, improves quality with fewer views, compatible with other NeRF methods, does not require domain-specific pre-training [S1].
*   **Limitations:** Relies on the quality of SfM keypoints; poor SfM leads to failure; assumes Gaussian uncertainty [S1].
*   **When to use:** When training data is sparse (few views) or when faster training is desired without sacrificing quality [S1].
*   **When not to use:** When SfM fails to produce reliable keypoints or when using methods that already have strong geometry priors (e.g., MVSNeRF) [S1].

## Background
*   **Origin:** DS-NeRF was introduced in the paper "Depth-supervised NeRF: Fewer Views and Faster Training for Free" [S1].
*   **History:** It leverages the "free" depth supervision provided by sparse 3D point clouds computed during standard SfM pre-processing steps (e.g., COLMAP) [S1].
*   **Usage:** Applied to LLFF, NeRF Real, Redwood-3dscan, and DTU datasets [S1]. It is compatible with depth from scanned depth sensors and RGB-D reconstruction outputs [S1].

## Sources
[S1] Depth-supervised NeRF: Fewer Views and Faster Training for Free, https://doi.org/10.1109/cvpr52688.2022.01254, fetched 2026-10-26
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26
