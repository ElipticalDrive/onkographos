# Stochastic Gaussian Splatting

Also called: Bayesian Gaussian Splatting, Gaussian Point Splatting, Stochastic 3DGS Rendering, StochasticSplats, VI-based Gaussian Splatting

## Overview
Stochastic Gaussian Splatting (SGS) refers to two distinct methodological extensions of 3D Gaussian Splatting that replace deterministic rasterization with probabilistic sampling [S2][S3, unofficial]. The first variant, often termed **Bayesian SGS** [S2], treats Gaussian parameters as random variables to enable uncertainty estimation alongside novel view synthesis. The second variant, **Gaussian Point Splatting** [S3, unofficial], eliminates the per-tile radix sort and tile-based rendering bottlenecks by sampling opaque points from Gaussians and compositing them via atomic operations, enabling the interactive rendering of hundreds of millions of Gaussians. Both methods rely on Variational Inference or stochastic transparency to approximate the volume rendering integral without explicit depth sorting [S2][S3, unofficial].

## How it works
The implementation details diverge significantly between the two primary SGS variants [S2][S3, unofficial]. The following sections describe the Bayesian approach for uncertainty estimation [S2] and the Point Splatting approach for massive scene rendering [S3, unofficial].

### Bayesian SGS (Uncertainty Estimation)
This method extends the deterministic GS pipeline by learning a parametric distribution over Gaussian kernels using Variational Inference (VI) [S2].

**Inputs and Outputs**
*   **Input:** A dataset of posed images $D = \{(o_i, d_i, y_i)\}$, where $(o_i, d_i)$ are pixel coordinates and viewing directions, and $y_i$ is the observed color [S2].
*   **Output:** A synthesized image $I(o, d)$ and an uncertainty map $\sigma(o, d)$ representing the standard deviation of pixel color [S2].

**Data Structures**
The scene is modeled as a collection of $K$ Gaussian kernels $G_k$ [S2]. Unlike deterministic GS, each kernel $G_k$ is defined by a posterior distribution $q_{\theta_k}(G_k)$ rather than fixed parameters [S2]. The learnable parameters $\theta$ include the means and covariances of these distributions:
*   $\mu^*_k \in \mathbb{R}^3$: Mean of the Gaussian position.
*   $\Gamma^*_k \in \mathbb{R}^{3 \times 3}$: Covariance of the Gaussian position.
*   $\alpha^*_k \in \mathbb{R}$: Parameter for opacity distribution.
*   $\Pi^*_k \in \mathbb{R}$: Variance parameter for opacity.
*   $c^*_{klm}, \Xi^*_{klm}$: Mean and variance parameters for spherical harmonic coefficients.

**Prior Initialization**
Directly optimizing KL divergence with random initialization causes convergence issues by fixing Gaussians in random positions [S2]. SGS uses an Empirical Bayes approach:
1.  **Phase 1 (Standard GS):** Run standard 3DGS for 2500 iterations [S2].
    *   Spherical Harmonics degree: 1.
    *   Densification: Applied until iteration 1000.
    *   Center Learning Rate: Fixed at $10^{-2}$ [S2].
2.  **Phase 2 (Bayesian SGS):** At iteration 2500, fix the current GS parameters as the prior $\hat{G}$ [S2].
    *   Initialize prior covariance matrices $\hat{\Sigma}$ as $10^{-2} I$ (identity matrix) [S2].
    *   Posterior Learning Rate: $10^{-4}$ for all learnable parameters [S2].
    *   Training continues until iteration 10000 [S2].

**Rendering Formula**
The image is rendered using alpha blending (EWA Volume Splatting), treating parameters as samples from their distributions [S2]:
$$ I(o, d) = \sum_{k=1}^{K} \alpha_k e^{z_k(o,d)} c_k(d) \prod_{j=1}^{k-1} (1 - \alpha_j e^{z_j(o,d)}) $$
where $z_k(o, d)$ is the splatting coefficient derived from the projected 2D Gaussian [S2]. The elliptical 3D Gaussian kernel $\phi$ is defined as:
$$ \phi(x; \mu_k, \Sigma_k) = \exp\left(-\frac{1}{2} (x - \mu_k)^T \Sigma_k^{-1} (x - \mu_k)\right) $$
where $\mu_k \in \mathbb{R}^3$ is the mean and $\Sigma_k \in \mathbb{R}^{3 \times 3}$ is the covariance matrix [S2].

**Optimization Objective**
The method minimizes the KL divergence between the approximate posterior $q_\theta(G)$ and the true posterior $P(G|D)$ [S2]. Using Bayes' theorem and assuming independence among kernels, the loss function is:
$$ L = L_{rec} + \lambda_{SSIM} L_{SSIM} + \lambda_{KL} L_{KL} + \lambda_{AUSE} L_{AUSE} $$
*   **Reconstruction Loss ($L_{rec}$):** Typically the $\ell_1$ norm between observed $y_i$ and rendered $I(o_i, d_i)$ [S2], though the probabilistic formulation models the conditional probability $P(y_i | (o_i, d_i), G)$ as a normal distribution, leading to a log-likelihood term proportional to the squared error $(y_i - I(o_i, d_i))^2$ [S2].
*   **KL Loss ($L_{KL}$):** Sum of KL divergences between prior (hat variables) and posterior (starred variables) for all parameters [S2]:
    $$ L_{KL} = \sum_{k=1}^{K} \left[ KL(\mathcal{N}_{\hat{\mu}_k, \hat{\Gamma}_k} || \mathcal{N}_{\mu^*_k, \Gamma^*_k}) + KL(\mathcal{N}_{\hat{\alpha}_k, \hat{\Pi}_k} || \mathcal{N}_{\alpha^*_k, \Pi^*_k}) + KL(\mathcal{N}_{\hat{c}_{klm}, \hat{\Xi}_{klm}} || \mathcal{N}_{c^*_{klm}, \Xi^*_{klm}}) \right] $$
    The closed-form KL divergence between two multivariate normals $\mathcal{N}_{\mu_0, \Sigma_0}$ and $\mathcal{N}_{\mu_1, \Sigma_1}$ is:
    $$ KL = -\frac{1}{2} + \frac{1}{2} \text{Tr}(\Sigma_1^{-1} \Sigma_0) + \frac{1}{2} (\mu_1 - \mu_0)^T \Sigma_1^{-1} (\mu_1 - \mu_0) + \frac{1}{2} \ln\left(\frac{\det \Sigma_1}{\det \Sigma_0}\right) $$
*   **AUSE Loss ($L_{AUSE}$):** Optimizes the Area Under the Sparsification Error [S2].

**Inference**
Uncertainty is estimated via Monte Carlo sampling [S2]. The posterior is sampled 8 times during both training and testing to compute the variance of pixel colors [S2].

### Gaussian Point Splatting (Massive Rendering)
This method removes sorting by sampling opaque points and using atomics [S3, unofficial].

**Inputs and Outputs**
*   **Input:** A list of $M$ 3D Gaussians (from a standard 3DGS model).
*   **Output:** A rendered image.

**Data Structures**
*   **Point Count List:** For each Gaussian $k$, compute the number of points $N_k$ to splat [S3, unofficial]. This depends on the Gaussian's opacity and size in view, ranging from 1 to millions [S3, unofficial].
*   **Thread Workload Map:** A parallel mapping from GPU thread IDs to the list of Gaussian IDs to splat, ensuring perfect workload distribution [S3, unofficial].

**Algorithm Steps**
1.  **Sample Points:** For each Gaussian $k$ in view, sample $N_k$ pixel-sized opaque points independently and identically from the Gaussian distribution [S3, unofficial].
2.  **Opacity Correction:** Direct sampling leads to underestimation of true opacity due to collisions (multiple points hitting the same pixel) [S3, unofficial]. The effective opacity $\alpha_{eff}$ is derived such that the probability of splatting *no* points to a pixel is exactly $1 - \alpha_{true}$, where $\alpha_{true}$ is the true opacity of the Gaussian at that pixel [S3, unofficial].
3.  **Atomic Splatting:** Splat the sampled points using 64-bit atomic operations to update the color and opacity buffers [S3, unofficial].
4.  **Stochastic Transparency:** Apply stochastic transparency to the accumulated results [S3, unofficial].

**Parameters**
*   **Samples Per Pixel (SPP):** The number of points sampled per pixel [S3, unofficial]. A typical value is 4, which produces a faithful approximation enabling real-time rendering [S3, unofficial].

**Pseudocode (Gaussian Point Splatting)** [S3, unofficial]
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def gaussian_point_splatting(gaussians, camera, width, height, spp):
    # Initialize buffers
    color_buffer = [[0.0, 0.0, 0.0] for _ in range(height * width)]
    alpha_buffer = [0.0] * (height * width)
    
    # 1. Compute point counts per Gaussian
    point_counts = []
    for k in gaussians:
        # [G] Compute projected area and opacity at pixel center
        # N_k depends on Gaussian k's opacity and size in view
        # [S3, unofficial] Count ranges from 1 to millions
        projected_area = 1.0 # [G] Source does not specify exact projected area formula
        alpha_true = 0.5 # [G] Source does not specify exact opacity evaluation formula
        n_k = max(1, int(spp * projected_area))
        point_counts.append(n_k)
        
    # 2. Map threads to Gaussian IDs
    gaussian_ids_to_splat = []
    for k, n_k in enumerate(point_counts):
        gaussian_ids_to_splat.extend([k] * n_k)
        
    # 3. Parallel Splatting
    # In practice, this runs on GPU with 64-bit atomics
    for g_id in gaussian_ids_to_splat:
        k = gaussians[g_id]
        
        # Sample a point from the Gaussian distribution
        # [G] Standard multivariate normal sampling
        point = [k.mu[0] + random.gauss(0, 1), k.mu[1] + random.gauss(0, 1), k.mu[2] + random.gauss(0, 1)]
        
        # Project point to screen
        # [G] Standard perspective projection
        screen_x, screen_y = int(point[0] * width), int(point[1] * height)
        
        # Check bounds
        if not (0 <= screen_x < width and 0 <= screen_y < height):
            continue
            
        # Compute true opacity at pixel
        alpha_true = math.exp(-0.5 * ((screen_x - k.mu[0])**2 + (screen_y - k.mu[1])**2)) # Simplified
        N_k = point_counts[g_id]
        
        # Opacity Correction: alpha_p ensures P(no hit) = 1 - alpha_true
        alpha_p = 1.0 - (1.0 - alpha_true) ** (1.0 / N_k)
        
        # Splat using atomic operations
        # [S3, unofficial] 64-bit atomic add to color and alpha buffers
        idx = screen_y * width + screen_x
        atomic_add_alpha(alpha_buffer, idx, alpha_p)
        atomic_add_color(color_buffer, idx, alpha_p * k.color)
        
    return color_buffer, alpha_buffer

def atomic_add_alpha(buffer, idx, val):
    # Simulates 64-bit atomic add
    buffer[idx] += val

def atomic_add_color(buffer, idx, val):
    # Simulates 64-bit atomic add
    for i in range(3):
        buffer[idx][i] += val[i]
```

## Operating characteristics
*   **Bayesian SGS:**
    *   **Performance:** Real-time rendering ($\ge 30$ fps at 1080p) [S2].
    *   **Quality:** On the LLFF dataset (504x378), it achieves PSNR 24.20, SSIM 0.842, LPIPS 0.121 [S2].
    *   **Uncertainty:** Provides AUSE RMSE 0.0147 and AUSE MAE 0.0092 [S2].
    *   **Limitations:** Requires 8 Monte Carlo samples per pixel for uncertainty, increasing compute cost compared to deterministic GS [S2]. Direct optimization of reconstruction loss alone underestimates pixel variance [S2].
*   **Gaussian Point Splatting:**
    *   **Performance:** Renders scenes with 425M Gaussians interactively on an NVIDIA RTX 4070 Ti SUPER with 4 SPP [S3, unofficial].
    *   **Scaling:** Eliminates tile-based rendering and sorting bottlenecks, allowing linear scaling with the number of Gaussians [S3, unofficial].
    *   **Quality:** Faithful approximation of 3DGS; noise is unbiased and can be mitigated with temporal accumulation [S3, unofficial].
    *   **Limitations:** Introduces noise at low SPP (e.g., 4 SPP) [S3, unofficial].

## Background
*   **Bayesian SGS:** Introduced by Savant, Valsesia, and Magli in 2024 (arXiv:2403.18476) as the first framework for uncertainty estimation in Gaussian Splatting using Variational Inference [S2]. It addresses the lack of confidence metrics in GS compared to NeRF-based methods like S-NeRF [S2].
*   **Gaussian Point Splatting:** Introduced by Rijsdijk, Peters, Weinmann, and Marroquim in 2026 (ACM TOG) as a method for rendering massive 3DGS scenes [S3, unofficial]. It addresses the latency of per-tile radix sort in standard 3DGS rasterization [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Modeling Uncertainty for Gaussian Splatting, https://doi.org/10.1109/tnnls.2025.3553582, fetched 2026-10-26
[S3] Gaussian Point Splatting – a stochastic method for rendering 3DGS scenes, https://jorisar.nl/gaussian_point_splatting/, fetched 2026-10-26 (unofficial)
