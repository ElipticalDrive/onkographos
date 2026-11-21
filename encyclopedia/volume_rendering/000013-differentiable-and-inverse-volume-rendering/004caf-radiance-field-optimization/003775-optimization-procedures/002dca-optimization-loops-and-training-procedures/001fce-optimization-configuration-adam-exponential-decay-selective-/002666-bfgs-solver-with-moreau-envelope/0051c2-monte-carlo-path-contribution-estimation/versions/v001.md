# Monte Carlo Path Contribution Estimation

Also called: Caustic contribution estimation, RRTE path estimator

## Overview
This method estimates the radiance contribution of a stationary light path found by the BFGS solver, based on the path formulation of the Radiative Transfer Equation (RRTE) [S1]. It differs from standard path tracing by evaluating contributions for specific, optimized stationary paths rather than sampling paths stochastically from the camera, and it differs from the parent BFGS solver by handling the final radiance accumulation and variance reduction [S1].

## How it works
**Inputs:**
*   Stationary path $\bar{\mathbf{x}}_k = (\mathbf{x}_0, \mathbf{x}_1, \dots, \mathbf{x}_n)$ found by the BFGS solver, where $\mathbf{x}_0$ is the receiver (camera) and $\mathbf{x}_n$ is the light source [S1].
*   Refractive Index Field (RIF) $\eta(\mathbf{x})$ describing the media [S1].
*   Extinction coefficient $\sigma_t(\mathbf{x})$ [S1].
*   Final inverse Hessian matrix from the BFGS solver [S1].
*   Scene geometry for occlusion testing [S1].

**Outputs:**
*   Estimated radiance contribution $I$ for the path [S1].

**Data Structures:**
*   Path points $\mathbf{x}_i \in \mathbb{R}^3$ [S1].
*   Generalized geometric factor $G$ [S1].
*   Transmittance $T$ [S1].

**Steps:**

1.  **Transmittance Integration:** Numerically integrate the transmittance along each ray section $\Gamma_k$ between $\mathbf{x}_k$ and $\mathbf{x}_{k+1}$. The transmittance is defined as:
    $$T(\mathbf{x}_k, \mathbf{x}_{k+1}) = \exp \left( - \int_{t_k}^{t_{k+1}} \sigma_t(\mathbf{x}(\tau)) \eta(\mathbf{x}(\tau)) d\tau \right)$$
    Assuming $\sigma_t$ varies slowly within a single tetrahedral cell, use a composite Simpson rule for numerical integration [S1].

2.  **Occlusion Test:** Trace a shadow ray from $\mathbf{x}_0$ in the direction of $\mathbf{x}_1$. Check for intersections with the rest of the scene or bubbles inside the panel. If an intersection occurs, the path is occluded and its contribution is zero [S1].

3.  **Generalized Geometric Factor:** Compute the generalized geometric factor $G(\mathbf{x}_1 \leftrightarrow \dots \leftrightarrow \mathbf{x}_{n-1})$ for the specular interfaces. This is computed using the implicit function theorem and the final inverse Hessian matrix estimation produced by the BFGS solver [S1].

4.  **Regularization:** Regularize $G$ to prevent instability and high variance:
    $$\bar{G}(\mathbf{x}_1 \leftrightarrow \dots \leftrightarrow \mathbf{x}_{n-1}) = \max(G(\mathbf{x}_1 \leftrightarrow \dots \leftrightarrow \mathbf{x}_{n-1}), \epsilon)$$
    Typical values for $\epsilon$ are between $10^{-2}$ and $10^{-3}$ [S1].

5.  **Probability Estimation:** Estimate the inverse probability $1/p(\bar{\mathbf{x}}_k)$ of the BFGS solver converging to the path $\bar{\mathbf{x}}_k$ using the unbiased iterative procedure described in [21] (not provided in source [S1]) [S1].

6.  **Contribution Calculation:** Compute the contribution function $f(\bar{\mathbf{x}}_k)$:
    $$f(\bar{\mathbf{x}}_k) = W_e(\mathbf{x}_0, \mathbf{x}_1) \eta^2(\mathbf{x}_n) L_e(\mathbf{x}_n, \mathbf{x}_{n-1}) G(\mathbf{x}_0, \mathbf{x}_1) T(\mathbf{x}_0, \mathbf{x}_1) \prod_{i=1}^{n-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) T(\mathbf{x}_i, \mathbf{x}_{i+1}) f_c(\mathbf{x}_{i-1}, \mathbf{x}_i, \mathbf{x}_{i+1})$$
    Where:
    *   $W_e$ is the exitance weight [S1].
    *   $\eta(\mathbf{x}_n)$ is the refractive index at the source [S1].
    *   $L_e$ is the emitted radiance [S1].
    *   $f_c$ is the phase function (implied by context of scattering, though not explicitly defined in the snippet, it is part of the product term) [S1].
    *   Note: The source formula includes $G(\mathbf{x}_0, \mathbf{x}_1)$ and $T(\mathbf{x}_0, \mathbf{x}_1)$ in the product, but $\mathbf{x}_0$ is the camera. The product index $i$ runs from $1$ to $n-1$. The term $G(\mathbf{x}_0, \mathbf{x}_1)$ appears outside the product in the source formula [S1].

7.  **Final Estimator:** The total radiance $I$ is estimated by summing over $K$ samples:
    $$I \simeq \sum_{k=1}^K \frac{f(\bar{\mathbf{x}}_k)}{p(\bar{\mathbf{x}}_k)}$$
    [S1]

**Parameters:**
*   $\epsilon$: Regularization constant for $G$. Typical values: $10^{-2}$ to $10^{-3}$ [S1].

**Pitfalls:**
*   The computation of the generalized geometric factor $G$ can be unstable, leading to high variance. Regularization is necessary [S1].
*   The initialization strategy of the BFGS solver does not guarantee finding all possible paths in regions with strong RIF variations, potentially resulting in local underestimation of transmitted lighting [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method is used in conjunction with BFGS optimization. The text mentions 1M samples per pixel at 1024x1024 resolution for the final images, but this refers to the overall rendering pipeline, not just this estimator [S1].
*   **Strengths:** Provides an unbiased estimate of the contribution of stationary paths. Regularization helps eliminate highly contributing outliers [S1].
*   **Limitations:** Can locally underestimate the brightest parts of the transmitted caustic due to regularization. May miss paths in regions with strong RIF variations [S1].

## Background
This method is part of a pipeline for rendering old glass windows, where ancient windows were composed of several smaller fragments of glass assembled with lead [S1]. The method builds on the path formulation of the RRTE [S1].

## Sources
[S1] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
