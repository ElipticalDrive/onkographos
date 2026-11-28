# Regularization Techniques

Also called: NeRF Regularization, Training Regularization

## Overview
Regularization techniques in NeRF training stabilize optimization, suppress artifacts (such as floaters and blur), and improve generalization in few-shot or sparse input scenarios. Unlike [HashNeRF Parameters](3068-hashnerf-parameters.md) which focus on architectural capacity, or [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md) which set initial states, regularization modifies the loss function or gradient updates during training to enforce physical priors or consistency.

## Shared principles
Regularization methods typically add auxiliary loss terms $\mathcal{L}_{reg}$ to the primary reconstruction loss $\mathcal{L}_{recon}$, resulting in a total loss $\mathcal{L} = \mathcal{L}_{recon} + \lambda_{reg} \mathcal{L}_{reg}$. These terms enforce constraints on:
1.  **Density/Opacity:** Preventing non-physical density values or suppressing background noise.
2.  **Consistency:** Enforcing agreement between multi-scale representations (coarse/fine) or geometric priors.
3.  **Frequency/Smoothness:** Restricting high-frequency updates to prevent overfitting to noise or sparse views.

## Subtopics

### [Spherical Harmonics Regularization](8693-spherical-harmonics-regularization.md)
See [Spherical Harmonics Regularization](8693-spherical-harmonics-regularization.md).

### Density Loss with Black and White Prior
- **Definition:** A regularization technique that minimizes the density of early samples along a ray to suppress background misplaces and floaters, adapted with a black/white prior for uniform backgrounds [S1].
- **Step:** When a pixel value is either black or white, minimize the density of the first $M+10$ samples along the ray instead of the first $M$ samples [S1].
- **Performance:** On the DTU dataset (3/6/9 views), RING-NeRF with density loss and continuous coarse-to-fine achieves 16.18/20.47/23.19 PSNR, 0.713/0.808/0.847 SSIM, 0.200/0.127/0.085 LPIPS [S1].
- **Comparison:** Using only the density loss yields correct metrics but produces a slightly less precise 3D reconstruction and lower stability compared to using continuous coarse-to-fine alone. The combination of both provides the best trade-off between 3D reconstruction and novel view synthesis quality [S1].

### Hierarchical Sampling Consistency
- **Definition:** A regularization technique to prevent blur in rendered images caused by inconsistencies between coarse and fine NeRF depth estimates in few-shot settings. It drives the two NeRFs to be consistent in their solutions [S2].
- **Formula:** The coarse-fine consistency loss is defined as:
  $$
  L_{cfc} = \mathbb{1}_{\{m_{cfc}=1\}} \odot \|z_c - \tau(z_f)\|^2 + \mathbb{1}_{\{m_{cfc}=-1\}} \odot \|\tau(z_c) - z_f\|^2
  $$
  where $z_c$ and $z_f$ are depths estimated by coarse and fine NeRFs, $\tau$ is a transformation function, and $m_{cfc}$ is a mask determined by reliability estimation [S2].
- **Pitfall:** Without this loss, dense sampling may not be employed around the region where the fine NeRF predicts the object surface, leading to blur in rendered images [S2].
- **Performance:** Disabling $L_{cfc}$ leads to a drop in performance and increases training time by almost 2× due to the inclusion of augmentations for the fine NeRF [S2].

### MixNeRF Regularization
- **Definition:** A regularization method that models input rays with mixture density models to achieve promising results with sparse inputs [S3].
- **Step:** Minimize the negative log-likelihood (NLL) to learn the density distribution effectively with sparse inputs [S3].
- **Formula:** The mixture coefficient $\pi_i$ is derived from the estimated volume density $\sigma_i$ as follows:
  $$
  \pi_i = \frac{w_i}{\sum_{m=1}^{M} w_m} = \frac{T_i(1-\exp(-\sigma_i \delta_i))}{\sum_{m=1}^{M} T_m(1-\exp(-\sigma_m \delta_m))}
  $$
  where $w_i$ is the alpha blending weight, $\delta_i$ is the sample interval, $T_i$ is the transmittance, $\sigma_i$ is the volume density, and $M$ is the number of samples [S3].

## Background
Regularization in NeRFs emerged as a response to the instability of training with sparse inputs and the tendency of volumetric rendering to produce "floaters" (spurious density in empty space). Early works like FreeNeRF introduced density loss to suppress background noise [S1]. Subsequent research focused on consistency between multi-scale networks (SimpleNeRF) [S2] and probabilistic modeling of ray densities (MixNeRF) [S3]. These techniques are critical for few-shot novel view synthesis and robust training in real-world scenarios.

## Sources
[S1] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
[S2] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S3] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
