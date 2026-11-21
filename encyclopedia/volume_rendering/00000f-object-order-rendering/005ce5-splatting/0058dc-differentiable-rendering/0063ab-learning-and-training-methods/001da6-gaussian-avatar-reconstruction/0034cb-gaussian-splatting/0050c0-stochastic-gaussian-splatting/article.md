# Stochastic Gaussian Splatting

Also called: Bayesian Gaussian Splatting, Gaussian Point Splatting, Stochastic 3DGS Rendering, StochasticSplats, VI-based Gaussian Splatting

## Overview
Stochastic Gaussian Splatting (SGS) refers to two distinct methodological extensions of 3D Gaussian Splatting that replace deterministic rasterization with probabilistic sampling. The first variant, often termed **Bayesian SGS** [S2], treats Gaussian parameters as random variables to enable uncertainty estimation alongside novel view synthesis. The second variant, **Gaussian Point Splatting** [S3, unofficial], eliminates the per-tile radix sort and tile-based rendering bottlenecks by sampling opaque points from Gaussians and compositing them via atomic operations, enabling the interactive rendering of hundreds of millions of Gaussians. Both methods rely on Variational Inference or stochastic transparency to approximate the volume rendering integral without explicit depth sorting.

## How it works
The implementation details diverge significantly between the two primary SGS variants. The following sections describe the Bayesian approach for uncertainty estimation [S2] and the Point Splatting approach for massive scene rendering [S3, unofficial].

### Bayesian SGS (Uncertainty Estimation)
This method extends the deterministic GS pipeline by learning a parametric distribution over Gaussian kernels using Variational Inference (VI).

**Inputs and Outputs**
*   **Input:** A dataset of posed images $D = \{(o_i, d_i, y_i)\}$, where $(o_i, d_i)$ are pixel coordinates and viewing directions, and $y_i$ is the observed color.
*   **Output:** A synthesized image $I(o, d)$ and an uncertainty map $\sigma(o, d)$ representing the standard deviation of pixel color.

**Data Structures**
The scene is modeled as a collection of $K$ Gaussian kernels $G_k$. Unlike deterministic GS, each kernel $G_k$ is defined by a posterior distribution $q_{\theta_k}(G_k)$ rather than fixed parameters. The learnable parameters $\theta$ include the means and covariances of these distributions:
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
    *   Center Learning Rate: Fixed at $10^{-2}$.
2.  **Phase 2 (Bayesian SGS):** At iteration 2500, fix the current GS parameters as the prior $\hat{G}$.
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
*   **Reconstruction Loss ($L_{rec}$):** $\ell_1$ norm between observed $y_i$ and rendered $I(o_i, d_i)$ [S2].
*   **KL Loss ($L_{KL}$):** Sum of KL divergences between prior (hat variables) and posterior (starred variables) for all parameters [S2]:
    $$ L_{KL} = \sum_{k=1}^{K} \left[ KL(\mathcal{N}_{\hat{\mu}_k, \hat{\Gamma}_k} || \mathcal{N}_{\mu^*_k, \Gamma^*_k}) + KL(\mathcal{N}_{\hat{\alpha}_k, \hat{\Pi}_k} || \mathcal{N}_{\alpha^*_k, \Pi^*_k}) + KL(\mathcal{N}_{\hat{c}_{klm}, \hat{\Xi}_{klm}} || \mathcal{N}_{c^*_{klm}, \Xi^*_{klm}}) \right] $$
    The closed-form KL divergence between two multivariate normals $\mathcal{N}_{\mu_0, \Sigma_0}$ and $\mathcal{N}_{\mu_1, \Sigma_1}$ is:
    $$ KL = -\frac{1}{2} + \frac{1}{2} \text{Tr}(\Sigma_1^{-1} \Sigma_0) + \frac{1}{2} (\mu_1 - \mu_0)^T \Sigma_1^{-1} (\mu_1 - \mu_0) + \frac{1}{2} \ln\left(\frac{\det \Sigma_1}{\det \Sigma_0}\right) $$
*   **AUSE Loss ($L_{AUSE}$):** Optimizes the Area Under the Sparsification Error [S2].

**Inference**
Uncertainty is estimated via Monte Carlo sampling. The posterior is sampled 8 times during both training and testing to compute the variance of pixel colors [S2].

### Gaussian Point Splatting (Massive Rendering)
This method removes sorting by sampling opaque points and using atomics [S3, unofficial].

**Inputs and Outputs**
*   **Input:** A list of $M$ 3D Gaussians (from a standard 3DGS model).
*   **Output:** A rendered image.

**Data Structures**
*   **Point Count List:** For each Gaussian $k$, compute the number of points $N_k$ to splat. This depends on the Gaussian's opacity and size in view, ranging from 1 to millions [S3, unofficial].
*   **Thread Workload Map:** A parallel mapping from GPU thread IDs to the list of Gaussian IDs to splat, ensuring perfect workload distribution [S3, unofficial].

**Algorithm Steps**
1.  **Sample Points:** For each Gaussian $k$ in view, sample $N_k$ pixel-sized opaque points independently and identically from the Gaussian distribution [S3, unofficial].
2.  **Opacity Correction:** Direct sampling leads to underestimation of true opacity due to collisions (multiple points hitting the same pixel). The effective opacity $\alpha_{eff}$ is derived such that the probability of splatting *no* points to a pixel is exactly $1 - \alpha_{true}$, where $\alpha_{true}$ is the true opacity of the Gaussian at that pixel [S3, unofficial].
3.  **Atomic Splatting:** Splat the sampled points using 64-bit atomic operations to update the color and opacity buffers [S3, unofficial].
4.  **Stochastic Transparency:** Apply stochastic transparency to the accumulated results [S3, unofficial].

**Parameters**
*   **Samples Per Pixel (SPP):** The number of points sampled per pixel. A typical value is 4, which produces a faithful approximation enabling real-time rendering [S3, unofficial].

**Pseudocode (Gaussian Point Splatting)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Input: Gaussians G, Camera C, Resolution (W, H), SPP=4
# Output: Image buffer Color[W, H, 3], Opacity buffer Alpha[W, H]

def gaussian_point_splatting(G, C, W, H, SPP):
    # Initialize buffers
    Color = zeros((H, W, 3))
    Alpha = zeros((H, W))
    
    # 1. Compute point counts per Gaussian
    # N_k depends on Gaussian k's opacity and size in view
    point_counts = []
    for k in G:
        # Estimate projected size and opacity
        projected_area = compute_projected_area(k, C)
        opacity = k.opacity
        # Heuristic for N_k based on area and SPP
        # [G] Implementation detail: N_k = ceil(SPP * projected_area / pixel_area)
        # Clamped to min 1
        n_k = max(1, int(SPP * projected_area / (W * H) * total_pixels_in_view))
        point_counts.append(n_k)
        
    # 2. Map threads to Gaussian IDs
    # Create a flat list of Gaussian IDs to process
    # Each ID appears N_k times
    gaussian_ids_to_splat = []
    for k, n_k in enumerate(point_counts):
        gaussian_ids_to_splat.extend([k] * n_k)
        
    # 3. Parallel Splatting (Simulated here sequentially for clarity)
    # In practice, this is done via GPU kernels with 64-bit atomics
    for g_id in gaussian_ids_to_splat:
        k = G[g_id]
        
        # Sample a point from the Gaussian
        # [G] Sample from N(mu_k, Sigma_k)
        point = sample_from_gaussian(k.mu, k.Sigma)
        
        # Project point to screen
        screen_x, screen_y = project_point(point, C)
        
        # Check bounds
        if not (0 <= screen_x < W and 0 <= screen_y < H):
            continue
            
        # Compute effective opacity for this single point
        # [S3, unofficial] alpha_eff ensures P(no points) = 1 - alpha_true
        # [G] Specific formula for alpha_eff depends on the collision model.
        # Typically: alpha_eff = 1 - (1 - alpha_true)^(1/N_k)
        # However, since we sample N_k points, and we want the result
        # to match the deterministic alpha blending, the opacity correction
        # is applied per point or per pixel accumulation.
        # The source states: "probability of splatting no points ... is exactly 1 - alpha_true"
        # If we splat N_k points, and each has opacity alpha_p,
        # P(no hit) = (1 - alpha_p)^N_k = 1 - alpha_true
        # => alpha_p = 1 - (1 - alpha_true)^(1/N_k)
        
        alpha_true = compute_gaussian_opacity_at_pixel(k, screen_x, screen_y)
        N_k = point_counts[g_id]
        alpha_p = 1.0 - (1.0 - alpha_true) ** (1.0 / N_k)
        
        # Splat using atomic operations
        # [G] AtomicAdd to Color and Alpha buffers
        atomic_add_alpha(screen_x, screen_y, alpha_p)
        atomic_add_color(screen_x, screen_y, alpha_p * k.color)
        
    # 4. Final Compositing (if not done atomically)
    # If atomics only added contributions, we need to resolve the order-independent transparency
    # [S3, unofficial] "applying stochastic transparency to the splatted points"
    # [G] Standard alpha blending loop over pixels if atomics were simple adds
    # Or if atomics handled the blending, the result is final.
    # Given "sort-free compositing", the atomics likely handle the blending
    # or the "stochastic transparency" implies a final pass.
    # However, standard 3DGS atomics usually just add to color/alpha.
    # To get correct alpha blending without sort, one might need a different approach.
    # [G] Note: The source says "eliminate the need for both sorting and tile-based rendering".
    # This implies the atomic accumulation *is* the compositing, or a very simple final pass.
    # Assuming standard atomic add for color/alpha is insufficient for correct alpha blending.
    # [G] Correction: Stochastic transparency often implies that the final pixel color
    # is the sum of contributions weighted by their probability, or a specific
    # order-independent transparency technique.
    # Given the lack of specific algorithm for the atomic blending in [S3, unofficial],
    # we assume the atomics update a buffer that represents the final color
    # after applying the stochastic transparency model.
    
    return Color
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
