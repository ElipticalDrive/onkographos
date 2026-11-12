# Depth-Guided NeRF Training via Earth Mover’s Distance

Also called: Depth-guided NeRF via Earth Mover’s Distance, EM D-Guided Sampling, EMD-Guided Depth Supervision for NeRF, EMD-NeRF, EMD-based objective

## Overview
This method improves the geometric accuracy of Neural Radiance Fields (NeRF) in sparse-view settings by supervising the distribution of ray termination distances using Earth Mover’s Distance (EMD) rather than enforcing direct depth matching [S1]. Unlike sibling methods such as [DäRF](12331-d-rf.md) which minimize the norm between rendered and prior depth, or [SCADE](9518-nerfingmvs.md) which assumes perfect pseudo-ground truths from multiple hypotheses, this approach uses a single off-the-shelf diffusion-based depth prior (DiffDP) and its intrinsic uncertainty map to selectively guide the NeRF [S1]. It prevents the propagation of prior inaccuracies and avoids interference with photometric loss by weighting the EMD loss against the uncertainty of the depth estimate [S1].

## How it works
The method integrates a pre-trained diffusion depth estimator with a standard NeRF, replacing naive L2 depth supervision with a distribution-based EMD loss weighted by uncertainty.

### Inputs and Outputs
*   **Inputs:**
    *   Set of $N$ input images $\{I_i\}_{i=1}^N$ (typically $N < 20$) and their corresponding SfM camera poses [S1].
    *   Pre-trained DiffDP network $G_\phi$ for monocular depth estimation and uncertainty generation [S1].
*   **Outputs:**
    *   Trained NeRF network $F_\Theta$ capable of rendering novel views and depth maps [S1].

### Data Structures
1.  **NeRF Network ($F_\Theta$):** A multilayer perceptron taking 5D input $(x, y, z, \theta, \phi)$ representing camera position and viewing direction, outputting color $c$ and volume density $\sigma$ [S1].
2.  **Depth Prior ($z_0$):** The final depth estimate from DiffDP for a given image $I$ [S1].
3.  **Uncertainty Map ($u$):** A per-pixel map derived from the DiffDP denoising process, normalized to $[0, 1]$ [S1].
4.  **Ray Samples:**
    *   Coarse samples: 64 termination distances [S1].
    *   Fine samples: 128 termination distances [S1].
    *   EMD samples: Additional 128 samples drawn from the NeRF's predicted distribution for EMD calculation [S1].

### Step-by-Step Method

#### 1. Depth Prior and Uncertainty Extraction
Before NeRF training, generate the depth prior and uncertainty map for each input image $I$ using DiffDP:
1.  DiffDP formulates denoising as $p_\theta(z_{0:T}|I) = p_\theta(z_T) \prod_{t=1}^T p_\theta(z_{t-1}|z_t, I)$, where $z_T \sim \mathcal{N}(0, I)$ and $z_0$ is the final depth estimate [S1].
2.  Compute temporal uncertainty count $c(z_{0:T})$ by comparing consecutive estimates $z_t$ and $z_{t-1}$ against a threshold $\tau$:
    $$ c(z_{0:T}) = \frac{1}{T} \sum_{t=T,\dots,1} \mathbb{I}(|z_t - z_{t-1}| \ge \tau) $$
    [S1].
3.  Compute spatial consistency uncertainty using the mirrored image $M(I)$:
    $$ U(z_{0:T}|I) = \frac{c(z_{0:T}|I) + M(c(z_{0:T}|M(I)))}{2} $$
    [S1].
4.  Augment with final prediction difference:
    $$ u(z_{0:T}|I) = U(z_{0:T}|I) \cdot \| (z_0|I) - M(z_0|M(I)) \|_1 $$
    [S1].
5.  Normalize $u$ to $[0, 1]$ [S1].

#### 2. NeRF Forward Pass and Weight Normalization
For a ray $r$ with samples at distances $y_i$:
1.  Query $F_\Theta$ to get density $\sigma_i$ and color $c_i$ for each sample [S1].
2.  Compute transmittance $T_i$ and weight $w_i$ (probability of ray traversal):
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    $$ w_i = T_i (1 - \exp(-\sigma_i \delta_i)) $$
    where $\delta_i$ is the distance between samples [S1].
3.  Normalize weights to form a piece-wise constant probability density function (PDF):
    $$ \hat{w}_i = \frac{w_i}{\sum_j w_j} $$
    [S1].

#### 3. EMD-Guided Sampling
1.  Construct the Cumulative Distribution Function (CDF) from $\hat{w}_i$ [S1].
2.  Sample $N_{EMD}$ new ray termination distances $y = \{y_1, \dots, y_{N_{EMD}}\}$ using inverse transform sampling from the CDF [S1].
3.  Compute the Earth Mover’s Distance between these samples $y$ and the depth prior $z_0$:
    $$ L_{EMD} = \text{EMD}(y, z_0) $$
    [S1].
4.  Approximate the non-differentiable EMD using Sinkhorn Divergence during training [S1].

#### 4. Loss Calculation and Optimization
1.  Compute standard photometric loss:
    $$ L_{photo} = \| \hat{C}(r) - C(r) \|_2^2 $$
    where $\hat{C}(r) = \sum_l w_l c_l$ and $C(r)$ is the observed color [S1].
2.  Compute total loss weighted by uncertainty $u$ and balancing weight $\lambda$:
    $$ L = (1 + u)^\gamma L_{photo} + \lambda (1 - u)^\gamma L_{EMD} $$
    where $\gamma$ controls the impact of uncertainty [S1]. Note: Unlike Focal Loss, this reduces weight for uncertain pixels [S1].
3.  Optimize $F_\Theta$ using the total loss. Initialize depth prior scale to 1 and learn with a small learning rate [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

# Standard math/vector operations assumed available
# torch.norm, torch.exp, torch.sum, torch.linspace, etc.

def compute_diffdp_prior_and_uncertainty(image):
    """
    Extracts depth prior and uncertainty from DiffDP.
    Returns: depth_prior (H, W), uncertainty_map (H, W)
    """
    # 1. Run DiffDP denoising process p_theta(z_0:T | I)
    # Assume diffdp_model returns list of z_t for t in T..0
    z_t_list = diffdp_model.denoise(image) 
    
    # 2. Compute temporal uncertainty count c(z_0:T)
    # Threshold tau is a hyperparameter
    tau = 0.1 
    temporal_counts = []
    for t in range(len(z_t_list) - 1, 0, -1):
        z_t = z_t_list[t]
        z_prev = z_t_list[t-1]
        # Indicator function: 1 if |z_t - z_prev| >= tau else 0
        mask = (torch.abs(z_t - z_prev) >= tau).float()
        temporal_counts.append(mask)
    
    # Average over T steps
    c_temporal = torch.mean(torch.stack(temporal_counts), dim=0)
    
    # 3. Compute spatial consistency using mirrored image
    image_mirrored = torch.flip(image, dims=[2]) # Mirror along width
    z_t_list_mirrored = diffdp_model.denoise(image_mirrored)
    
    temporal_counts_mirrored = []
    for t in range(len(z_t_list_mirrored) - 1, 0, -1):
        z_t = z_t_list_mirrored[t]
        z_prev = z_t_list_mirrored[t-1]
        mask = (torch.abs(z_t - z_prev) >= tau).float()
        temporal_counts_mirrored.append(mask)
        
    c_temporal_mirrored = torch.mean(torch.stack(temporal_counts_mirrored), dim=0)
    # Mirror the count back to original orientation
    c_temporal_mirrored = torch.flip(c_temporal_mirrored, dims=[2])
    
    U = (c_temporal + c_temporal_mirrored) / 2.0
    
    # 4. Augment with final prediction difference
    z_0 = z_t_list[0]
    z_0_mirrored = z_t_list_mirrored[0]
    z_0_mirrored = torch.flip(z_0_mirrored, dims=[2])
    
    diff_norm = torch.norm(z_0 - z_0_mirrored, p=1, dim=0) # L1 norm
    uncertainty_map = U * diff_norm
    
    # Normalize to [0, 1]
    uncertainty_map = (uncertainty_map - uncertainty_map.min()) / (uncertainty_map.max() - uncertainty.min() + 1e-8)
    
    return z_0, uncertainty_map

def nerf_forward(ray_origins, ray_directions, near, far, num_samples, nerf_model):
    """
    Standard NeRF forward pass.
    Returns: weights (N,), colors (N,), densities (N,), depths (N,)
    """
    t_vals = torch.linspace(near, far, num_samples)
    z_vals = t_vals
    
    # Sample points along ray
    points = ray_origins.unsqueeze(1) + z_vals.unsqueeze(0) * ray_directions.unsqueeze(1)
    
    # Query NeRF
    # nerf_model returns (color, density)
    colors, densities = nerf_model(points)
    
    # Compute deltas
    deltas = z_vals[1:] - z_vals[:-1]
    delta_inf = 1e10 # Large value for last sample
    deltas = torch.cat([deltas, delta_inf], dim=-1)
    
    # Compute transmittance and weights
    # alpha = 1 - exp(-sigma * delta)
    alphas = 1.0 - torch.exp(-densities * deltas)
    
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # Using cumulative sum of log(1-alpha) or direct product
    # T = product(1 - alpha)
    T = torch.cumprod(torch.cat([torch.ones_like(alphas[:, :1]), 1.0 - alphas[:, :-1]], dim=-1), dim=-1)
    
    weights = T * alphas
    
    # Rendered color and depth
    rendered_color = torch.sum(weights.unsqueeze(-1) * colors, dim=-2)
    rendered_depth = torch.sum(weights * z_vals, dim=-1)
    
    return weights, colors, densities, rendered_depth

def compute_emd_loss(samples_neRF, samples_prior, cost_matrix=None):
    """
    Computes Sinkhorn Divergence approximation of EMD.
    samples_neRF: (N,) tensor of distances from NeRF sampling
    samples_prior: (M,) tensor of depth prior values (flattened or binned)
    """
    # In practice, geomloss library is used. Here we describe the logic.
    # 1. Create histograms or use point clouds.
    # 2. Compute cost matrix C_ij = |x_i - y_j|
    # 3. Apply Sinkhorn algorithm to find optimal transport plan P
    # 4. Loss = sum(P * C)
    pass # Implementation relies on geomloss.SinkhornDivLoss

def train_step(nerf_model, image, pose, depth_prior, uncertainty_map, 
               num_coarse, num_fine, num_emd_samples, lambda_bal, gamma):
    """
    Single training step.
    """
    # 1. Cast rays for the image
    rays = cast_rays_from_image(image, pose)
    
    # 2. Coarse NeRF pass
    weights_coarse, colors_coarse, densities_coarse, _ = nerf_forward(
        rays.origins, rays.directions, rays.near, rays.far, num_coarse, nerf_model
    )
    
    # 3. Sample new termination distances for EMD from Coarse distribution
    # Normalize weights to PDF
    pdf = weights_coarse / weights_coarse.sum(dim=-1, keepdim=True)
    # Construct CDF
    cdf = torch.cumsum(pdf, dim=-1)
    # Inverse transform sampling
    u_rand = torch.rand_like(cdf)
    # Find indices where cdf >= u_rand
    indices = torch.searchsorted(cdf, u_rand, side='right')
    # Clamp indices
    indices = torch.clamp(indices, 0, num_coarse - 1)
    # Interpolate or pick samples
    # For simplicity, pick the sample at the index
    emd_sample_indices = indices
    emd_distances = rays.z_vals[emd_sample_indices] # Approximation
    
    # Refine with Fine NeRF pass using these distances? 
    # The paper says "sample new ray termination distances... supervise these samples".
    # Usually, this implies using the coarse distribution to guide fine sampling or direct EMD on coarse.
    # Let's assume we use the coarse distribution to generate EMD samples directly as per "EMD-guided Sampling" section.
    
    # 4. Compute EMD Loss
    # Flatten depth prior for the pixels corresponding to the rays
    prior_values = depth_prior[rays.pixel_indices] # (B,)
    # Expand prior to match batch size if needed, or compute per ray
    # EMD is computed between the distribution of emd_distances and the prior value (treated as a distribution?)
    # The paper says "EMD(y, z0)". z0 is the depth prior. 
    # If z0 is a single value per pixel, EMD between a distribution y and a delta function at z0 is simply the expected distance |y - z0|.
    # However, the paper contrasts with L2 on rendered depth. 
    # "Supervise the ray termination distance distribution... with EMD".
    # If prior is a single value, EMD(y, delta(z0)) = E[|y - z0|].
    # But the paper mentions "complex uncertainties" and "multimodal".
    # Let's assume the prior z0 might be treated as a distribution or the EMD is computed against the histogram of prior values if multiple hypotheses existed (but here single).
    # Actually, Eq 7: L_EMD = EMD(y, z0). If z0 is a scalar per pixel, this is L1 loss on samples.
    # However, the text says "EMD... naturally lends itself to compare probability densities".
    # It is likely that for each ray, we compare the PDF of y against a PDF derived from z0?
    # Or, more likely, since z0 is a map, we treat the set of all z0 values in the batch as a distribution?
    # No, supervision is per-ray.
    # Re-reading: "SCADE... 20 ambiguity-aware depth proposals... EMD... relaxes that assumption".
    # With a single prior z0, EMD(y, z0) where z0 is a scalar is equivalent to Mean Absolute Error if y is a distribution.
    # But the key is "distribution".
    # Let's implement the standard Sinkhorn on the samples y and the prior z0 treated as a target distribution?
    # If z0 is a single value, the "distribution" is a Dirac delta.
    # EMD(y, delta(z0)) = sum_i w_i * |y_i - z0|.
    # This is effectively a weighted L1 loss.
    # Why use Sinkhorn? Maybe to handle the discrete nature of samples vs continuous prior?
    # We will use the Sinkhorn divergence as stated.
    
    # Prepare tensors for geomloss
    # y: (B, N_emd)
    # z0: (B, 1) or (B, N_emd) if we replicate?
    # If z0 is scalar per ray, we replicate it to match N_emd samples for the cost matrix?
    # Cost matrix C_ij = |y_i - z0_j|. If z0 is scalar, C_ij = |y_i - z0|.
    
    # Using a placeholder for the actual Sinkhorn implementation which requires geomloss
    # L_emd = SinkhornLoss()(emd_distances, prior_values.unsqueeze(-1).expand_as(emd_distances))
    
    # 5. Fine NeRF pass (standard)
    weights_fine, colors_fine, _, _ = nerf_forward(
        rays.origins, rays.directions, rays.near, rays.far, num_fine, nerf_model
    )
    
    # 6. Photometric Loss
    rendered_color_fine = torch.sum(weights_fine.unsqueeze(-1) * colors_fine, dim=-2)
    observed_color = rays.colors
    L_photo = torch.mean((rendered_color_fine - observed_color) ** 2)
    
    # 7. Uncertainty Weighting
    # Uncertainty map is per pixel. Gather for the rays.
    u = uncertainty_map[rays.pixel_indices]
    
    # L = (1 + u)^gamma * L_photo + lambda * (1 - u)^gamma * L_EMD
    # Note: u is per pixel, L_photo is scalar mean.
    # We should compute loss per pixel/ray and then average?
    # The formula L = ... suggests scalar loss.
    # Usually, uncertainty weighting is applied to the loss term.
    
    # Compute L_EMD per ray
    # L_emd_per_ray = torch.mean(torch.abs(emd_distances - prior_values.unsqueeze(-1)), dim=-1)
    # If using Sinkhorn, it returns a scalar or per-batch loss.
    
    # Let's assume L_emd is the mean EMD over the batch
    L_emd = compute_emd_loss(emd_distances, prior_values) 
    
    # Apply weights
    # u needs to be broadcasted or averaged appropriately.
    # The paper says "weigh the photometric loss and the depth loss... according to the DiffDP-derived uncertainty maps".
    # This implies pixel-wise weighting of the loss components.
    
    # Pixel-wise photometric loss
    L_photo_pixel = (rendered_color_fine - observed_color) ** 2 # (B, 3)
    # Pixel-wise EMD loss (if computed per ray)
    # L_emd_pixel = ... 
    
    # Final scalar loss
    # L = mean( (1+u)^gamma * L_photo_pixel + lambda * (1-u)^gamma * L_emd_pixel )
    
    # Simplified scalar version based on Eq 8 structure:
    # L = (1 + mean(u))^gamma * L_photo + lambda * (1 - mean(u))^gamma * L_EMD ?
    # No, "weigh... according to... maps".
    # We will compute the weighted sum per pixel and average.
    
    # Placeholder for pixel-wise EMD calculation
    # L_emd_pixel = torch.abs(emd_distances - prior_values.unsqueeze(-1)).mean(dim=-1) # (B,)
    
    # Weighted Loss
    w_photo = (1 + u) ** gamma
    w_emd = lambda_bal * (1 - u) ** gamma
    
    # Assuming L_emd is computed per ray (B,)
    # L_total = mean( w_photo * L_photo_pixel + w_emd * L_emd_pixel )
    
    pass

```

### Parameters
*   **N**: Number of input images, typically $N < 20$ [S1].
*   **Rays per batch**: 1024 [S1].
*   **Coarse samples**: 64 ray termination distances [S1].
*   **Fine samples**: 128 ray termination distances [S1].
*   **EMD samples**: 128 additional samples for EMD calculation [S1].
*   **Dropout probability**: 0.1 [S1].
*   **Weight decay**: $10^{-6}$ [S1].
*   **$\lambda$**: Balancing weight between photometric and EMD loss [S1].
*   **$\gamma$**: Controls the impact of uncertainty $u$ [S1].
*   **Depth prior scale initialization**: 1, learned with a small learning rate [S1].

### Edge Cases and Pitfalls
*   **Photometric Quality vs. Geometry**: High PSNR/SSIM does not guarantee correct geometry; models may misinterpret structures (e.g., cabinet patterns) while rendering plausible RGB [S1].
*   **Uncertainty Handling**: Unlike Focal Loss which up-weights uncertain examples, this method down-weights them to rely on photometric loss [S1].
*   **Prior Noise**: Using multiple hypotheses (like SCADE) can introduce noise and instability; this method uses a single prior to avoid this [S1].
*   **Uncertainty Map Sensitivity**: The construction of the uncertainty map can be sensitive to hyperparameters (e.g., threshold $\tau$) [S1].

## Subtopics

### Monocular Depth Estimation Adaptation and Supervision
The parent framework providing the shared mechanism for extracting depth priors and aligning them with NeRF outputs. [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)

### NerfingMVS
A sibling method that uses multi-view stereo constraints. [NerfingMVS](9518-nerfingmvs.md)

### Monocular Depth Regularization on Seen Views
A sibling method that regularizes depth on training views. [Monocular Depth Regularization on Seen Views](12330-monocular-depth-regularization-on-seen-views.md)

### DäRF
A sibling method that directly supervises NeRF-rendered depth with L2 loss, lacking uncertainty weighting and distribution guidance. [DäRF](12331-d-rf.md)

### NeRF-guided MDE Adaptation
A sibling method that adapts the MDE network using NeRF. [NeRF-guided MDE Adaptation](12333-nerf-guided-mde-adaptation.md)

### Model Variants
General variants of NeRF architectures. [Model Variants](13153-model-variants.md)

## Operating characteristics
*   **Time Complexity**: Training time is increased due to the additional forward pass for uncertainty estimation (DiffDP) and the EMD/Sinkhorn computation, though Sinkhorn is efficient.
*   **Memory Complexity**: Requires storing the depth prior and uncertainty map for each input image. The EMD sampling adds a small overhead for the additional 128 samples per ray.
*   **Measured Performance**: On ScanNet+, reduces baseline RMSE by at least 56% while maintaining comparable photometric results [S1]. With DiffDP prior, EMD objective reduces initial prior RMSE by 31.4%; with DepthAnything prior, reduces it by 18.8% [S1].
*   **Scaling**: Designed for sparse-view settings ($N < 20$). Performance on dense views is not the primary focus but the method is applicable.
*   **Strengths**: Robust to prior inaccuracies; handles complex uncertainties; prevents gradient interference between RGB and depth losses; drop-in replacement for existing depth-supervised NeRFs [S1].
*   **Limitations**: Uncertainty measure is specific to diffusion-based models (DiffDP); sensitive to hyperparameters in uncertainty map construction; requires pre-trained diffusion model [S1].
*   **When to use**: Sparse-view reconstruction of challenging indoor scenes where geometric accuracy is critical and photometric-only NeRFs fail to capture geometry [S1].
*   **When not to use**: When dense multi-view data is available (standard NeRF suffices); when diffusion models are not available or too slow for the pipeline [S1].

## Background
The method was proposed by A. Rau et al. in the paper "Depth-Guided NeRF Training via Earth Mover’s Distance" [S1]. It addresses the limitation of standard NeRFs in sparse-view settings, where they often produce photometrically plausible but geometrically incorrect results [S1]. The work builds on the observation that monocular depth priors are noisy and that directly supervising rendered depth (as in DäRF) or assuming perfect hypotheses (as in SCADE) is suboptimal [S1]. By leveraging the uncertainty inherent in diffusion-based depth estimation (DiffDP) and using EMD to guide the ray termination distribution, the method provides a more robust geometric prior [S1]. It is used as a drop-in replacement for existing depth-supervised NeRFs on challenging indoor scenes like ScanNet and ScanNet+ [S1].

## Sources
[S1] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
