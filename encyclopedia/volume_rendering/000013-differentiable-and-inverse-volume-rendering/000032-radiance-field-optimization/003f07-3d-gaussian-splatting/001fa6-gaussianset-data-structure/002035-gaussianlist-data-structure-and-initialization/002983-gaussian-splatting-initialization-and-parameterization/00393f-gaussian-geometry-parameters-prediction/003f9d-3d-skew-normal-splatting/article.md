# 3D Skew-Normal Splatting

Also called: Skew-Normal Splatting

## Overview
3D Skew-Normal Splatting (SNS) is a volume rendering method that replaces the standard symmetric 3D Gaussian kernel with the Azzalini Skew-Normal distribution to model asymmetric structures like sharp boundaries and one-sided surfaces [S1, unofficial]. It sets itself apart from symmetric variants like [3D Gaussian Splatting](8441-3d-gaussian-splatting.md) and other asymmetric approaches like [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md) by providing continuous shape interpolation between symmetric Gaussians and half-ellipsoids while preserving analytical tractability under affine transformations [S1, unofficial]. This allows for seamless integration into existing splat-based rasterization pipelines without the distributional discontinuities introduced by hard truncation methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Skew-Normal primitives, each defined by location $\mu$, scale matrix $\Omega$, and latent skew variable $k$; camera parameters (view/projection matrices); image resolution.
*   **Outputs:** A rendered 2D image with pixel colors $C(u)$.

**Data Structures:**
Each primitive $i$ stores:
*   $\mu_i \in \mathbb{R}^3$: Location parameter.
*   $R_i \in SO(3)$: Rotation matrix.
*   $S_i \in \mathbb{R}^3$: Scale vector (diagonal elements of scaling matrix).
*   $\Omega_i = R_i S_i S_i^\top R_i^\top \in \mathbb{R}^{3 \times 3}$: Scale matrix.
*   $k_i \in \mathbb{R}^3$: Latent skew variable, factorized into magnitude $m_{k,i} \in \mathbb{R}_+$ and direction $d_{k,i} \in \mathbb{R}^3$ ($\|d_{k,i}\|_2=1$).
*   $c_i$: Color (SH coefficients).
*   $o_i \in [-1, 1]$: Signed opacity.

**Parameters:**
*   Skew magnitude parameterization: $m_k = \frac{8}{1 + \exp(-\frac{x}{6})}$, where $x \in \mathbb{R}$ is a learnable scalar [S1, unofficial].
*   Skew direction: $d_k = \frac{v}{\|v\|_2 + \epsilon}$, where $v \in \mathbb{R}^3$ is a learnable vector [S1, unofficial].
*   Skew vector: $k = m_k d_k$ [S1, unofficial].

**Steps:**

1.  **Parameter Decoupling:** Compute the slant parameter $\alpha$ to decouple skewness from rotation and scale:
    $$ \alpha = \omega R S^{-1} k $$
    where $\omega = (\Omega \odot I_3)^{1/2}$ is the diagonal matrix of the square roots of the diagonal elements of $\Omega$, and $\odot$ is the Hadamard product [S1, unofficial].

2.  **Mean Correction:** Calculate the corrected operational center for bounding box localization, as the location parameter $\mu$ does not coincide with the spatial mean:
    $$ \mu_{bbox} = \mu + \sqrt{\frac{2}{\pi}} \cdot \frac{R S k}{\sqrt{1+k^\top k}} $$
    This offset shifts the footprint to the true spatial mean $\mathbb{E}[X]$ [S1, unofficial].

3.  **2D Projection:** Project the 3D primitive to the image plane using the affine transformation matrix $A = JW \in \mathbb{R}^{2 \times 3}$ (where $J$ is the Jacobian of the projection and $W$ is the view matrix):
    *   Projected center: $\mu^{2D} = A\mu$
    *   Projected covariance: $\Omega^{2D} = A\Omega A^\top$
    *   Projected skewness parameter $m^{2D}$:
        $$ m^{2D} = \frac{(\Omega^{2D})^{-1} A R S k}{\sqrt{1 + k^\top k} - (A R S k)^\top (\Omega^{2D})^{-1} A R S k} $$
    The 2D Skew-Normal kernel is then:
    $$ SN^{2D}(u; \mu^{2D}, \Omega^{2D}, m^{2D}) = G_2(u - \mu^{2D}; \Omega^{2D}) \Phi \left( (m^{2D})^\top (u - \mu^{2D}) \right) $$
    where $G_2$ is the 2D Gaussian PDF and $\Phi$ is the standard normal CDF [S1, unofficial].

4.  **Rasterization:** For each pixel $u$, aggregate colors from depth-sorted primitives $i \in \mathcal{N}$ using signed opacity blending:
    $$ C(u) = \sum_{i \in \mathcal{N}} c_i o_i SN_i^{2D}(u) \prod_{j=1}^{i-1} (1 - o_j SN_j^{2D}(u)) $$
    Signed opacity $o_i \in [-1, 1]$ allows subtractive volumetric density [S1, unofficial].

5.  **Optimization:**
    *   **Phase 1:** Jointly optimize all parameters ($R, S, m_k, d_k$) using standard Adam optimizer to establish global structure [S1, unofficial].
    *   **Phase 2:** Use Block Coordinate Descent (BCD) with a cyclic alternating schedule:
        *   Part 1: Update ellipsoidal base parameters ($R, S$) while freezing skew parameters ($m_k, d_k$).
        *   Part 2: Refine skew parameters ($m_k, d_k$) while holding ($R, S$) constant [S1, unofficial].
    *   **Positional Update:** Update positional parameters $\mu$ using Stochastic Gradient Hamiltonian Monte Carlo (SGHMC) to escape local minima [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Unbounded Tails:** Unbounded asymmetric tails cause primitives to intersect excessive tiles, degrading tile-based rasterization performance and violating local affine approximation assumptions. The bounded skewness of the Skew-Normal distribution prevents this [S1, unofficial].
*   **Optimization Instability:** Strong coupling between scale, rotation, and skewness causes instability. The decoupled parameterization $\alpha = \omega R S^{-1} k$ and factorization of $k$ into $m_k, d_k$ resolves this [S1, unofficial].
*   **Bounding Box Error:** Using the nominal location parameter $\mu^{2D}$ for bounding boxes is ineffective because the slant parameter shifts the probability mass. The corrected center $\mu_{bbox}$ must be used [S1, unofficial].

## Subtopics

### 2D Skew-Normal Parameter Projection
Computes the parameters of a 2D Skew-Normal distribution resulting from the linear projection of a 3D Skew-Normal kernel onto the image plane. The skewness parameter $\alpha_Y$ is derived from the projected covariance and the transformed skew vector. The final projection formula for skewness is $\alpha_Y = \frac{\omega_Y \Omega_Y^{-1} A R S k}{\sqrt{1 + k^\top k} - (A R S k)^\top \Omega_Y^{-1} A R S k}$, where $\omega_Y = (\Omega_Y \odot I_2)^{1/2}$ [S1, unofficial].

### Skew-Normal Mean Correction
Addresses the fact that the location parameter $\mu$ of a Skew-Normal distribution does not equal its spatial mean. Using the decoupled parameterization, the true mean is $\mathbb{E}[X] = \mu + \sqrt{\frac{2}{\pi}} \cdot \frac{R S k}{\sqrt{1+k^\top k}}$. This correction is critical for accurate bounding-box localization in rasterization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Comparable to [Sign-Split Splatting](9435-sign-split-splatting.md) (SSS) and [3D Gaussian Splatting](8441-3d-gaussian-splatting.md) (3DGS) due to analytical tractability. On the drjohnson scene, SNS completes 30,000 iterations in 38m 4s, comparable to SSS's 38m 7s [S1, unofficial].
*   **Memory Complexity:** Similar to 3DGS, with additional storage for skew parameters ($m_k, d_k$) per primitive.
*   **Performance:** On the Mip-NeRF360 dataset, SNS achieves 30.17 PSNR, 0.895 SSIM, and 0.144 LPIPS, outperforming SSS (29.90 PSNR) and 3DGS (28.69 PSNR) [S1, unofficial].
*   **Strengths:** Continuous shape interpolation allows modeling of sharp boundaries and one-sided surfaces with fewer primitives than symmetric Gaussians. Bounded skewness ensures efficient tile-based rasterization.
*   **Limitations:** Optimization requires a two-phase strategy (BCD) to handle parameter coupling, potentially complicating implementation. Signed opacity requires careful handling of negative values during blending.
*   **When to use:** Scenes with sharp edges, flat surfaces, or asymmetric structures where symmetric Gaussians require excessive overlap.
*   **When not to use:** Scenes with purely symmetric, smooth geometry where the added complexity of skew parameters provides negligible benefit.

## Background
3D Skew-Normal Splatting was introduced in the paper "3D Skew-Normal Splatting" [S1, unofficial]. It builds on the foundation of [3D Gaussian Splatting](8441-3d-gaussian-splatting.md) and [Sign-Split Splatting](9435-sign-split-splatting.md), addressing their limitations in representing asymmetric features. The method leverages the Azzalini Skew-Normal distribution, known for its analytical properties under affine transformations, to extend the expressive power of Gaussian splatting primitives. It is implemented in PyTorch and tested on NVIDIA RTX A6000 GPUs [S1, unofficial].

## Sources
[S1] 3D Skew-Normal Splatting, https://doi.org/10.48550/arxiv.2605.15010, fetched 2026-10-26 (unofficial)
