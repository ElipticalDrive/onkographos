# Generalized Gaussian Particles

Also called: Generalized Gaussians of degree n

## Overview
Generalized Gaussian Particles replace the standard quadratic exponent in the 2D Gaussian probability density function with a power-law exponent controlled by a degree parameter $n$ [S1]. This modification allows for steeper and narrower density fall-off, enabling denser particle packing and finer control over the trade-off between rendering quality and computational speed compared to standard Gaussians [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Position $\mathbf{x} \in \mathbb{R}^2$, mean $\boldsymbol{\mu} \in \mathbb{R}^2$, covariance matrix $\boldsymbol{\Sigma} \in \mathbb{R}^{2 \times 2}$, kernel degree $n \in \mathbb{N}$, and scale factor $\lambda$ [S1].
*   **Output:** Kernel response $\rho(\mathbf{x}) \in [0, 1]$ [S1].

**Data Structures:**
The covariance matrix $\boldsymbol{\Sigma}$ is typically decomposed into a scaling matrix $\mathbf{S}$ and rotation matrix $\mathbf{R}$ such that $\boldsymbol{\Sigma} = \mathbf{R}\mathbf{S}\mathbf{S}^T\mathbf{R}^T$ [G]. The inverse quadratic form $(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x} - \boldsymbol{\mu})$ is computed using this decomposition or direct inversion [G].

**Parameters:**
*   $n$: Kernel degree. $n=2$ corresponds to the standard Gaussian. Higher degrees ($n=4, 8, \dots$) produce steeper and narrower fall-off [S1]. Note that the "degree 2" generalized Gaussian in 3DGRT corresponds to degree $n=4$ in this formulation [S1].
*   $\lambda$: Scale factor defined to get the same kernel response at a given distance $r$ as the reference Gaussian kernel [S1].
    *   Formula: $\lambda = r^{2-n} / 2$, derived by equating the generalized response $\exp(-\lambda r^n)$ to the standard Gaussian response $\exp(-r^2/2)$ at distance $r$ [G].
    *   Typical value: $r = 3$ [S1].

**Steps:**
1.  Compute the displacement vector: $\mathbf{d} = \mathbf{x} - \boldsymbol{\mu}$ [G].
2.  Compute the squared Mahalanobis distance (quadratic form): $q = \mathbf{d}^T \boldsymbol{\Sigma}^{-1} \mathbf{d}$ [G].
3.  Apply the power-law exponent: $p = q^{n/2}$ [S1].
4.  Scale by $\lambda$: $s = \lambda \cdot p$ [S1].
5.  Compute the response: $\rho(\mathbf{x}) = \exp(-s)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generalized_gaussian_response(x, mu, Sigma_inv, n, r=3.0):
    # 1. Displacement vector
    d = [x[0] - mu[0], x[1] - mu[1]]
    
    # 2. Quadratic form: d^T * Sigma_inv * d
    q = d[0] * (Sigma_inv[0][0] * d[0] + Sigma_inv[0][1] * d[1]) + \
        d[1] * (Sigma_inv[1][0] * d[0] + Sigma_inv[1][1] * d[1])
    
    # 3. Clamp to avoid NaN from negative values due to floating-point errors [G]
    if q < 0.0:
        q = 0.0
        
    # 4. Power-law exponent: q^(n/2)
    p = q ** (n / 2.0)
    
    # 5. Calculate lambda to match standard Gaussian response at distance r [G]
    lambda_val = (r ** (2 - n)) / 2.0
    
    # 6. Scale and Exponentiate
    s = lambda_val * p
    rho = math.exp(-s)
    
    return rho
```

**Edge Cases and Pitfalls:**
*   **Precision:** If $q$ is slightly negative due to floating-point errors, raising it to a fractional power ($n/2$) results in NaN. Clamp $q$ to $\ge 0$ [G].
*   **Degree Selection:** Higher degrees ($n > 2$) result in narrower particles [S1].
*   **3DGRT Compatibility:** When integrating with existing pipelines using 3DGRT's "degree 2" generalized Gaussians, use $n=4$ in this formulation [S1].

## Subtopics

### [Flat-top Gaussian Feature Density](11118-flat-top-gaussian-feature-density.md)
Uses a differentiable approximation of feature density with flat-top Gaussians that become box-like as parameter $p$ increases. Chosen when box-like feature representation is needed.

## Operating characteristics
**Measured Performance:**
On the MipNeRF360 dataset (excluding flower and treehill scenes):
*   Degree $n=2$ (Standard): 28.77 PSNR at 207 FPS [S1].
*   Degree $n=4$: 28.46 PSNR at 233 FPS [S1].
*   Degree $n=8$: 27.63 PSNR at 243 FPS [S1].

Higher degrees increase rendering speed (FPS) but slightly decrease reconstruction quality (PSNR) [S1].

**Strengths and Limitations:**
*   **Strengths:** Allows tuning of particle density and fall-off steepness. Higher degrees enable faster rendering speeds [S1].
*   **Limitations:** Higher degrees reduce PSNR. The choice of $n$ requires balancing quality and speed [S1].

**When to Use:**
Use higher degrees ($n=4, 8$) when rendering speed is prioritized over maximum reconstruction quality, or when narrower particle support is beneficial for specific scene characteristics [S1]. Use $n=2$ for standard compatibility and highest reconstruction quality (28.77 PSNR on MipNeRF360) [S1].

## Background
Generalized Gaussian Particles were introduced in the context of 3DGRT, which proposed using particles with different kernel functions, specifically a generalized Gaussian of degree 2 [S1]. The formulation presented here generalizes this to arbitrary degree $n$ [S1]. The method is used in Gaussian Splatting variants to optimize the trade-off between rendering quality and speed [S1].

## Sources
[S1] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
