# GaussianFlow

Also called: Dual-Domain Deformation Model (DDDM), A method using an explicit flow field for temporal regularization in dynamic scenes, Guo et al

## Overview
GaussianFlow is a point-based differentiable rendering approach for dynamic 3D scene reconstruction that models a 4D scene as a set of deformable 3D Gaussian particles advected by an explicit 3D velocity field [S1, unofficial][S3]. It solves the problem of efficient dynamic scene reconstruction by introducing a Dual-Domain Deformation Model (DDDM) that explicitly models attribute deformations (position, rotation, radiance) in both time and frequency domains, eliminating the need for implicit neural fields or per-frame optimization [S3]. Unlike [Dynamic 3D Gaussians](5703-dynamic-3d-gaussian-splatting.md) which tracks motion via persistent view synthesis and local constraints, GaussianFlow uses a global flow-based deformation model, achieving 5× faster training than per-frame 3DGS while preserving real-time rendering speeds [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of images (multi-view or monocular video) with known camera poses.
*   **Outputs:** A set of 3D Gaussians $\{G_i\}$ where each Gaussian $i$ has time-varying attributes: position $\boldsymbol{\mu}_i(t)$, rotation $q_i(t)$, and radiance/color $c_i(t)$, along with opacity $\alpha_i$ (static or time-varying depending on implementation, typically static in base DDDM but modulated by flow) and scale $\sigma_i$ (static or time-varying). The method outputs a 4D representation that can be rendered at any arbitrary time $t$.

**Data Structures:**
Each Gaussian $i$ stores:
*   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Base position at reference time $t_0$.
*   $q_i \in \mathbb{R}^4$: Base rotation (quaternion) at $t_0$.
*   $c_i \in \mathbb{R}^3$: Base radiance at $t_0$.
*   $\sigma_i \in \mathbb{R}^3$: Scale.
*   $\alpha_i \in \mathbb{R}$: Opacity.
*   Deformation parameters for DDDM:
    *   Polynomial coefficients $\mathbf{P}_{pos}, \mathbf{P}_{rot}, \mathbf{P}_{color}$ for time-domain fitting.
    *   Fourier coefficients $\mathbf{F}_{pos}, \mathbf{F}_{rot}, \mathbf{F}_{color}$ for frequency-domain fitting.
    *   (Note: Exact bit-packing of coefficients is not specified; typically stored as floating-point arrays) [S3].

**The Dual-Domain Deformation Model (DDDM):**
The core innovation is modeling the time-dependent residual of each attribute using a combination of polynomial fitting (time domain) and Fourier series fitting (frequency domain) [S2, S3].

For a Gaussian $i$ and time $t$, the deformation residual $\Delta \mathbf{A}_i(t)$ for attribute $\mathbf{A} \in \{\boldsymbol{\mu}, q, c\}$ is:
$$ \Delta \mathbf{A}_i(t) = \mathbf{P}_i(t) + \mathbf{F}_i(t) $$
where:
*   $\mathbf{P}_i(t)$ is a polynomial function of $t$:
    $$ \mathbf{P}_i(t) = \sum_{k=0}^{K} \mathbf{a}_{k,i} t^k $$
    Here, $\mathbf{a}_{k,i}$ are learnable polynomial coefficients for Gaussian $i$, and $K$ is the polynomial degree (typically small, e.g., 2-4) [S3].
*   $\mathbf{F}_i(t)$ is a Fourier series approximation:
    $$ \mathbf{F}_i(t) = \sum_{m=1}^{M} \left( \mathbf{b}_{m,i} \cos(\omega_m t) + \mathbf{c}_{m,i} \sin(\omega_m t) \right) $$
    Here, $\mathbf{b}_{m,i}, \mathbf{c}_{m,i}$ are learnable Fourier coefficients, $\omega_m$ are fixed angular frequencies (e.g., $\omega_m = \frac{2\pi m}{T}$ where $T$ is the total duration), and $M$ is the number of harmonics [S2, S3].

The final attributes at time $t$ are:
$$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta \boldsymbol{\mu}_i(t) $$
$$ q_i(t) = \text{Normalize}(q_i \otimes \Delta q_i(t)) $$
$$ c_i(t) = c_i + \Delta c_i(t) $$
where $\otimes$ denotes quaternion multiplication [S3].

**Adaptive Timestep Scaling:**
To prevent optimization bias towards frames with violent motion due to uniform time division, adaptive timestep scaling is applied. The time domain is normalized such that frames with higher motion variance are weighted differently, or the effective time step is scaled based on local motion magnitude [S2, S3]. Specifically, the time $t$ is mapped to a scaled time $t'$ where the distribution of $t'$ accounts for the density of keyframes or motion intensity [S3].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from static 3DGS reconstruction at $t_0$. Initialize DDDM coefficients to zero.
2.  **Static Optimization:** Optimize static attributes ($\boldsymbol{\mu}, q, c, \sigma, \alpha$) and DDDM coefficients for a subset of frames or the reference frame to establish a baseline [S2].
3.  **Dynamic Optimization Loop:**
    a.  **Forward Pass:** For each training image at time $t_j$:
        i.  Compute deformation residuals $\Delta \mathbf{A}_i(t_j)$ using DDDM.
        ii. Update Gaussian attributes to $\mathbf{A}_i(t_j)$.
        iii. Render the image using differentiable rasterization [S3].
    b.  **Loss Computation:**
        i.  **Image Loss:** $\mathcal{L}_{img} = \lambda_{rgb} \mathcal{L}_{rgb} + \lambda_{ssim} \mathcal{L}_{SSIM}$ between rendered and ground truth images [S2].
        ii. **Temporal Smoothness Loss:** Encourages smooth deformation over time. For adjacent frames $t_j, t_{j+1}$:
            $$ \mathcal{L}_{smooth} = \sum_i \| \Delta \mathbf{A}_i(t_{j+1}) - \Delta \mathbf{A}_i(t_j) \|^2 $$
            (Note: Exact formulation may vary, but it penalizes abrupt changes in deformation residuals) [S2].
        iii. **KNN Rigid Loss:** Regularizes motion trajectory by assuming local neighborhoods of Gaussians move rigidly. For a Gaussian $i$ and its K nearest neighbors $N(i)$:
            $$ \mathcal{L}_{rigid} = \sum_i \sum_{k \in N(i)} \| (\boldsymbol{\mu}_k(t) - \boldsymbol{\mu}_i(t)) - R_i (\boldsymbol{\mu}_k(t_0) - \boldsymbol{\mu}_i(t_0)) \|^2 $$
            where $R_i$ is a rotation matrix derived from $q_i(t)$ and $q_i(t_0)$ [S2, S3].
        iv. **Flow Loss (Optional/Implicit):** If 2D flow estimation is used, a bidirectional optical flow loss is introduced to align dynamic objects across frames [S2].
    c.  **Backward Pass:** Compute gradients w.r.t. DDDM coefficients and static attributes.
    d.  **Update:** Update parameters using Adam optimizer [S2].
4.  **Iteration:** Repeat step 3 until convergence [S2].

**Parameters:**
*   Polynomial degree $K$: Not given in the sources.
*   Fourier harmonics $M$: Not given in the sources.
*   Learning rate: Not given in the sources.
*   Loss weights $\lambda_{rgb}, \lambda_{ssim}, \lambda_{smooth}, \lambda_{rigid}$: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Overfitting to Violent Motion:** Uniform time division can cause the model to overfit to frames with high motion. Adaptive timestep scaling mitigates this [S3].
*   **Precision:** Fourier series may require high $M$ for high-frequency motion, increasing memory. Polynomial fitting may struggle with non-smooth motion [S2].
*   **Boundary Conditions:** The method assumes the scene is captured within the time range $[t_{min}, t_{max}]$. Extrapolation outside this range is not guaranteed to be accurate [S3].

## Subtopics

### Gaussian Splatting Lucas-Kanade
A method that directly applies optical flow estimation to the rendered Gaussians, using the flow to align dynamic objects across frames and improve temporal consistency [S1, unofficial]. Unlike GaussianFlow which models deformation via DDDM, this variant uses explicit 2D flow fields to guide Gaussian movement, potentially offering simpler implementation but less explicit 3D motion modeling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is 5× faster than per-frame 3DGS modeling [S3]. Rendering speed is comparable to original 3DGS, enabling real-time performance [S3].
*   **Memory Complexity:** Higher than static 3DGS due to storage of DDDM coefficients (polynomial and Fourier) for each Gaussian. Lower than implicit neural field methods (like NeRF) which store large networks [S3].
*   **Measured Performance:** Achieves superior novel view rendering quality compared to previous leading methods on multi-view and monocular datasets (HyperNeRF, Plenoptic) [S3].
*   **Strengths:** Ultra-fast training and rendering; explicit 4D representation; supports both multi-view and monocular videos; no need for neural fields [S3].
*   **Limitations:** Requires careful tuning of DDDM parameters ($K, M$); may struggle with highly complex, non-rigid deformations if polynomial/Fourier bases are insufficient; adaptive timestep scaling adds complexity [S2, S3].
*   **When to Use:** Dynamic scene reconstruction from videos where real-time rendering and fast training are critical, and the motion is not extremely chaotic [S3].
*   **When Not to Use:** If the scene contains very high-frequency, non-smooth motion that cannot be captured by low-degree polynomials or limited Fourier harmonics, or if per-frame accuracy is more important than global consistency [S2].

## Background
GaussianFlow was introduced by Guo et al. in 2024 [S3]. It builds on the success of 3D Gaussian Splatting (3DGS) for static scenes and addresses the limitations of existing dynamic methods, which either suffer from slow training/rendering (NeRF-based) or lack global consistency (per-frame 3DGS) [S3]. The method was developed to enable fast dynamic scene reconstruction and real-time rendering from both multi-view and monocular videos [S3]. It is part of the broader family of dynamic 3D Gaussian Splatting methods, which aim to extend the explicit, efficient representation of 3DGS to 4D [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Gaussian-Flow: 4D Reconstruction with Dynamic 3D Gaussian Particle, https://doi.org/10.1109/cvpr52733.2024.01997, fetched 2026-10-26
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
