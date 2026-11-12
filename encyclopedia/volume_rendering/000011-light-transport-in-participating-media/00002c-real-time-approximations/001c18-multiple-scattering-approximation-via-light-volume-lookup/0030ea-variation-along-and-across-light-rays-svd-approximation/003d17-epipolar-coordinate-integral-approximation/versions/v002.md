# Epipolar Coordinate Integral Approximation

Also called: SVD-based integrand approximation

## Overview
This method approximates the multiple scattering integral in volumetric shadowing by decomposing the view-independent part of the integrand using Singular Value Decomposition (SVD) in epipolar coordinates [S1]. It reduces the computational cost from per-pixel to per-slice by processing each camera ray independently, which enables massive parallelism compared to methods that maintain interdependent partial sum trees across rays [S1]. It builds on [Variation Along and Across Light Rays (SVD Approximation)](6391-variation-along-and-across-light-rays-svd-approximation.md) [G].

## How it works
**Inputs:**
- Camera depth map (unrectified) [S1].
- Light depth map (rectified into epipolar coordinates, stored as shadow map $S$) [S1].
- Volume density $\sigma_t$, scattering coefficient $\sigma_s$, phase function $\rho$, and incoming light $L_{\text{in}}$ [S1].
- Number of singular vectors $N$ (typically 4) [S1].

**Outputs:**
- Radiance $L(\alpha, \beta)$ for each camera pixel $(\alpha, \beta)$ [S1].

**Data Structures:**
- **Rectified Shadow Map $S$**: A 2D array where $S[\alpha, \gamma]$ stores the $\beta$ coordinate (height) of the camera ray at which the light ray $(\alpha, \gamma)$ is blocked [S1].
- **Singular Value Decomposition Components**: Left singular vectors $B_i(\beta)$ and right singular vectors $\Gamma_i(\gamma)$, precomputed once per frame on the CPU [S1].
- **Prefix Sum Tables**: For each singular vector $i \in [1, N]$, a 1D prefix sum table of $\Gamma_i(\gamma) \Delta \gamma$ is computed [S1].

**Steps:**

1. **Change of Variables**: Rewrite the scattering integral in epipolar coordinates $(\alpha, \beta, \gamma)$ [S1]. The radiance $L(\alpha, \beta)$ for a camera ray defined by $(\alpha, \beta)$ is:
   $$L(\alpha, \beta) = \int_0^{D(\alpha, \beta)} e^{-\sigma_t s(\beta, \gamma)} V(\alpha, \beta, \gamma) \sigma_s \rho(\beta, \gamma) L_{\text{in}}(\beta, \gamma) \frac{ds}{d\gamma} d\gamma$$
   where $D(\alpha, \beta)$ is the $\gamma$ coordinate of the light ray at which the camera ray $(\alpha, \beta)$ is blocked [S1].

2. **Separate Visibility**: Define the view-independent integrand $\mathcal{I}(\beta, \gamma)$ containing all terms except visibility $V$ [S1]:
   $$\mathcal{I}(\beta, \gamma) = e^{-\sigma_t s(\beta, \gamma)} \sigma_s \rho(\beta, \gamma) L_{\text{in}}(\beta, \gamma) \frac{ds}{d\gamma}$$
   The integral becomes:
   $$L(\alpha, \beta) = \int_0^{D(\alpha, \beta)} V(\alpha, \beta, \gamma) \mathcal{I}(\beta, \gamma) d\gamma$$

3. **SVD Approximation**: Sample $\mathcal{I}(\beta, \gamma)$ on a $64 \times 64$ grid [S1]. Compute the SVD of this sampled matrix on the CPU [S1]. Select the top $N$ singular values and their corresponding left singular vectors $B_i(\beta)$ and right singular vectors $\Gamma_i(\gamma)$ [S1]. Approximate the integrand as:
   $$\mathcal{I}(\beta, \gamma) \approx \sum_{i=1}^N B_i(\beta) \Gamma_i(\gamma)$$
   Substitute this into the integral:
   $$L(\alpha, \beta) \approx \sum_{i=1}^N \left( B_i(\beta) \int_0^{D(\alpha, \beta)} V(\alpha, \beta, \gamma) \Gamma_i(\gamma) d\gamma \right)$$

4. **Discretization and Visibility**: Approximate the inner integral as a Riemann sum [S1]. Visibility $V(\alpha, \beta, \gamma)$ is 1 if the point is lit, 0 otherwise [S1]. In epipolar coordinates, a point $(\alpha, \beta, \gamma)$ is lit if $\gamma < D(\alpha, \beta)$ and $S[\alpha, \gamma] > \beta$ [S1]. The radiance is:
   $$L(\alpha, \beta) \approx \sum_{i=1}^N \left( B_i(\beta) \sum_{\substack{\gamma < D(\alpha, \beta) \\ S[\alpha, \gamma] > \beta}} \Gamma_i(\gamma) \Delta \gamma \right)$$

5. **Parallel Evaluation**: For each camera ray $(\alpha, \beta)$, identify the lit segments along $\gamma$ where $S[\alpha, \gamma] > \beta$ [S1]. Use 1D min-max mipmaps to accelerate finding these segments [S1]. Compute the sum over each segment using the precomputed prefix sum tables for $\Gamma_i(\gamma) \Delta \gamma$ [S1]. Multiply by $B_i(\beta)$ and sum over $i=1 \dots N$ [S1].

**Parameters:**
- $N$: Number of singular vectors. Typical value: 4 [S1].

**Edge Cases and Pitfalls:**
- The camera depth map does not need to be rectified, simplifying the pipeline [S1].
- Precision issues may arise if the SVD approximation error is high for complex lighting; however, smooth variation of $\mathcal{I}$ usually allows small $N$ to suffice [S1].

## Operating characteristics
- **Time Complexity**: Reduces computational cost from per-pixel to per-slice [S1]. Instead of maintaining a partial sum tree for each $\beta$ (as in incremental integration), it computes only $N$ prefix sum tables per frame [S1]. Each ray is processed independently, enabling massive parallelism on the GPU [S1].
- **Memory Complexity**: Requires storage for the rectified shadow map, $N$ prefix sum tables, and the SVD components [S1]. The SVD computation is done on the CPU once per frame [S1].
- **Strengths**: Massive parallelism due to independent ray processing; reduced computational overhead compared to incremental integration [S1].
- **Limitations**: Approximation error depends on the number of singular vectors $N$ and the smoothness of the integrand [S1].

## Background
This method is part of the "Real-Time Volumetric Shadows using 1D Min-Max Mipmaps" technique, which uses epipolar coordinates and SVD to approximate multiple scattering in participating media [S1]. It was introduced by [S1] to address the performance bottlenecks of previous real-time volumetric shadowing methods.

## Sources
[S1] Real-Time Volumetric Shadows using 1D Min-Max Mipmaps, https://groups.csail.mit.edu/graphics/mmvs/mmvs.pdf, fetched 2026-10-28
