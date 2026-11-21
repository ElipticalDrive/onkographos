# Dual-Domain Deformation Model


## Overview
The Dual-Domain Deformation Model (DDDM) is a method for representing the temporal dynamics of 3D Gaussian particles in dynamic scene reconstruction. It solves the problem of modeling complex, time-varying scene geometry and appearance by explicitly fitting attribute residuals to time-dependent curves, eliminating the need for per-frame models or implicit neural fields. It sets itself apart from sibling methods like [KNN Rigid Loss](12815-knn-rigid-loss.md) by focusing on the parametric fitting of individual Gaussian attributes rather than spatial consistency constraints, and from [Adaptive Spatio-Temporal 3D Gaussian Splatting](9535-adaptive-spatio-temporal-3d-gaussian-splatting.md) by introducing a specific dual-domain fitting strategy for attribute deformation.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian particles. Each particle $i$ has base attributes at a reference time $t_0$: position $\mu_0$, rotation $q_0$, radiance $c_0$, scaling $s_0$, and opacity $\alpha_0$. The input also includes the normalized frame index $t \in [0, 1]$ for each observation.
*   **Output:** Deformed attributes $\mu(t), q(t), c(t)$ for any time $t$, while $s(t)=s_0$ and $\alpha(t)=\alpha_0$ remain constant.

**Data Structures:**
Each Gaussian particle stores:
1.  **Base Attributes:** $\mu_0 \in \mathbb{R}^3$, $q_0 \in \mathbb{R}^4$ (quaternion), $c_0 \in \mathbb{R}^3$ (SH coefficients or RGB), $s_0 \in \mathbb{R}^3$, $\alpha_0 \in \mathbb{R}$.
2.  **Deformation Coefficients:**
    *   Polynomial coefficients $a = \{a_n\}_{n=0}^N$ for each dimension of $\mu, q, c$.
    *   Fourier coefficients $f = \{f_l^{\sin}, f_l^{\cos}\}_{l=1}^L$ for each dimension of $\mu, q, c$.
    *   Scaling parameters $\lambda_s, \lambda_b$ for adaptive timestamp scaling.

**Numbered Steps:**
1.  **Initialization:** Set reference time $t_0$ (typically the first frame). Initialize base attributes $\mu_0, q_0, c_0, s_0, \alpha_0$ from static 3DGS reconstruction. Initialize deformation coefficients $a, f$ to zero or small random values. Initialize $\lambda_s = 1, \lambda_b = 0$.
2.  **Attribute Selection:** Identify attributes to model over time: position $\mu$, rotation $q$, and radiance $c$. Scaling $s$ and opacity $\alpha$ are held constant.
3.  **Dimension-wise Assignment:** Assign a separate deformation function $D(t)$ for each dimension of the selected attributes. For position $\mu$, use $\{D_{\mu_i}(t)\}_{i=0}^2$. For rotation $q$, use $\{D_{q_j}(t)\}_{j=0}^3$. For radiance $c$, use $\{D_{c_k}(t)\}_{k=0}^2$ (or per SH coefficient).
4.  **Temporal Scaling (Optional):** For each Gaussian, compute the scaled time input $t_s$ using the adaptive timestamp scaling technique:
    $$ t_s = \lambda_s \cdot t + \lambda_b $$
    where $t$ is the normalized frame index.
5.  **Residual Calculation:** Compute the time-dependent residual $D(t)$ for each dimension using the dual-domain model:
    $$ D(t) = P_N(t) + F_L(t) $$
    where $P_N(t)$ is the polynomial component and $F_L(t)$ is the Fourier series component.
6.  **Attribute Reconstruction:** Compute the deformed attribute $S(t)$ for a dimension with base value $S_0$:
    $$ S(t) = S_0 + D(t) $$
    For rotation, ensure the result is normalized to a unit quaternion after addition.

**Formulas:**
The residual $D(t)$ is defined as:
$$ D(t) = \sum_{n=0}^{N} a_n t^n + \sum_{l=1}^{L} \left( f_l^{\sin} \cos(l t_s) + f_l^{\cos} \sin(l t_s) \right) $$
where:
*   $N$ is the polynomial order.
*   $L$ is the Fourier series order.
*   $a_n$ are polynomial coefficients.
*   $f_l^{\sin}, f_l^{\cos}$ are Fourier coefficients.
*   $t_s$ is the scaled time input.

**Parameters:**
*   **Polynomial Order $N$:** Hyperparameter controlling polynomial complexity. Not given in the sources.
*   **Fourier Series Order $L$:** Hyperparameter controlling Fourier complexity. In experiments on HyperNeRF, $L=16$ yields best results; performance drops at $L=32$ due to over-parameterization [S1].
*   **Reference Time $t_0$:** Usually the first frame [S1].
*   **Scaling Factors:** $\lambda_s$ initialized to 1, $\lambda_b$ initialized to 0 [S1].

**Edge Cases and Pitfalls:**
*   **Polynomial Overfitting:** High-order polynomials can overfit violent motion, causing unreasonable oscillations in the fitted trajectory [S1].
*   **Fourier Underfitting:** Fourier series alone requires manual order reduction for smooth motion to avoid unnecessary complexity [S1].
*   **Optimization Instability:** Using standard normalized time $t \in [0,1]$ for substantial motions in short time frames requires exceedingly large coefficients, inducing instability or breakdown in optimization [S1]. Adaptive timestamp scaling mitigates this.
*   **Rotation Normalization:** Since $q$ is a quaternion, adding a residual $D_q(t)$ may result in a non-unit quaternion. The result must be normalized [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_deformation_residual(t, t_s, poly_coeffs, fourier_coeffs, N, L):
    """
    Computes D(t) = P_N(t) + F_L(t_s)
    t: raw time (unused if t_s is used directly in Fourier, but kept for consistency)
    t_s: scaled time input
    poly_coeffs: list/array of length N+1, [a_0, ..., a_N]
    fourier_coeffs: tuple (f_sin, f_cos), each of length L
    N: polynomial order
    L: Fourier order
    """
    # Polynomial part
    poly_val = 0.0
    for n in range(N + 1):
        poly_val += poly_coeffs[n] * (t_s ** n)
    
    # Fourier part
    fourier_val = 0.0
    f_sin, f_cos = fourier_coeffs
    for l in range(1, L + 1):
        fourier_val += f_sin[l-1] * np.cos(l * t_s) + f_cos[l-1] * np.sin(l * t_s)
        
    return poly_val + fourier_val

def get_deformed_attribute(base_attr, t_s, poly_coeffs, fourier_coeffs, N, L, is_rotation=False):
    """
    Computes S(t) = S_0 + D(t)
    base_attr: scalar base value
    t_s: scaled time
    poly_coeffs: coefficients for this specific dimension
    fourier_coeffs: coefficients for this specific dimension
    is_rotation: if True, normalize result to unit magnitude (for quaternion components)
    """
    residual = compute_deformation_residual(t_s, t_s, poly_coeffs, fourier_coeffs, N, L)
    deformed = base_attr + residual
    
    if is_rotation:
        # Note: For quaternions, simple addition is an approximation. 
        # The source implies direct addition S(t) = S_0 + D(t).
        # Normalization is required to maintain unit quaternion property [G].
        deformed = deformed / np.linalg.norm(deformed) if np.linalg.norm(deformed) > 1e-8 else deformed
        
    return deformed

def get_gaussian_attributes(gaussian, t, lambda_s, lambda_b):
    """
    Returns deformed attributes for a Gaussian at time t.
    gaussian: object with attributes _mu0, _q0, _c0, _s0, _alpha0
              and deformation coefficients _poly_mu, _fourier_mu, etc.
    t: normalized frame index [0, 1]
    """
    # Adaptive timestamp scaling
    t_s = lambda_s * t + lambda_b
    
    # Position
    mu = np.zeros(3)
    for i in range(3):
        mu[i] = get_deformed_attribute(gaussian._mu0[i], t_s, gaussian._poly_mu[i], gaussian._fourier_mu[i], N, L)
        
    # Rotation (Quaternion)
    q = np.zeros(4)
    for j in range(4):
        q[j] = get_deformed_attribute(gaussian._q0[j], t_s, gaussian._poly_q[j], gaussian._fourier_q[j], N, L, is_rotation=True)
        
    # Radiance
    c = np.zeros(3)
    for k in range(3):
        c[k] = get_deformed_attribute(gaussian._c0[k], t_s, gaussian._poly_c[k], gaussian._fourier_c[k], N, L)
        
    # Scaling and Opacity (Constant)
    s = gaussian._s0
    alpha = gaussian._alpha0
    
    return mu, q, c, s, alpha
```

## Subtopics

### [KNN Rigid Loss](12815-knn-rigid-loss.md)
The KNN Rigid Loss is a regularization term used in dynamic 3D Gaussian Splatting to enforce spatial local consistency by penalizing differences in attribute residuals between a Gaussian point and its K nearest neighbors. It addresses the instability caused by dynamic density control (adding/removing points) by restricting the application of spatial constraints to optimization stages where the point set is static.

### Adaptive Timestamp Scaling
Adaptive Timestamp Scaling is a technique to scale the temporal input for each Gaussian point to prevent optimization instability caused by large coefficients when modeling substantial motions in short time frames [S1]. It uses a dilation factor $\lambda_s$ and a base factor $\lambda_b$ to transform the normalized time $t$ into $t_s = \lambda_s \cdot t + \lambda_b$ [S1]. $\lambda_s$ is initialized to 1 and $\lambda_b$ to 0 [S1]. Without this, standard normalized time $t \in [0,1]$ for substantial motions requires exceedingly large coefficients, inducing instability or breakdown in optimization [S1].

## Operating characteristics
*   **Time Complexity:** The deformation model adds a linear cost in the number of attributes and dimensions per Gaussian per frame. The joint polynomial and Fourier series fitting is computationally cheap compared to implicit neural fields, preserving the ultra-fast training and rendering speed of the original 3DGS [S1].
*   **Memory Complexity:** Each Gaussian stores additional coefficients for position, rotation, and radiance deformation. The memory overhead is proportional to $(N + 2L) \times \text{dimensions}$ per attribute [S1].
*   **Measured Performance:** Achieves a 5× faster training speed compared to per-frame 3DGS modeling [S1].
*   **Strengths:** Capable of modeling complex scene deformations across long video footage; eliminates the need for training separate 3DGS for each frame or introducing an additional implicit neural field; preserves discretized nature of 3D Gaussian points [S1].
*   **Limitations:** Requires careful tuning of polynomial order $N$ and Fourier order $L$ to balance smooth and violent motions; over-parameterization (e.g., $L > 32$) can degrade performance [S1].
*   **When to use:** When ultra-fast training and rendering of 4D scenes are required, and the scene dynamics can be approximated by polynomial and Fourier series residuals.
*   **When not to use:** When the scene dynamics are too complex to be captured by low-order polynomials and Fourier series, or when high-frequency, non-periodic motions dominate without sufficient data coverage.

## Background
The Dual-Domain Deformation Model was proposed in the paper "Gaussian-Flow: 4D Reconstruction with Dynamic 3D Gaussian Particle" by [S1]. It builds on the advancements in point-based 3D Gaussian Splatting (3DGS) to address the slow training and rendering speeds of prevalent NeRF-based approaches for dynamic scenes. The method leverages the explicit deformation modeling of discretized Gaussian points to achieve efficiency comparable to static 3DGS.

## Sources
[S1] Gaussian-Flow: 4D Reconstruction with Dynamic 3D Gaussian Particle, https://doi.org/10.1109/cvpr52733.2024.01997, fetched 2026-10-26
