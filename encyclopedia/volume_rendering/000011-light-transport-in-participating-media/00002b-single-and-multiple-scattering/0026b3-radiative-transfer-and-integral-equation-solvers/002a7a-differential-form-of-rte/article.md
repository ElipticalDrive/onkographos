# Differential Form of RTE

Also called: Analytical Differential Expression for Transmittance, Reynolds Transport Theorem for Single Scattering

## Overview
The Differential Form of RTE provides an analytical method to compute the gradient of the Radiative Transfer Equation (RTE) with respect to medium density, enabling differentiable rendering for fluid modeling [S1]. It sets itself apart from numerical differentiation or automatic differentiation by deriving closed-form expressions for the derivatives of transmittance and single-scattering radiance directly from the continuous integral formulation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Density field $\rho$, extinction coefficient $\kappa_t$, reduced incident radiance $L_{ri}$ (or $J_{ss}$), ray direction $\boldsymbol{\omega}$, and sampling step size $\Delta d$ [S1].
*   **Outputs:** The derivative of the single-scattering radiance $\dot{L}_{ss}$ and the derivative of the reduced incident radiance $\dot{L}_{ri}$ with respect to density [S1].

**Data Structures:**
*   **Voxel Grid:** Stores density $\rho$ and derived coefficients $\kappa_t$ [S1].
*   **Ray:** Defined by origin, direction, and current position $v_p$ in the voxel grid [G].

**Formulas:**
1.  **Transmittance:** The transmittance $\tau(x_0, x_1)$ from position $x_0$ to $x_1$ is defined as [S1]:
    $$ \tau(x_0, x_1) = e^{-\int_{D_1}^{D_0} \kappa_t(x_u) du} $$
2.  **Transmittance Derivative:** The derivative of transmittance $\tau$ from $v_{x_0}$ to $v_{x_1}$ is the negative product of $\tau$ and the sum of extinction coefficient derivatives $\dot{\kappa}_t$ times step size $\Delta d$ [S1]:
    $$ \dot{\tau}(x_0, x_1) = -\tau(x_0, x_1) \sum \dot{\kappa}_t \Delta d $$
    where $\dot{\kappa}_t$ is the derivative of the extinction coefficient, equal to the extinction cross-section $\sigma_t$, and $\Delta d$ is the length of the sampling step in the voxel grid [S1].
3.  **Single-Scattering Radiance:** The single-scattering radiance is defined as [S1]:
    $$ L_{ss}(x, \boldsymbol{\omega}) = \int_{D_b}^{D_a} \tau(x_u, x_b) \kappa_t(x_u) J_{ss}(x_u, \boldsymbol{\omega}) du $$
    where $J_{ss}$ is the reduced incident radiance [S1].

**Steps:**
1.  Compute density derivatives directly using the analytical differential expression instead of automatic differentiation [S1].
2.  Calculate the transmittance derivative $\dot{\tau}$ along the ray segment using the sum of $\dot{\kappa}_t \Delta d$ [S1].
3.  Apply the Reynolds transport theorem to derive $\dot{L}_{ss}$ from the integral form of $L_{ss}$, assuming continuity of $L_{ri}$ in $S^2$ [S1].
4.  Combine $\dot{L}_{ss}$ and $\dot{L}_{ri}$ to obtain the total gradient for the differentiable rendering process [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_differential_rte_derivative(tau, kappa_t, sigma_t, step_size, j_ss):
    # tau: transmittance τ(x_0, x_1)
    # kappa_t: extinction coefficient κ_t
    # sigma_t: extinction cross-section σ_t (derivative of κ_t)
    # step_size: sampling step Δd
    # j_ss: reduced incident radiance J_ss
    # Computes derivative of single-scattering radiance with respect to density
    dot_kappa_t = sigma_t  # κ˙_t is the derivative of the extinction coefficient, equal to σ_t [S1]
    dot_tau = -tau * dot_kappa_t * step_size  # Transmittance derivative [S1]
    # Derivative of L_ss = ∫ τ κ_t J_ss du
    # Using Reynolds transport theorem assuming continuity of L_ri in S^2 [S1]
    d_L_ss = dot_tau * kappa_t * j_ss
    return d_L_ss
```

**Edge Cases and Pitfalls:**
*   **Continuity Assumption:** The method calculates the derivative of $J_{ss}$ using the Reynolds transport theorem, which assumes continuity of $L_{ri}$ in $S^2$ [S1]. Discontinuities in incident radiance may require special handling or break the analytical gradient [G].
*   **Discrete Approximation:** The continuous integral for $L_{ss}$ and $\tau$ is discretized for implementation; accuracy depends on the voxel grid resolution and step size $\Delta d$ [S1].

## Subtopics

### [Forward Scattering Dipole Model](10173-forward-scattering-dipole-model.md)
An analytical subsurface scattering model derived from the functional integral formulation of radiative transport, linked to the partition function of a worm-like chain in polymer physics. It explicitly assumes strong forward scattering to avoid errors from isotropic diffusion assumptions in optically dense media.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampling steps along the ray, as it involves a summation over $\dot{\kappa}_t \Delta d$ [S1].
*   **Memory Complexity:** Constant, requiring storage for current radiance, density, and coefficients [S1].
*   **Strengths:** Provides exact gradients for density, enabling efficient optimization in differentiable rendering [S1].
*   **Limitations:** Derived primarily for single-scattering; multiple scattering requires additional approximations or integration [S1].
*   **When to Use:** When differentiable rendering of participating media is required, specifically for fluid modeling from monocular video [S1].

## Background
The method originates from the paper "Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video" [S1]. It is used in the context of differentiable rendering to estimate density and velocity fields from video data by minimizing the difference between rendered and observed images [S1].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
