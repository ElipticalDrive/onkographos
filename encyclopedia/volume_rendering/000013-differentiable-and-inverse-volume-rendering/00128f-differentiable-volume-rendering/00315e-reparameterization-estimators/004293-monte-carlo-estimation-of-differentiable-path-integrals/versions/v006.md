# Monte Carlo Estimation of Differentiable Path Integrals

Also called: Differentiable Path Tracing, Gradient Estimation via Path Integral

## Overview
This method estimates the gradient of a path integral rendering equation with respect to scene parameters $\pi$ by decomposing the derivative into interior contributions (derivative of the path contribution function) and boundary contributions (derivative of the visibility boundary) [S1]. It differs from [Reparameterization Estimators](6499-reparameterization-estimators.md) by explicitly handling the boundary terms arising from the Reynolds Transport Theorem rather than solely relying on domain transformation to eliminate them, and from [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) by providing the specific Monte Carlo sampling strategies for these boundaries in complex translucent materials [S1].

## How it works
**Inputs:** Scene parameters $\pi$ (e.g., geometry, material properties), path contribution function $\hat{f}(p)$, probability density functions (PDFs) for sampling $p(y)$.
**Outputs:** Unbiased estimate of the gradient $\frac{\partial I}{\partial \pi}$.

**Data Structures:**
*   Path $p$: Sequence of vertices $p_0, p_1, \dots, p_k$. $p_0$ is the light source, $p_k$ is the camera.
*   Sample positions $y_i$: Random variables used to generate vertices $p_i$.
*   PDFs $p(y_i)$: Probability densities associated with the sampling of $y_i$.

**Steps:**
1.  **Decompose the Gradient:** The total gradient is split into an interior term (derivative of the integrand) and boundary terms (derivative of the integration domain limits) [S1].
    $$
    \frac{\partial I}{\partial \pi} = \int \frac{\partial \hat{f}}{\partial \pi} d\mu + \text{Boundary Terms}
    $$
2.  **Estimate Interior Term:** Sample paths using conventional path tracing strategies. For each path, compute the partial derivative of the path contribution function $\hat{f}$ with respect to $\pi$ [S1].
3.  **Estimate Boundary Terms:** Sample points on discontinuity boundaries. Two types are identified:
    *   **Primary Visibility Boundary:** Corresponds to silhouette edges in the image.
    *   **Secondary Visibility Boundary:** Corresponds to shadow edges cast on the surface. For translucent materials (BSSRDF), this discontinuity is blurred by subsurface scattering, and its motion affects outgoing radiance at surface points near the edge [S1].
4.  **Sample Secondary Boundaries:** To sample shadow edges:
    *   Sample points on triangle edges.
    *   Connect these points to the point light source.
    *   Trace rays from the light connection back against the surface to find intersection points. These intersections lie on the shadow edges [S1].
5.  **Aggregate:** Sum the weighted contributions of interior and boundary samples to estimate the gradient.

**Formulas:**
The Monte Carlo estimator for the gradient of a 3-segment path (light $p_0$, surface $p_1, p_2$, camera $p_3$) is:
$$
\langle I_3 \rangle = \frac{1}{N} \sum_{i=1}^N \frac{\hat{f}(p_0 p_1 p_2 p_3)}{p(y_1)p(y_2)}
$$
where $y_1, y_2$ are the sample positions generating $p_1, p_2$ [S1].

The estimator for a 2-segment path (light $p_0$, surface $p_1$, camera $p_2$) is:
$$
\langle I_2 \rangle = \frac{1}{N} \sum_{i=1}^N \frac{\hat{f}(p_0 p_1 p_2)}{p(y_1)}
$$
where $y_1$ is the sample position generating $p_1$ [S1].

The interior term for the 3-segment path is estimated by:
$$
\frac{\partial I_3}{\partial \pi} \approx \frac{1}{N} \sum_{i=1}^N \frac{\partial \hat{f}(p_0 p_1 p_2 p_3) / \partial \pi}{p(y_1)p(y_2)}
$$
where the path is sampled as in conventional path tracing [S1].

**Parameters:**
*   $N$: Number of samples. Used in Monte Carlo summation for unbiased estimation [S1].

**Pitfalls:**
*   **Biased Gradients from Naive Loss:** Minimizing the loss of an unbiased rendering estimate produces biased gradients. Specifically, minimizing $\langle (\tilde{I} - I_{ref})^2 \rangle$ minimizes the variance of the estimator, not the true error between the expected rendering and the reference [S1]. The correct loss is $(\langle \tilde{I} \rangle - I_{ref})^2$.

## Operating characteristics
*   **Complexity:** Linear in the number of samples $N$ and path length. Boundary sampling adds overhead proportional to the number of discontinuity boundaries (silhouettes and shadow edges) [S1].
*   **Strengths:** Handles complex translucent materials (BSSRDF) where shadow edges are blurred, which opaque surface methods fail to capture robustly [S1].
*   **Limitations:** Requires explicit sampling of discontinuity boundaries, which can be complex to implement for arbitrary scene geometries. The naive application of loss functions leads to biased optimization [S1].

## Background
Originated in the context of differentiable rendering for subsurface scattering materials (BSSRDF) [S1]. The method addresses the challenge of gradient estimation when visibility boundaries move with respect to scene parameters, particularly in translucent objects where shadow edges are not sharp [S1].

## Sources
[S1] Reconstructing Translucent Objects using Differentiable Rendering, https://doi.org/10.1145/3528233.3530714, fetched 2026-11-03
