# Differential Form of RTE

Also called: Analytical Differential Expression for Transmittance, Reynolds Transport Theorem for Single Scattering

## Overview
The Differential Form of RTE provides an analytical method to compute the gradient of the Radiative Transfer Equation (RTE) with respect to medium parameters, specifically density, enabling differentiable rendering for fluid modeling [S1]. It sets itself apart from numerical differentiation or automatic differentiation by deriving closed-form expressions for the derivatives of transmittance and single-scattering radiance, avoiding the accumulation of errors from discrete sampling steps [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Density field $\rho$, extinction coefficient $\kappa_t$ (or $\sigma_t$), incident radiance $L_{ri}$, ray direction $\boldsymbol{\omega}$, and sampling step size $\Delta d$.
*   **Outputs:** The derivative of the single-scattering radiance $\dot{L}_{ss}$ and the derivative of the reduced incident radiance $\dot{L}_{ri}$ with respect to density.

**Data Structures:**
*   **Voxel Grid:** Stores density $\rho$ and derived coefficients $\kappa_t$.
*   **Ray:** Defined by origin, direction, and current position $v_p$ in the voxel grid.

**Formulas:**
1.  **Transmittance Derivative:** The derivative of transmittance $\tau(x_0, x_1)$ from position $x_0$ to $x_1$ is given by [S1]:
    $$ \dot{\tau}(x_0, x_1) = -\tau(x_0, x_1) \sum_{i} \dot{\kappa}_t(x_i) \Delta d $$
    where $\dot{\kappa}_t$ is the derivative of the extinction coefficient (equal to the extinction cross-section $\sigma_t$) and $\Delta d$ is the step size [S1].

2.  **Single-Scattering Radiance Derivative:** The derivative of the single-scattering term $L_{ss}$ is computed using the Reynolds transport theorem, assuming continuity of the reduced incident radiance $L_{ri}$ on the sphere $S^2$ [S1]. The single-scattering radiance is defined as [S1]:
    $$ L_{ss}(x, \boldsymbol{\omega}) = \int_{D_b}^{D_a} \tau(x_u, x_b) \kappa_t(x_u) J_{ss}(x_u, \boldsymbol{\omega}) du $$
    where $J_{ss}$ is the reduced incident radiance [S1].

**Steps:**
1.  Compute the extinction coefficient derivative $\dot{\kappa}_t$ directly from the density derivatives [S1].
2.  Calculate the transmittance derivative $\dot{\tau}$ along the ray segment using the sum of $\dot{\kappa}_t \Delta d$ [S1].
3.  Apply the Reynolds transport theorem to derive $\dot{L}_{ss}$ from the integral form of $L_{ss}$ [S1].
4.  Combine $\dot{L}_{ss}$ and $\dot{L}_{ri}$ to obtain the total gradient for the differentiable rendering process [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_differential_rte(radiance, density, extinction_coeff, step_size):
    """
    Computes the derivative of single-scattering radiance using the differential form of RTE.
    
    Args:
        radiance (float): Current radiance L_ss.
        density (float): Current density rho.
        extinction_coeff (float): Extinction coefficient kappa_t.
        step_size (float): Sampling step size Delta_d.
        
    Returns:
        float: Derivative of radiance dL_ss/d_rho.
    """
    # Compute derivative of extinction coefficient (sigma_t)
    # Note: In the source, kappa_t derivative is equal to sigma_t.
    # Assuming density derivative is available or computed externally.
    d_extinction = compute_extinction_derivative(density) 
    
    # Compute transmittance derivative
    # tau_dot = -tau * sum(kappa_dot * Delta_d)
    # For a single step:
    transmittance = compute_transmittance(extinction_coeff, step_size)
    transmittance_derivative = -transmittance * d_extinction * step_size
    
    # Compute single-scattering radiance derivative
    # Using Reynolds transport theorem for J_ss
    # L_ss = integral(tau * kappa_t * J_ss)
    # The derivative involves the derivative of tau and kappa_t
    reduced_incident_radiance = compute_reduced_incident_radiance()
    
    # dL_ss/d_rho = integral( tau_dot * kappa_t * J_ss + tau * kappa_t_dot * J_ss + tau * kappa_t * J_ss_dot )
    # Simplified for single step with J_ss continuity assumption:
    d_radiance = transmittance_derivative * extinction_coeff * reduced_incident_radiance
    
    return d_radiance
```

**Edge Cases and Pitfalls:**
*   **Discontinuity:** The method assumes continuity of $L_{ri}$ in $S^2$; discontinuities may require special handling [S1].
*   **Precision:** Numerical stability of the exponential in transmittance calculation should be considered [S1].

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
