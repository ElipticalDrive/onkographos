# Regularization Techniques
## Overview
Regularization techniques in NeRF training are strategies applied to the optimization objective or the model parameters to mitigate artifacts such as floaters, blur, and overfitting, particularly in sparse-view or ill-posed reconstruction scenarios [S1, S2, S3, S6]. They set themselves apart from general hyperparameters (like learning rate or batch size) by directly modifying the geometry, density distribution, or view-dependency of the radiance field to enforce physical plausibility or structural consistency [S1, S4, S5].
## Shared principles
The family shares the goal of constraining the solution space of the neural radiance field to prevent degenerate solutions (e.g., empty space filled with noise, or "floaters") that fit the training images but do not represent the true 3D scene [S1, S6]. Common mechanisms include:
1. **Density Constraints:** Penalizing non-zero density in regions where no geometry should exist (e.g., near the camera or in the background) to suppress floaters [S2, S6].
2. **Consistency Losses:** Enforcing agreement between different representations of the scene (e.g., coarse vs. fine NeRFs, or predicted vs. geometry normals) to stabilize optimization [S3, S4].
3. **Frequency/Complexity Control:** Restricting the model's capacity to learn high-frequency details too early or in noisy conditions, often via curriculum learning or simplified architectures [S1, S5].
4. **Geometric Priors:** Using surface normals or edge-aware constraints to guide the shape of the reconstructed geometry [S1, S4].
## Subtopics

### [Spherical Harmonics Regularization](8693-spherical-harmonics-regularization.md)
[Spherical Harmonics Regularization](8693-spherical-harmonics-regularization.md) restricts the maximum degree of Spherical Harmonics (SH) during early optimization to prevent high-frequency overfitting and improve coarse geometry reconstruction. It stabilizes optimization in harsh input conditions by allowing the model to learn diffuse reflectance (low-frequency SH) before view-direction-sensitive components (high-frequency SH). It sets itself apart by applying frequency regularization directly to SH coefficients in a voxel grid representation, rather than to positional encodings or Fourier coefficients.

### [Bilateral Edge-Aware Constraint](16372-bilateral-edge-aware-constraint.md)
[Bilateral Edge-Aware Constraint](16372-bilateral-edge-aware-constraint.md) is a geometric regularization technique for NeRF-based volume rendering that suppresses high-frequency noise while preserving sharp geometric discontinuities such as building roofs and shadow boundaries. Unlike standard smoothness constraints that blur distinct edges, this method dynamically weights geometric penalties using photometric differences between neighboring rays, ensuring that significant color changes are not penalized as geometric errors. It addresses the lack of fixed neighborhood connectivity in implicit radiance fields by sampling neighbors via image-plane offsets.

### Simple-RF Loss Function
- **Definition:** The total loss function used to train the Simple-RF models, combining multiple regularization terms to mitigate floaters and improve geometry in sparse input scenarios [S1].
- **Step:** During the first 10k iterations of training, only impose $L_m$, $L_a$, and $L_{sd}$. Do not impose $L_{aug}$ and $L_{cfc}$ [S1]. After 10k iterations, impose all loss terms including $L_{aug}$ and $L_{cfc}$ [S1].
- **Formula:** $L = \lambda_m L_m + \lambda_a L_a + \lambda_{sd} L_{sd} + \lambda_{aug} L_{aug} + \lambda_{cfc} L_{cfc} + \lambda_{mc} L_{mc}$ [S1].
- **Parameter:** $\lambda_{sd}$: Weight for the smoothness/depth loss. Value: 0.1 [S1]. $\lambda_{aug}$: Weight for the augmentation loss. Value: 0.1 [S1]. $\lambda_{cfc}$: Weight for the consistency/contrastive loss (main NeRF only). Value: 0.1 [S1].
- **Pitfall:** $L_{cfc}$ and $L_{mc}$ are imposed only for the main NeRF and augmented TensoRF models respectively, not for all variants simultaneously in the same way [S1].

### Density Loss with Black and White Prior
- **Definition:** A regularization technique that minimizes the density of early samples along a ray to suppress background misplaces and floaters, adapted with a black/white prior for uniform backgrounds [S2].
- **Step:** When a pixel value is either black or white, minimize the density of the first $M+10$ samples along the ray instead of the first $M$ samples [S2].
- **Number:** On the DTU dataset (3/6/9 views), RING-NeRF with density loss and continuous coarse-to-fine achieves 16.18/20.47/23.19 PSNR, 0.713/0.808/0.847 SSIM, 0.200/0.127/0.085 LPIPS [S2].
- **Comparison:** Using only the density loss yields correct metrics but produces a slightly less precise 3D reconstruction and lower stability compared to using continuous coarse-to-fine alone. The combination of both provides the best trade-off between 3D reconstruction and novel view synthesis quality [S2].

### Hierarchical Sampling Consistency
- **Definition:** A regularization technique to prevent blur in rendered images caused by inconsistencies between coarse and fine NeRF depth estimates in few-shot settings. It drives the two NeRFs to be consistent in their solutions [S3].
- **Formula:** $L_{cfc} = \mathbb{1}_{\{m_{cfc}=1\}} \odot \|z_c - \tau(z_f)\|^2 + \mathbb{1}_{\{m_{cfc}=-1\}} \odot \|\tau(z_c) - z_f\|^2$, where $z_c$ and $z_f$ are depths estimated by coarse and fine NeRFs, $\tau$ is a transformation function, and $m_{cfc}$ is a mask determined by reliability estimation [S3].
- **Pitfall:** Without this loss, dense sampling may not be employed around the region where the fine NeRF predicts the object surface, leading to blur in rendered images [S3].
- **Number:** Disabling $L_{cfc}$ leads to a drop in performance and increases training time by almost 2× due to the inclusion of augmentations for the fine NeRF [S3].

### Asymmetric Predicted Normal Loss
- **Definition:** Apply an asymmetric predicted normal loss to tie predicted normals $\tilde{n}$ to geometry normals $n$ and rendering weights $w$, allowing separate multipliers for gradients flowing from geometry normals and weights to predicted normals, and vice versa [S4].
- **Formula:** Compute asymmetric loss $L_{pred} = \lambda_n L_p(w, n, \text{sg}(\tilde{n})) + \lambda_{\tilde{n}} L_p(\text{sg}(w), \text{sg}(n), \tilde{n})$, where $L_p(w, n, \tilde{n}) = \sum w^{(i)} \|n^{(i)} - \tilde{n}^{(i)}\|^2$ and $\text{sg}(\cdot)$ is stop-gradient [S4].
- **Parameter:** $\lambda_n$: Multiplier for the term tying predicted normals to geometry normals. Typical value: $10^{-3}$ [S4].
- **Pitfall:** Using standard symmetric predicted normal loss tends to oversmooth or underconstrain geometry, preventing normal vectors from converging to the correct solution [S4].

### MixNeRF Regularization
- **Definition:** A regularization method that models input rays with mixture density models to achieve promising results with sparse inputs [S5].
- **Step:** Minimize the negative log-likelihood (NLL) to learn the density distribution effectively with sparse inputs [S5].
- **Formula:** Mixture coefficient $\pi_i = w_i / \sum_{m=1}^{M} w_m = T_i(1-\exp(-\sigma_i \delta_i)) / \sum_{m=1}^{M} T_m(1-\exp(-\sigma_m \delta_m))$, where $w_i$ is alpha blending weight, $\delta_i$ is sample interval, $T_i$ is transmittance, $\sigma_i$ is volume density, and $M$ is number of samples [S5].

### Occlusion Regularization
- **Definition:** A regularization term to prevent 'floater' or 'walls' artifacts in sparse-view training by constraining near-camera density to zero. It operates effectively where geometry regularization cannot due to missing correspondences [S6].
- **Formula:** Total loss: $L(\theta, t) = L_{color}(\theta) + L_{geo}(\theta, t) + \lambda_{occ} L_{occ}(\theta)$, where $\lambda_{occ}$ is a pre-defined weighting factor [S6].
- **Parameter:** Occlusion regularization weight ($\lambda_{occ}$): 0.01 across all experiments [S6].

## Background
Regularization in NeRFs emerged as a response to the inherent ill-posedness of the inverse rendering problem, particularly when training data is sparse or noisy [G]. Early NeRF implementations often produced "floaters" (spurious geometry in empty space) or blurry reconstructions [G]. Simple-RF [S1] and SimpleNeRF [S3] introduced simplified augmentation strategies and consistency losses to address these issues in sparse settings. RING-NeRF [S2] highlighted the importance of density losses combined with coarse-to-fine sampling for stability. FreeNeRF [S6] formalized occlusion regularization to suppress near-camera artifacts. Ref-NeRF [S4] and subsequent works like NeRF-Casting introduced normal-based regularization to improve surface detail and specular consistency. MixNeRF [S5] utilized mixture density models to handle ambiguity in sparse inputs.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S2] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
[S3] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S4] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S5] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
[S6] SGCNeRF: Few-Shot Neural Rendering via Sparse Geometric Consistency Guidance, https://doi.org/10.1109/tcsvt.2025.3610950, fetched 2026-10-30
