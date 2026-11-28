# Unbiased Monte Carlo Estimator for Radiance and Radiance Derivatives

Also called: Joint Estimation of L(1) and L̂(1)

## Overview
This method provides an unbiased Monte Carlo estimator for the first-order radiance derivative $\hat{L}^{(1)}$ and the radiance $L^{(1)}$ in participating media, handling volumetric and interfacial scattering events separately based on the free-flight distance relative to the boundary distance $D$ [S1, unofficial]. It distinguishes itself from standard volumetric path tracing by explicitly propagating scene derivatives alongside radiance values, enabling the computation of gradients required for differentiable rendering [S1, unofficial]. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md), which computes only radiance, this estimator jointly tracks the sensitivity of the solution to scene parameters [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**:
    *   $\mathbf{x}, \hat{\mathbf{x}}$: Current position and its scene derivative [S1, unofficial].
    *   $\omega, \hat{\omega}$: Current direction and its scene derivative [S1, unofficial].
    *   $n$: Current path depth, used for Russian Roulette termination [S1, unofficial].
*   **Outputs**:
    *   $L^{(1)}$: The estimated radiance [S1, unofficial].
    *   $\hat{L}^{(1)}$: The estimated radiance derivative [S1, unofficial].

### Data Structures and Variables
*   $D$: Distance from $\mathbf{x}$ to the nearest boundary $\partial \Omega$ along $\omega$ [S1, unofficial].
*   $\hat{D}$: Scene derivative of $D$ [S1, unofficial].
*   $\tau$: Sampled free-flight distance [S1, unofficial].
*   $\sigma_s(\mathbf{x})$: Scattering coefficient [S1, unofficial].
*   $\sigma_t(\mathbf{x})$: Extinction coefficient [S1, unofficial].
*   $\hat{\sigma}_s(\mathbf{x})$: Scene derivative of the scattering coefficient [S1, unofficial].
*   $\Sigma_t(\mathbf{x}, \omega, \tau)$: Optical depth from $\mathbf{x}$ to $\mathbf{x} - \tau \omega$ [S1, unofficial].
*   $f_p(\mathbf{x}, \omega', \omega)$: Phase function [S1, unofficial].
*   $\hat{f}_p(\mathbf{x}, \omega', \omega)$: Scene derivative of the phase function [S1, unofficial].
*   $f_s(\mathbf{x}, \omega', \omega)$: BSDF at the boundary [S1, unofficial].
*   $\hat{f}_s(\mathbf{x}, \omega', \omega)$: Scene derivative of the BSDF [S1, unofficial].
*   $L^{(0)}(\mathbf{x}, \omega)$: Zero-order radiance (emission or direct illumination estimate) [S1, unofficial].

### Parameters
*   **Russian Roulette**: A survival probability is used to terminate paths stochastically to avoid infinite recursion during indirect term estimation [S1, unofficial].

### Algorithm Steps
1.  **Calculate Boundary Distance**: Compute $D$ and $\hat{D}$ based on $\mathbf{x}, \hat{\mathbf{x}}, \omega, \hat{\omega}$ [S1, unofficial].
2.  **Sample Free-Flight Distance**: Draw $\tau$ from the probability density $p(\tau) = \sigma_t(\mathbf{x}') T(\mathbf{x}', \mathbf{x})$, where $T$ is the transmittance [S1, unofficial].
3.  **Branch on Event Type**:
    *   **Volumetric Event ($\tau < D$)**:
        1.  Compute new position and derivative: $\mathbf{x}' = \mathbf{x} - \tau \omega$ and $\hat{\mathbf{x}}' = \hat{\mathbf{x}} - \tau \hat{\omega}$, ensuring $\mathbf{x}' \in \Omega \setminus \partial \Omega$ [S1, unofficial].
        2.  Compute scattering ratio $\alpha$ and derivative term $\beta$: $\alpha = \sigma_s(\mathbf{x}')/\sigma_t(\mathbf{x}')$ and $\beta = (\hat{\sigma}_s(\mathbf{x}') - \Sigma_t(\mathbf{x}, \omega, \tau)\sigma_s(\mathbf{x}'))/\sigma_t(\mathbf{x}')$ [S1, unofficial].
        3.  Draw new direction $\omega'$ from the phase function $f_p(\mathbf{x}', -\omega', \omega)$ [S1, unofficial].
        4.  **Recursive Call**: Call `radiance(x', x̂', ω', 0)` to obtain $(L^{(1)}, \hat{L}^{(1)})$ for the indirect term [S1, unofficial].
        5.  Initialize indirect radiance derivative $\hat{L}_{ins}$: $\hat{L}_{ins} = \hat{L}^{(1)} + (\hat{f}_p(\mathbf{x}', -\omega', \omega)/f_p(\mathbf{x}', -\omega', \omega))L^{(1)}$ [S1, unofficial].
        6.  Estimate direct volumetric term $L_{ins}$ by integrating the source term over the hemisphere $S^2$: $L_{ins} \leftarrow L_{ins} + \int_{S^2} f_p(\mathbf{x}', -\omega', \omega)L^{(0)}(\mathbf{x}', \omega') d\omega'$ [S1, unofficial].
        7.  Estimate direct derivative term $\hat{L}_{ins}$ by integrating the derivative of the source term: $\hat{L}_{ins} \leftarrow \hat{L}_{ins} + \int_{S^2} \partial_\pi[f_p(\mathbf{x}', -\omega', \omega)L^{(0)}(\mathbf{x}', \omega')] d\omega'$ [S1, unofficial].
        8.  Add the edge integral contribution to $\hat{L}_{ins}$ by calling `edgeIntegral(x', x̂', ω, fp)` [S1, unofficial].
        9.  Compute final volumetric estimates: $L^{(1)} = \alpha L_{ins}$ and $\hat{L}^{(1)} = \alpha \hat{L}_{ins} + \beta L_{ins}$ [S1, unofficial].
    *   **Interfacial Event ($\tau \ge D$)**:
        1.  Compute intersection point and derivative: $\mathbf{x}_0 = \mathbf{x} - D \omega$ and $\hat{\mathbf{x}}_0 = \hat{\mathbf{x}} - D \hat{\omega}$, ensuring $\mathbf{x}_0 \in \partial \Omega$ [S1, unofficial].
        2.  Draw new direction $\omega'$ with probability $p(\omega')$ from the local BSDF [S1, unofficial].
        3.  Compute BSDF ratio $\alpha$ and derivative term $\beta$: $\alpha = f_s(\mathbf{x}_0, \omega', \omega)/p(\omega')$ and $\beta = \Sigma_t(\mathbf{x}, \omega, D) + \hat{D}\sigma_t(\mathbf{x}_0)$ [S1, unofficial].
        4.  **Recursive Call**: Call `radiance(x0, x̂0, ω', 0)` to obtain $(L^{(1)}, \hat{L}^{(1)})$ for the reflected/scattered radiance [S1, unofficial].
        5.  Compute reflected radiance $L_{rs}$ and its derivative $\hat{L}_{rs}$: $L_{rs} = \alpha L^{(1)}$ and $\hat{L}_{rs} = \alpha \hat{L}^{(1)} + (\hat{f}_s(\mathbf{x}_0, -\omega', \omega)/p(\omega'))L^{(1)}$ [S1, unofficial].
        6.  Estimate direct interfacial term $L_{rs}$ by integrating the source term: $L_{rs} \leftarrow L_{rs} + \int_{S^2} f_s(\mathbf{x}_0, -\omega', \omega)L^{(0)}(\mathbf{x}_0, \omega') d\omega'$ [S1, unofficial].
        7.  Estimate direct derivative term $\hat{L}_{rs}$ by integrating the derivative of the source term: $\hat{L}_{rs} \leftarrow \hat{L}_{rs} + \int_{S^2} \partial_\pi[f_s(\mathbf{x}_0, -\omega', \omega)L^{(0)}(\mathbf{x}_0, \omega')] d\omega'$ [S1, unofficial].
        8.  Add the edge integral contribution to $\hat{L}_{rs}$ by calling `edgeIntegral(x0, x̂0, ω, fs)` [S1, unofficial].
        9.  Estimate the in-scattered radiance $L_{ins}$ at the boundary using the method from Appendix B [S1, unofficial].
        10. Compute final interfacial estimates: $L^{(1)} = L_{rs}$ and $\hat{L}^{(1)} = -\beta L_{rs} + \hat{L}_{rs} + \hat{D}\sigma_s(\mathbf{x}_0)L_{ins}$ [S1, unofficial].
4.  **Return**: Return $(L^{(1)}, \hat{L}^{(1)})$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def radiance(x, x_hat, omega, omega_hat, depth):
    """
    Computes unbiased Monte Carlo estimates for radiance L(1) and its derivative L_hat(1).
    """
    # 1. Calculate boundary distance D and its derivative D_hat
    D, D_hat = compute_boundary_distance(x, x_hat, omega, omega_hat) # [G]

    # 2. Sample free-flight distance tau from p(tau) = sigma_t(x') * T(x', x)
    tau = sample_free_flight(x, omega) # [G]

    # Russian roulette to avoid infinite recursion [S1, unofficial]
    if depth > 0 and random.random() > 0.5: # [G]
        return 0.0, 0.0

    if tau < D:
        # --- Volumetric Event ---
        x_prime = x - tau * omega
        x_hat_prime = x_hat - tau * omega_hat

        sigma_s_p = get_sigma_s(x_prime)
        sigma_t_p = get_sigma_t(x_prime)
        alpha = sigma_s_p / sigma_t_p

        sigma_t_x_tau = compute_optical_depth(x, omega, tau)
        sigma_s_hat_p = get_sigma_s_hat(x_prime)
        beta = (sigma_s_hat_p - sigma_t_x_tau * sigma_s_p) / sigma_t_p

        omega_prime = sample_phase_function(x_prime, -omega, omega)

        # Recursive call for indirect term
        L_ind, L_hat_ind = radiance(x_prime, x_hat_prime, omega_prime, 0.0, depth + 1)

        # Initialize indirect radiance derivative
        f_p = get_f_p(x_prime, -omega_prime, omega)
        f_p_hat = get_f_p_hat(x_prime, -omega_prime, omega)
        L_ins = L_ind
        L_hat_ins = L_hat_ind + (f_p_hat / f_p) * L_ind

        # Estimate direct volumetric term
        L_ins += integrate_hemisphere(lambda w: get_f_p(x_prime, -w, omega) * get_L0(x_prime, w))
        L_hat_ins += integrate_hemisphere(lambda w: derivative_pi(get_f_p(x_prime, -w, omega) * get_L0(x_prime, w)))

        # Add edge integral contribution
        L_hat_ins += edge_integral(x_prime, x_hat_prime, omega, get_f_p) # [S1, unofficial]

        # Final volumetric estimates
        L_out = alpha * L_ins
        L_hat_out = alpha * L_hat_ins + beta * L_ins
    else:
        # --- Interfacial Event ---
        x_0 = x - D * omega
        x_hat_0 = x_hat - D * omega_hat

        omega_prime = sample_bsdf(x_0, -omega, omega)
        p_omega = get_bsdf_prob(x_0, omega_prime, omega)

        f_s = get_f_s(x_0, omega_prime, omega)
        alpha = f_s / p_omega

        sigma_t_x_D = compute_optical_depth(x, omega, D)
        sigma_t_x0 = get_sigma_t(x_0)
        beta = sigma_t_x_D + D_hat * sigma_t_x0

        # Recursive call
        L_ind, L_hat_ind = radiance(x_0, x_hat_0, omega_prime, 0.0, depth + 1)

        # Compute reflected radiance and derivative
        L_rs = alpha * L_ind
        f_s_hat = get_f_s_hat(x_0, -omega_prime, omega)
        L_hat_rs = alpha * L_hat_ind + (f_s_hat / p_omega) * L_ind

        # Estimate direct interfacial term
        L_rs += integrate_hemisphere(lambda w: get_f_s(x_0, -w, omega) * get_L0(x_0, w))
        L_hat_rs += integrate_hemisphere(lambda w: derivative_pi(get_f_s(x_0, -w, omega) * get_L0(x_0, w)))

        # Add edge integral contribution
        L_hat_rs += edge_integral(x_0, x_hat_0, omega, get_f_s) # [S1, unofficial]

        # Estimate in-scattered radiance at boundary
        L_ins_boundary = in_scattered_radiance(x_0, omega) # [S1, unofficial]

        # Final interfacial estimates
        L_out = L_rs
        L_hat_out = -beta * L_rs + L_hat_rs + D_hat * get_sigma_s(x_0) * L_ins_boundary

    return L_out, L_hat_out

# Helper definitions (standard operations or comments per rules)
def compute_boundary_distance(x, x_hat, omega, omega_hat): # [G]
    # Computes distance to nearest boundary along omega and its scene derivative
    pass
def sample_free_flight(x, omega): # [G]
    # Samples tau from p(tau) = sigma_t(x') * T(x', x) using inverse CDF or delta tracking
    pass
def get_sigma_s(x): # [G]
    # Returns scattering coefficient at position x
    pass
def get_sigma_t(x): # [G]
    # Returns extinction coefficient at position x
    pass
def get_sigma_s_hat(x): # [G]
    # Returns scene derivative of scattering coefficient at position x
    pass
def compute_optical_depth(x, omega, tau): # [G]
    # Computes exp(-integral_0^tau sigma_t(x - s*omega) ds)
    pass
def sample_phase_function(x, omega_in, omega_out): # [G]
    # Draws omega' from fp(x, -omega', omega)
    pass
def get_f_p(x, omega_in, omega_out): # [G]
    # Returns phase function value
    pass
def get_f_p_hat(x, omega_in, omega_out): # [G]
    # Returns scene derivative of phase function
    pass
def integrate_hemisphere(func): # [G]
    # Monte Carlo integration over S^2
    pass
def derivative_pi(expr): # [G]
    # Computes partial derivative with respect to scene parameters
    pass
def edge_integral(x, x_hat, omega, phase_func): # [S1, unofficial]
    # Traces additional side paths to estimate edge integrals depending on delta L across boundaries
    pass
def sample_bsdf(x, omega_in, omega_out): # [G]
    # Draws omega' from local BSDF
    pass
def get_bsdf_prob(x, omega_prime, omega): # [G]
    # Returns probability density of the sampled BSDF direction
    pass
def get_f_s(x, omega_prime, omega): # [G]
    # Returns BSDF value
    pass
def get_f_s_hat(x, omega_prime, omega): # [G]
    # Returns scene derivative of BSDF
    pass
def get_L0(x, omega): # [G]
    # Returns zero-order radiance (emission or direct illumination)
    pass
def in_scattered_radiance(x, omega): # [S1, unofficial]
    # Estimates in-scattered radiance at boundary using method from Appendix B
    pass
```

### Edge Cases and Pitfalls
*   **Boundaries**: The condition $\tau < D$ strictly separates volumetric and interfacial events. If $\tau = D$, the interfacial case applies [S1, unofficial].
*   **Degenerate Input**: If the phase function or BSDF is zero, division by zero may occur in the derivative terms involving ratios like $\hat{f}/f$ or $\hat{f}_s/p(\omega')$. Guard clauses or alternative sampling strategies are required [G].
*   **Edge Integrals**: The `edgeIntegral` function requires tracing additional side paths and depends on the difference $\Delta L$ of radiance across boundary curves [S1, unofficial].

## Subtopics

### Multiple Scattering Volume Path Tracing with Phase Function Sampling
Uses phase function sampling for multiple scattering events, improving efficiency in highly scattering media compared to delta tracking [G]. [Link](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md)

### Volumetric Path Tracing with Spectral Integration
Extends the estimator to handle spectral radiance, allowing for color-accurate differentiable rendering [G]. [Link](2188-volumetric-path-tracing-with-spectral-integration.md)

### Volumetric Path Tracing Estimator Derivation
Provides the mathematical derivation of the radiance and derivative estimators, including the transport equation and adjoint formulations [G]. [Link](3830-volumetric-path-tracing-estimator-derivation.md)

### Volumetric Path Tracing with Transmittance Sampling
Uses transmittance sampling instead of delta tracking for more efficient free-flight distance sampling in heterogeneous media [G]. [Link](6593-volumetric-path-tracing-with-transmittance-sampling.md)

## Operating characteristics
*   **Time Complexity**: The method involves recursive calls and integrals over the hemisphere, making it computationally expensive. The complexity is proportional to the number of scattering events and the cost of evaluating derivatives [G].
*   **Memory Complexity**: Requires storing scene derivatives and intermediate radiance values along the path. Memory usage scales with path length [G].
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Provides unbiased estimates of radiance derivatives, enabling gradient-based optimization in differentiable rendering [S1, unofficial].
*   **Limitations**: High variance due to the complexity of the derivative terms and the need for edge integrals. Computationally intensive compared to standard path tracing [S1, unofficial].
*   **When to Use**: When gradients of the radiance with respect to scene parameters are required, such as in inverse rendering or material optimization [S1, unofficial].
*   **When Not to Use**: For standard rendering where only radiance values are needed, or when computational resources are limited [G].

## Background
The method originates from "A Differential Theory of Radiative Transfer" [S1, unofficial]. It extends the theory of radiative transfer to include derivatives, providing a framework for differentiable rendering in participating media [S1, unofficial]. The approach handles both volumetric and interfacial scattering events, offering a comprehensive solution for computing radiance derivatives in complex scenes [S1, unofficial].

## Sources
[S1] A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
