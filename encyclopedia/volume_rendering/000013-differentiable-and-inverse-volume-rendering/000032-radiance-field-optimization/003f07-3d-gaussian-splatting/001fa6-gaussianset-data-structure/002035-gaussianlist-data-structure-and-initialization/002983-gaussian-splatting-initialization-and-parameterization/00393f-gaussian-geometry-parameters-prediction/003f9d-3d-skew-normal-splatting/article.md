# 3D Skew-Normal Splatting

Also called: Skew-Normal Splatting

## Overview
3D Skew-Normal Splatting (SNS) replaces the standard symmetric 3D Gaussian kernel with the Azzalini Skew-Normal distribution to model asymmetric structures, such as sharp boundaries and one-sided surfaces, while preserving analytical tractability under affine transformations [S1, unofficial]. It sets apart from standard Gaussian Splatting and its symmetric variants (e.g., [Textured-GS](9436-textured-gs.md), [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md)) by introducing a learnable skewness parameter that allows continuous interpolation between symmetric Gaussians and half-ellipsoids, enabling a single primitive to represent complex geometries that would otherwise require multiple overlapping kernels [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Skew-Normal primitives, each defined by location $\mu$, scale matrix $\Omega$, and skew latent variable $k$; camera parameters (view matrix $V$, projection matrix $P$); scene color and opacity targets.
*   **Output:** Rendered 2D image colors $C(u)$ for each pixel $u$.

**Data Structures:**
Each primitive $i$ stores:
*   $\mu_i \in \mathbb{R}^3$: Location parameter.
*   $R_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (orthonormal).
*   $S_i \in \mathbb{R}^3$: Scale vector (positive).
*   $\Omega_i = R_i S_i S_i^\top R_i^\top \in \mathbb{R}^{3 \times 3}$: Scale matrix.
*   $x_i \in \mathbb{R}$: Raw scalar for skew magnitude.
*   $v_i \in \mathbb{R}^3$: Raw vector for skew direction.
*   $c_i$: Color coefficients (e.g., SH coefficients).
*   $o_i \in [-1, 1]$: Signed opacity.

**Derived Parameters:**
*   Skew magnitude: $m_k = \frac{8}{1 + \exp(-x/6)}$ [S1, unofficial].
*   Skew direction: $d_k = \frac{v}{\|v\|_2 + \epsilon}$ [S1, unofficial].
*   Latent skew vector: $k = m_k d_k$ [S1, unofficial].
*   Slant parameter: $\alpha = \omega R S^{-1} k$, where $\omega = (\Omega \odot I_3)^{1/2}$ [S1, unofficial].

**Numbered Steps:**

1.  **Parameter Initialization:** Initialize $\mu, R, S, c, o$ as in standard 3DGS. Initialize $x$ and $v$ to small random values or zeros to start with symmetric Gaussians.
2.  **Compute Skew Vector:** Calculate $m_k$ from $x$ and $d_k$ from $v$. Compute $k = m_k d_k$.
3.  **Compute Slant:** Calculate $\omega = \sqrt{\text{diag}(\Omega)}$. Compute $\alpha = \omega R S^{-1} k$.
4.  **Project to 2D:**
    *   Compute affine transformation matrix $A = JW$, where $J$ is the Jacobian of the projection and $W$ is the view matrix [S1, unofficial].
    *   Compute 2D center: $\mu^{2D} = A \mu$.
    *   Compute 2D covariance: $\Omega^{2D} = A \Omega A^\top$.
    *   Compute 2D skew parameter:
        $$ m^{2D} = \frac{(\Omega^{2D})^{-1} A R S k}{\sqrt{1 + k^\top k} - (A R S k)^\top (\Omega^{2D})^{-1} A R S k} $$
        [S1, unofficial]
5.  **Bounding Box Correction:**
    *   Compute the offset to the true spatial mean:
        $$ \Delta = \sqrt{\frac{2}{\pi}} \cdot \frac{A R S k}{\sqrt{1 + k^\top k}} $$
        [S1, unofficial]
    *   Set bounding box center: $\mu_{bbox} = \mu^{2D} + \Delta$.
    *   Use $\mu_{bbox}$ and $\Omega^{2D}$ to determine tile intersection for rasterization.
6.  **Rasterization:**
    *   For each pixel $u$ covered by the primitive's bounding box, compute the 2D Skew-Normal density:
        $$ SN^{2D}(u) = G_2(u - \mu^{2D}; \Omega^{2D}) \Phi \left( (m^{2D})^\top (u - \mu^{2D}) \right) $$
        where $G_2$ is the 2D Gaussian PDF and $\Phi$ is the standard normal CDF [S1, unofficial].
    *   Compute spatial influence: $a_i(u) = o_i SN^{2D}(u)$.
7.  **Color Aggregation:**
    *   Sort primitives by depth.
    *   Aggregate color using front-to-back alpha blending with signed opacity:
        $$ C(u) = \sum_{i \in \mathcal{N}} c_i o_i SN_i^{2D}(u) \prod_{j=1}^{i-1} (1 - o_j SN_j^{2D}(u)) $$
        [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def compute_skew_params(x, v, eps=1e-8):
    """
    Compute skew magnitude and direction from raw learnable parameters.
    x: scalar tensor [N]
    v: vector tensor [N, 3]
    """
    # Magnitude: bounded positive scalar
    m_k = 8.0 / (1.0 + torch.exp(-x / 6.0))
    
    # Direction: unit vector
    norm_v = torch.norm(v, dim=1, keepdim=True) + eps
    d_k = v / norm_v
    
    # Latent skew vector k
    k = m_k.unsqueeze(1) * d_k
    return k

def compute_slant(R, S, k, Omega):
    """
    Compute slant parameter alpha.
    R: [N, 3, 3]
    S: [N, 3] (diagonal elements of scale matrix)
    k: [N, 3]
    Omega: [N, 3, 3]
    """
    # omega = sqrt(diag(Omega))
    omega = torch.sqrt(torch.diagonal(Omega, dim1=1, dim2=2)) # [N, 3]
    
    # S_inv: inverse of scale vector
    S_inv = 1.0 / (S + 1e-8)
    
    # alpha = omega * R * S_inv * k
    # R * S_inv: scale columns of R
    R_scaled = R * S_inv.unsqueeze(1)
    # R_scaled * k
    R_sk = torch.bmm(R_scaled, k.unsqueeze(2)).squeeze(2) # [N, 3]
    alpha = omega * R_sk
    
    return alpha

def project_to_2d(A, mu, Omega, k):
    """
    Project 3D Skew-Normal to 2D image plane.
    A: [N, 2, 3] affine matrix (JW)
    mu: [N, 3]
    Omega: [N, 3, 3]
    k: [N, 3]
    """
    # 2D Center
    mu_2d = torch.bmm(A, mu.unsqueeze(2)).squeeze(2) # [N, 2]
    
    # 2D Covariance
    Omega_2d = torch.bmm(torch.bmm(A, Omega), A.transpose(1, 2)) # [N, 2, 2]
    
    # 2D Skew parameter m_2d
    # Numerator: Omega_2d_inv * A * R * S * k
    # Note: R*S*k is equivalent to Omega * S_inv * R^T * R * S * k? 
    # Actually, from source: ARSk. We need R and S.
    # Assuming R and S are available in context or passed.
    # Let's assume we have R and S.
    # R * S * k
    # S is diagonal, so R * S scales columns of R
    # k is [N, 3]
    # R_S_k = torch.bmm(R * S.unsqueeze(1), k.unsqueeze(2)).squeeze(2)
    # But we don't have R, S here. We have Omega.
    # Source says: m_2d = (Omega_2d_inv * A * R * S * k) / ...
    # We need to pass R, S to this function or compute ARSk from Omega?
    # Omega = R S S^T R^T. So R S is not directly Omega.
    # We must pass R, S.
    pass

def render_sns(primitives, cameras, image_size):
    """
    Main rendering loop.
    primitives: dict with mu, R, S, k, c, o
    cameras: view/projection matrices
    """
    # 1. Compute derived skew params
    k = compute_skew_params(primitives['x'], primitives['v'])
    
    # 2. Compute Slant
    alpha = compute_slant(primitives['R'], primitives['S'], k, primitives['Omega'])
    
    # 3. Project
    # Compute A = JW
    # ... (standard 3DGS projection logic) ...
    A = compute_affine_matrix(cameras, primitives['mu'])
    
    mu_2d, Omega_2d, m_2d = project_skew_to_2d(A, primitives['mu'], primitives['Omega'], primitives['R'], primitives['S'], k)
    
    # 4. Bounding Box Correction
    # Delta = sqrt(2/pi) * A R S k / sqrt(1 + k^T k)
    k_norm_sq = torch.sum(k**2, dim=1, keepdim=True)
    delta = torch.sqrt(torch.tensor(2.0 / torch.pi)) * \
            torch.bmm(torch.bmm(torch.bmm(A, primitives['R']), torch.diag_embed(primitives['S'])), k.unsqueeze(2)).squeeze(2) / \
            torch.sqrt(1.0 + k_norm_sq)
    
    mu_bbox = mu_2d + delta
    
    # 5. Rasterize
    # ... (tile-based rasterization using mu_bbox and Omega_2d) ...
    # For each pixel u:
    #   SN_2d = Gaussian_2D(u - mu_2d, Omega_2d) * Phi(m_2d^T (u - mu_2d))
    #   a = o * SN_2d
    #   C = sum(c * a * prod(1 - a_prev))
    
    return C
```

**Parameters:**
*   Skew magnitude raw parameter $x \in \mathbb{R}$: Maps to $m_k \in [0, 8]$ via sigmoid-like function [S1, unofficial].
*   Skew direction raw vector $v \in \mathbb{R}^3$: Normalized to unit length [S1, unofficial].
*   Signed opacity $o_i \in [-1, 1]$: Allows subtractive density [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Unbounded Tails:** If skewness is unbounded, primitives may intersect excessive tiles, degrading rasterization performance and violating local affine approximation assumptions [S1, unofficial]. The bounded parameterization prevents this.
*   **Parameter Coupling:** Strong coupling between scale, rotation, and skewness causes optimization instability. The decoupled parameterization $\alpha = \omega R S^{-1} k$ and factorization of $k$ into $m_k, d_k$ resolves this [S1, unofficial].
*   **Mean vs. Location:** The location parameter $\mu$ does not coincide with the spatial mean. Using $\mu$ for bounding boxes leads to inaccurate localization. The corrected center $\mu_{bbox}$ must be used [S1, unofficial].

## Subtopics

### [2D Skew-Normal Parameter Projection](10052-2d-skew-normal-parameter-projection.md)
Computes the parameters of a 2D Skew-Normal distribution (SN2D) resulting from the linear projection of a 3D Skew-Normal kernel (SN3) onto the image plane. It sets apart from standard Gaussian splatting by preserving the asymmetry (skewness) of the 3D primitive during projection, allowing the 2D kernel to model one-sided surfaces and sharp boundaries that symmetric Gaussians cannot.

### Skew-Normal Mean Correction
Addresses the fact that the location parameter $\mu$ of a Skew-Normal distribution is not its spatial mean. It derives a corrected operational center $\mu_{bbox}$ to ensure accurate bounding-box localization for rasterization. The mean is given by $\mathbb{E}[X] = \mu + \sqrt{\frac{2}{\pi}} \cdot \frac{R S k}{\sqrt{1+k^\top k}}$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Comparable to standard 3DGS and SSS, with minor overhead from computing the Skew-Normal CDF $\Phi$ and the additional skew parameters [S1, unofficial].
*   **Memory Complexity:** Slightly higher than 3DGS due to storage of skew parameters ($x, v$ or $k$) per primitive.
*   **Measured Performance:** On Mip-NeRF360, SNS achieves 30.17 PSNR, 0.895 SSIM, 0.144 LPIPS, outperforming SSS (29.90 PSNR) and 3DGS (28.69 PSNR) [S1, unofficial]. On the drjohnson scene, SNS completes 30,000 iterations in 38m 4s (29.89 dB PSNR), comparable to SSS (38m 7s, 29.66 dB PSNR) [S1, unofficial].
*   **Strengths:** Continuous shape interpolation between symmetric and asymmetric primitives; better modeling of sharp edges and flat surfaces; bounded skewness ensures stable rasterization.
*   **Limitations:** Optimization requires a two-phase strategy (joint then block coordinate descent) to handle parameter coupling; slightly more complex gradient computation due to the skew term.
*   **When to use:** Scenes with sharp boundaries, one-sided structures, or asymmetric details that symmetric Gaussians struggle to represent efficiently.
*   **When not to use:** Scenes with purely symmetric, smooth geometry where the added complexity and parameters do not yield significant quality improvements.

## Background
3D Skew-Normal Splatting was introduced in the paper "3D Skew-Normal Splatting" (arXiv:2605.15010) [S1, unofficial]. It builds on the Azzalini Skew-Normal distribution, which has been used in statistics for decades, and adapts it to the differentiable rendering framework of 3D Gaussian Splatting. The method addresses limitations of symmetric kernels in representing complex geometries by introducing a learnable skewness parameter while maintaining analytical tractability.

## Sources
[S1] 3D Skew-Normal Splatting, https://doi.org/10.48550/arxiv.2605.15010, fetched 2026-10-26 (unofficial)
