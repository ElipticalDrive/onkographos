# Regularization Techniques
## Overview
Regularization techniques in NeRF optimization are strategies designed to mitigate artifacts such as floaters, blur, and geometric instability, particularly in sparse-view or few-shot settings where the inverse problem is ill-posed [S1, S2, S3]. Unlike standard hyperparameters that control convergence speed, these techniques modify the loss landscape or the representation capacity during training to enforce smoother geometry, consistency between coarse and fine sampling, or physical plausibility [S1, S4, S5]. They set themselves apart from sibling techniques like initialization strategies or architecture modifications by actively constraining the optimization trajectory or the gradient flow rather than just setting the starting point or the model capacity [S1, S2, S3, S4, S5].

## Shared principles
The family of regularization techniques shares the goal of improving the generalization of the radiance field from limited input views. Common mechanisms include:
1.  **Loss Augmentation:** Adding auxiliary terms to the total loss function $\mathcal{L}$ beyond the standard photometric reconstruction loss $\mathcal{L}_{recon}$ [S1, S2, S3, S4, S5].
2.  **Curriculum Learning:** Gradually introducing complexity or regularization strength over the course of training (e.g., epochs or iterations) to stabilize early optimization [S1, S2].
3.  **Consistency Constraints:** Enforcing agreement between different representations or sampling strategies (e.g., coarse vs. fine NeRF, predicted vs. geometry normals) to reduce ambiguity [S3, S4].
4.  **Representation Restriction:** Limiting the effective capacity of the model (e.g., masking high-frequency SH coefficients) to prevent overfitting to noise in sparse data [S1, S2].

## Subtopics

### [Spherical Harmonics Regularization](8693-spherical-harmonics-regularization.md)
Spherical Harmonics (SH) Regularization restricts the maximum degree of Spherical Harmonics during early optimization to prevent high-frequency overfitting and improve coarse geometry reconstruction. It stabilizes optimization in harsh input conditions by scheduling the learning of diffuse reflectance (low-frequency SH) before view-direction-sensitive components (high-frequency SH). This method sets itself apart from sibling techniques by applying frequency regularization directly to the SH coefficients of a voxel grid representation, rather than to positional encodings or Fourier coefficients.

### Simple-RF Loss Function
**Definition:** The total loss function used to train the Simple-RF models, combining multiple regularization terms to mitigate floaters and improve geometry in sparse input scenarios [S1].
**Step:** During the first 10k iterations of training, only impose $L_m$, $L_a$, and $L_{sd}$. Do not impose $L_{aug}$ and $L_{cfc}$. After 10k iterations, impose all loss terms including $L_{aug}$ and $L_{cfc}$ [S1].
**Formula:** $L = \lambda_m L_m + \lambda_a L_a + \lambda_{sd} L_{sd} + \lambda_{aug} L_{aug} + \lambda_{cfc} L_{cfc} + \lambda_{mc} L_{mc}$ [S1].
**Parameter:** $\lambda_{sd}$: Weight for the smoothness/depth loss. Value: 0.1. $\lambda_{aug}$: Weight for the augmentation loss. Value: 0.1. $\lambda_{cfc}$: Weight for the consistency/contrastive loss (main NeRF only). Value: 0.1 [S1].
**Pitfall:** $L_{cfc}$ and $L_{mc}$ are imposed only for the main NeRF and augmented TensoRF models respectively, not for all variants simultaneously in the same way [S1].

### Density Loss with Black and White Prior
**Definition:** A regularization technique that minimizes the density of early samples along a ray to suppress background misplaces and floaters, adapted with a black/white prior for uniform backgrounds [S2].
**Step:** When a pixel value is either black or white, minimize the density of the first $M+10$ samples along the ray instead of the first $M$ samples [S2].
**Number:** On the DTU dataset (3/6/9 views), RING-NeRF with density loss and continuous coarse-to-fine achieves 16.18/20.47/23.19 PSNR, 0.713/0.808/0.847 SSIM, 0.200/0.127/0.085 LPIPS [S2].
**Comparison:** Using only the density loss yields correct metrics but produces a slightly less precise 3D reconstruction and lower stability compared to using continuous coarse-to-fine alone. The combination of both provides the best trade-off between 3D reconstruction and novel view synthesis quality [S2].

### Hierarchical Sampling Consistency
**Definition:** A regularization technique to prevent blur in rendered images caused by inconsistencies between coarse and fine NeRF depth estimates in few-shot settings. It drives the two NeRFs to be consistent in their solutions [S3].
**Formula:** $L_{cfc} = \mathbb{1}_{\{m_{cfc}=1\}} \odot \|z_c - \tau(z_f)\|^2 + \mathbb{1}_{\{m_{cfc}=-1\}} \odot \|\tau(z_c) - z_f\|^2$, where $z_c$ and $z_f$ are depths estimated by coarse and fine NeRFs, $\tau$ is a transformation function, and $m_{cfc}$ is a mask determined by reliability estimation [S3].
**Pitfall:** Without this loss, dense sampling may not be employed around the region where the fine NeRF predicts the object surface, leading to blur in rendered images [S3].
**Number:** Disabling $L_{cfc}$ leads to a drop in performance and increases training time by almost 2× due to the inclusion of augmentations for the fine NeRF [S3].

### Asymmetric Predicted Normal Loss
**Definition:** Apply an asymmetric predicted normal loss to tie predicted normals $\tilde{\mathbf{n}}$ to geometry normals $\mathbf{n}$ and rendering weights $w$, allowing separate multipliers for gradients flowing from geometry normals and weights to predicted normals, and vice versa [S4].
**Formula:** Compute asymmetric loss $L_{pred} = \lambda_n L_p(w, \mathbf{n}, \text{sg}(\tilde{\mathbf{n}})) + \lambda_{\tilde{n}} L_p(\text{sg}(w), \text{sg}(\mathbf{n}), \tilde{\mathbf{n}})$, where $L_p(w, \mathbf{n}, \tilde{\mathbf{n}}) = \sum w^{(i)} \|\mathbf{n}^{(i)} - \tilde{\mathbf{n}}^{(i)}\|^2$ and $\text{sg}(\cdot)$ is stop-gradient [S4].
**Parameter:** $\lambda_n$: Multiplier for the term tying predicted normals to geometry normals. Typical value: $10^{-3}$ [S4].
**Pitfall:** Using standard symmetric predicted normal loss tends to oversmooth or underconstrain geometry, preventing normal vectors from converging to the correct solution [S4].

### MixNeRF Regularization
**Definition:** A regularization method that models input rays with mixture density models to achieve promising results with sparse inputs [S5].
**Step:** Minimize the negative log-likelihood (NLL) to learn the density distribution effectively with sparse inputs [S5].
**Formula:** Mixture coefficient $\pi_i = w_i / \sum_{m=1}^{M} w_m = T_i(1-\exp(-\sigma_i \delta_i)) / \sum_{m=1}^{M} T_m(1-\exp(-\sigma_m \delta_m))$, where $w_i$ is alpha blending weight, $\delta_i$ is sample interval, $T_i$ is transmittance, $\sigma_i$ is volume density, and $M$ is number of samples [S5].

## Background
Regularization in NeRFs emerged as a response to the instability of training neural radiance fields with sparse input views, where standard volume rendering often produces "floaters" (spurious geometry) or blurry reconstructions. Simple-RF [S1] and SimpleNeRF [S3] introduced the concept of "simpler solutions" via augmentation and consistency losses to regularize the optimization. RING-NeRF [S2] incorporated density loss with specific priors for uniform backgrounds to handle sparse DTU data. NeRF-Casting [S4] addressed view-dependent appearance and normal consistency using asymmetric losses. FlipNeRF [S5] built upon MixNeRF's mixture density modeling to further regularize sparse inputs using flipped reflection rays. These methods collectively form a family of techniques that prioritize geometric and photometric consistency over raw fitting capacity in data-scarce regimes.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S2] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
[S3] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S4] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S5] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
