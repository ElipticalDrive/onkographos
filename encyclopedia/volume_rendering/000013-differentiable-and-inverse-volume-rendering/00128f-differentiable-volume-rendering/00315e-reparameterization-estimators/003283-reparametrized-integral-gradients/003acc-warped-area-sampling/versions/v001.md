# Warped-Area Sampling

Also called: Warped-area reparameterization

## Overview
Warped-Area Sampling (WAS) is an unbiased method for differentiable rendering that eliminates the need for explicit boundary sampling by reparameterizing boundary integrals into interior integrals using a continuous warp field [S1, unofficial]. It builds on [Reparametrized Integral Gradients](6785-reparametrized-integral-gradients.md) but improves upon the biased spherical rotation approach of Loubet et al. by satisfying the strict continuity requirements of the divergence theorem [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene parameters $\pi$, primal ray direction $\omega$, and the discontinuity boundary $\Delta S^2$ in the spherical domain $S^2$.
*   **Output:** An unbiased estimator for the gradient $\frac{\partial I}{\partial \pi}$ via a single interior integral over $S^2$.

**Data Structures:**
*   **Warp Field $\mathbf{V}$:** A vector field defined on $S^2$. It must be continuous everywhere and converge to the boundary velocity $\mathbf{v}_{\partial}$ as $\omega$ approaches $\Delta S^2$.
*   **Discontinuous Warp Field $\mathbf{V}_{dis}$:** An intermediate field used to construct $\mathbf{V}$. It is discontinuous at $\Delta S^2$.
*   **Smoothing Kernel $w(\omega, \omega')$:** A convolution kernel used to smooth $\mathbf{V}_{dis}$ into $\mathbf{V}$.

**Steps:**
1.  **Construct $\mathbf{V}_{dis}$:** Cast auxiliary rays around the primal sampled ray. For each auxiliary ray direction $\omega_p$, if it intersects an object dependent on $\pi$, compute the velocity $\mathbf{v}_p := \partial \omega_p / \partial \pi$ by differentiating the ray intersection process [S1, unofficial]. This ensures that as $\omega_p$ approaches the boundary $\Delta S^2$, $\mathbf{v}_p$ converges to $\mathbf{v}_{\partial}$ [S1, unofficial].
2.  **Smooth $\mathbf{V}_{dis}$:** Convolve $\mathbf{V}_{dis}$ with the kernel $w$ to produce the continuous warp field $\mathbf{V}$. The kernel is defined as:
    $$ \frac{1}{w(\omega, \omega')} := \frac{1}{D(\omega, \omega') + B(\omega')} $$
    where $D(\omega, \omega')$ is a distance function and $B(\omega')$ is a boundary-test function that converges to 0 when $\omega' \to \Delta S^2$ [S1, unofficial].
3.  **Estimate Gradient:** Apply the divergence theorem to reparameterize the boundary integral into an interior integral:
    $$ \int_{\partial S^2} (\Delta f \mathbf{v}) \cdot \mathbf{n} d\ell(\omega_i) = \int_{S^2} (\nabla \cdot \Delta f \mathbf{V}) d\sigma(\omega_i) $$
    The total gradient estimator is the reparameterized spherical integral:
    $$ \frac{\partial I}{\partial \pi} = \int_{S^2} \left[ \frac{\partial f}{\partial \pi} + (\nabla \cdot \Delta f \mathbf{V}) \right] d\sigma(\omega_i) $$
    where $f := L_i f_s$ is the integrand [S1, unofficial].

**Parameters:**
*   **Warp Field Requirements:** (a) $\mathbf{V}$ must be continuous; (b) $\mathbf{V}(\omega)$ must converge to $\mathbf{v}_{\partial}$ as $\omega \to \Delta S^2$ [S1, unofficial].
*   **Kernel Parameters:** $D$ and $B$ are not given in the sources.

**Pitfalls:**
*   Using only one auxiliary ray to approximate boundary motion (as in Loubet et al.) results in a biased estimator because the resulting warp field does not strictly satisfy the boundary requirement [S1, unofficial].
*   Precision issues may arise if the smoothing kernel does not preserve the boundary velocity $\mathbf{v}_{\partial}$ correctly.

## Subtopics

### Warped-Area Reparameterization for Differential Path Integrals
An extension of WAS to differential path integrals by Xu et al. [XBLZ23], enabling multi-bounce light transport estimation without explicit boundary sampling in path space [S1, unofficial]. It decomposes the boundary integral from PSDR into a sum of integrals over sub-spaces $\partial \hat{\Omega}_{N,K}$:
$$ \int_{\partial \hat{\Omega}} \Delta \hat{f}_K(\bar{p}) \mathbf{V}_K(p_K) d\mu'(\bar{p}) = \sum_{N=1}^{\infty} \sum_{K=0}^{N-1} \int_{\partial \hat{\Omega}_{N,K}} \Delta \hat{f}_K(\bar{p}) \mathbf{V}_K(p_K) d\mu'(\bar{p}) $$
The reparameterized differential path integral is:
$$ \frac{\partial I}{\partial \pi} = \int_{\hat{\Omega}} \left[ \frac{\partial \hat{f}(\bar{p})}{\partial \pi} + \sum_{K=0}^{N-1} \nabla \cdot \Delta \hat{f}_K \mathbf{V}_K(p_K) \right] d\mu(\bar{p}) $$
Compared to the reparameterized spherical integral, this path integral has a more generalized form depicting multi-bounce light transport, enabling advanced Monte Carlo strategies like bidirectional path tracing [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Unbiased estimation; avoids explicit boundary sampling; enables multi-bounce transport estimation (via the path integral variant).
*   **Limitations:** Requires construction of a continuous warp field; the auxiliary ray approximation approach is biased if not handled correctly.
*   **When to use:** When unbiased differentiable rendering is required and explicit boundary sampling is difficult or biased.
*   **When not to use:** When biased estimators are acceptable and simpler methods (like spherical rotation) suffice.

## Background
Warped-Area Sampling was proposed by Bangaru et al. [BLD20] as a refinement to the biased spherical rotation approach by Loubet et al. [LHJ19]. It draws inspiration from the divergence theorem to reparameterize boundary integrals into interior integrals [S1, unofficial]. The method was later extended to differential path integrals by Xu et al. [XBLZ23] to support multi-bounce light transport [S1, unofficial].

## Sources
[S1] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
