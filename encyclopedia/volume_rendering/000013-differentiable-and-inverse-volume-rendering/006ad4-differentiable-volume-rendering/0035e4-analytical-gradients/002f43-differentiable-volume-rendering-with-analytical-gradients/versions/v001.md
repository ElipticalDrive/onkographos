# Differentiable Volume Rendering with Analytical Gradients

Also called: Analytical Differentiable Rendering, DiffRenderer

## Overview
A rendering technique that computes radiance and its analytical derivatives with respect to volume density directly, avoiding the overhead of propagating gradients through long computational graphs inherent in automatic differentiation [S1]. It sets itself apart from siblings like [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) by deriving an explicit analytical form of the radiance transfer equation for density optimization, rather than relying on basis function interpolation or mesh-based projection derivatives [S1].

## How it works
**Inputs:** Ray origin $\mathbf{x}_a$, ray direction $\boldsymbol{\omega}$, volume density field $\sigma(\mathbf{x})$, and extinction coefficient $\kappa_t$.
**Outputs:** Radiance $L(\mathbf{x}_a, \boldsymbol{\omega})$ and its gradient $\nabla_{\sigma} L$.

**Data Structures:**
*   Ray state: Current position $\mathbf{x}_u$, accumulated transmittance $\tau$, accumulated radiance $L_{ss}$.
*   Density field: Discretized volume or continuous function allowing evaluation $\sigma(\mathbf{x})$ and derivative $\nabla_{\sigma} \sigma(\mathbf{x})$ (typically identity or interpolation Jacobian).

**Steps:**
1.  Initialize accumulated radiance $L_{ss} \leftarrow 0$ and transmittance $\tau \leftarrow 1$ [S1].
2.  Set current ray position $\mathbf{x}_u \leftarrow \mathbf{x}_a$ [S1].
3.  March along the ray from $\mathbf{x}_a$ to $\mathbf{x}_b$:
    a.  Advance ray position: $\mathbf{x}_u \leftarrow \mathbf{x}_u + \boldsymbol{\omega} \Delta d$ [S1].
    b.  Evaluate density $\sigma_u = \sigma(\mathbf{x}_u)$ and its derivative with respect to the optimization parameter (density itself) $\frac{\partial \sigma_u}{\partial \sigma}$.
    c.  Compute local transmittance $T_u = \exp(-\kappa_t \Delta d)$ [G].
    d.  Update global transmittance: $\tau \leftarrow \tau \cdot T_u$.
    e.  Compute emitted radiance contribution: $J_{ss}(\mathbf{x}_u, \boldsymbol{\omega}) = \sigma_u \kappa_t \tau$ [S1, derived from Eq 5 context].
    f.  Update accumulated radiance: $L_{ss} \leftarrow L_{ss} + J_{ss}(\mathbf{x}_u, \boldsymbol{\omega})$ [S1].
    g.  Compute analytical gradient contribution. Since $L$ is a sum of terms dependent on $\sigma$, and $\tau$ depends on previous $\sigma$, the gradient $\frac{\partial L}{\partial \sigma_k}$ for a specific sample $k$ involves the product of transmittance up to $k$ and the emission at $k$, plus the derivative of subsequent terms where $\sigma_k$ affects transmittance. Specifically, $\frac{\partial L}{\partial \sigma_k} = \Delta d \cdot \kappa_t \cdot \tau_{k-1} \cdot \exp(-\kappa_t \Delta d) \cdot (1 - \sum_{j=k+1}^{N} \dots)$ [G, standard analytical volume gradient].
4.  Return $L_{ss}$ and $\nabla_{\sigma} L_{ss}$.

**Parameters:**
*   $\Delta d$: Step size. Not given in the sources.
*   $\kappa_t$: Extinction coefficient. Not given in the sources.

**Edge Cases:**
*   Singularities in density derivatives. The method requires a robust implementation to handle cases where density gradients are undefined or infinite [S1, via child link].
*   Precision loss in transmittance accumulation for long rays. Not given in the sources.

## Subtopics

### [Differentiable Moment Bounds](8855-differentiable-moment-bounds.md)
Computes continuously differentiable upper and lower bounds on cumulative mass using power moments. It provides tighter bounds than Chebyshev's inequality by generalizing to arbitrary moment sequences ($2n+1$ moments) and handles singularities for stable use in differentiable pipelines.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of ray steps $N$, $O(N)$, but with a significantly smaller constant factor than automatic differentiation (AD) because it avoids storing and traversing the computational graph [S1].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, as it does not store intermediate gradient states for backpropagation [S1].
*   **Performance:** Improves performance for optimizing dynamic volumetric densities, such as in fluid reconstruction tasks, by eliminating gradient tracking overhead [S1].
*   **Strengths:** Direct computation of derivatives; no redundant operations; efficient for large-scale density optimization.
*   **Limitations:** Requires the derivation of specific analytical forms for each rendering equation variant; less flexible than AD for arbitrary, non-differentiable operations.

## Background
Originating from the need to optimize dynamic volumetric densities (e.g., fluid reconstruction) efficiently, this method addresses the computational bottleneck of Automatic Differentiation (AD) in volumetric rendering [S1]. It builds on the insight that while AD works, it propagates gradients through long computational graphs, which is fundamentally slower than direct analytical differentiation [S1]. The approach derives an analytical form of the radiance transfer equation to enable direct and efficient computation of radiance derivatives with respect to density [S1].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
