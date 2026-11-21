# Simplified Model with Isotropic Self-Shadowing

Also called: Isotropic self-shadowing model

## Overview
This method simplifies the Microscopic Self-Shadowing Microflake Model by assuming the shadowing function $A$ is isotropic (constant for all directions) [S1]. This reduces parameter count and implementation complexity while maintaining sufficient accuracy in practice [S1]. It differs from the parent model by decoupling the directional dependence of shadowing, allowing closed-form solutions for spherical integrals that would otherwise require numerical approximation [S1].

## How it works
**Inputs:**
*   $\lambda$: Wavelength.
*   $\rho$: Microflake density.
*   $D(\omega_m)$: Microflake distribution function.
*   $\alpha_{ss}(\lambda)$: Single scattering albedo.
*   $\alpha_{ms}(\lambda)$: Multiple scattering albedo.
*   $A$: Isotropic shadowing factor (scalar, $0 \le A \le 1$) [S1].

**Outputs:**
*   $\sigma_t(\omega)$: Total attenuation coefficient.
*   $\sigma_{ss}(\omega)$: Single scattering coefficient.
*   $f_{ss}(\omega \rightarrow \omega')$: Single scattering phase function.
*   $\sigma_{ms}(\omega)$: Multiple scattering coefficient.
*   $f_{ms}(\omega')$: Multiple scattering phase function [S1].

**Data Structures:**
The method requires standard scalar and vector types for directions and coefficients, along with a function or analytical expression for $D(\omega_m)$ [S1].

**Steps:**

1.  **Compute Attenuation Coefficient $\sigma_t(\omega)$:**
    Calculate the spherical integral of the microflake distribution weighted by the cosine of the angle between the view direction $\omega$ and microflake normal $\omega_m$, scaled by density $\rho$ and the isotropic shadowing factor $A$ [S1]:
    $$
    \sigma_t(\omega) = A \rho \int_{\Omega} D(\omega_m) \langle \omega \cdot \omega_m \rangle \, d\omega_m
    $$
    where $\langle \cdot \rangle$ denotes the positive part $\max(0, \omega \cdot \omega_m)$ [S1]. Note that $\sigma_t(\omega)$ equals $A$ times the attenuation coefficient of the standard microflake model [S1].

2.  **Compute Single Scattering Coefficient $\sigma_{ss}(\omega)$:**
    Scale the attenuation coefficient by the single scattering albedo and the shadowing factor $A$ [S1]:
    $$
    \sigma_{ss}(\omega) = \alpha_{ss}(\lambda) A \sigma_t(\omega)
    $$
    *Note: The source table lists $\sigma_i(\omega)$ in Eq. 4.9, but the surrounding text and Eq. 4.8 clarify this as a typo for $\sigma_t(\omega)$ [S1].*

3.  **Compute Single Scattering Phase Function $f_{ss}(\omega \rightarrow \omega')$:**
    Evaluate the specular phase function using the half-vector $\omega_h = \frac{\omega + \omega'}{\|\omega + \omega'\|}$ [S1]:
    $$
    f_{ss}(\omega \rightarrow \omega') = \frac{\rho \alpha_{ss}(\lambda) A^2 D(\omega_h)}{4 \sigma_{ss}(\omega)}
    $$
    *Note: The source text contains garbled LaTeX $\boldsymbol{\rho}_{\sigma_{ss}}(\boldsymbol{\lambda})$ in Eq. 4.10, which is corrected here to standard microflake model notation [S1].*

4.  **Compute Multiple Scattering Coefficient $\sigma_{ms}(\omega)$:**
    Calculate the remaining attenuation attributed to multiple scattering, scaled by the multiple scattering albedo and the factor $(1-A)$ [S1]:
    $$
    \sigma_{ms}(\omega) = \alpha_{ms}(\lambda) \sigma_t(\omega) (1 - A)
    $$

5.  **Compute Multiple Scattering Phase Function $f_{ms}(\omega')$:**
    The multiple scattering phase function is proportional to the total attenuation coefficient [S1]:
    $$
    f_{ms}(\omega') = \frac{\sigma_t(\omega')}{\int_{\Omega} \sigma_t(\omega'') \, d\omega''}
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_isotropic_self_shadowing_params(omega, omega_prime, rho, alpha_ss, alpha_ms, A, D_func):
    # Step 1: Compute sigma_t
    # Evaluate spherical integral of D(omega_m) * max(0, omega . omega_m)
    # using closed-form expressions for trigonometric lobes as per Section 5 [S1].
    I_t = evaluate_analytical_integral(omega, D_func)
    sigma_t = A * rho * I_t

    # Step 2: Compute sigma_ss
    sigma_ss = alpha_ss * A * sigma_t

    # Step 3: Compute f_ss
    omega_h = omega + omega_prime
    norm_omega_h = sqrt(omega_h[0]**2 + omega_h[1]**2 + omega_h[2]**2)
    omega_h = omega_h / norm_omega_h
    D_omega_h = D_func(omega_h)
    numerator_ss = rho * alpha_ss * (A ** 2) * D_omega_h
    denominator_ss = 4 * sigma_ss
    # Standard numerical guard against division by zero [G]
    f_ss = numerator_ss / max(denominator_ss, 1e-8)

    # Step 4: Compute sigma_ms
    sigma_ms = alpha_ms * sigma_t * (1 - A)

    # Step 5: Compute f_ms
    # Evaluate integral of sigma_t(omega'') over the sphere analytically [S1].
    I_sigma_t = evaluate_analytical_integral_sphere(rho, A, D_func)
    I_t_prime = evaluate_analytical_integral(omega_prime, D_func)
    sigma_t_prime = A * rho * I_t_prime
    f_ms = sigma_t_prime / max(I_sigma_t, 1e-8)

    return sigma_t, sigma_ss, f_ss, sigma_ms, f_ms

def evaluate_analytical_integral(omega, D_func):
    # The source provides closed-form solutions for D functions composed of
    # trigonometric lobes (cos^2n, sin^2n) in Section 5 [S1].
    # The implementer must substitute the specific analytical expression 
    # for their chosen D_func type.
    pass

def evaluate_analytical_integral_sphere(rho, A, D_func):
    # Integrates sigma_t(omega'') over the sphere analytically based on D [S1].
    # The implementer must substitute the specific analytical expression 
    # for their chosen D_func type.
    pass
```

**Parameters:**
*   $A$: Isotropic shadowing factor. Typical values are not explicitly given, but it acts as a scalar attenuation multiplier [S1].
*   $\alpha_{ss}(\lambda), \alpha_{ms}(\lambda)$: Albedos, dependent on wavelength [S1].

**Edge Cases and Pitfalls:**
*   The spherical integrals require closed-form evaluation for specific distribution functions $D(\omega_m)$, typically trigonometric lobes [S1]. If $D(\omega_m)$ lacks a closed form, the method's efficiency advantage is lost [S1].

## Subtopics

### Trigonometric Lobes Implementation
*   **What it is:** A technique for evaluating the spherical integrals in the self-shadowing model using functions like $\cos^{2n}$ and $\sin^{2n}$.
*   **Difference:** Provides closed-form solutions for the integrals, enabling efficient implementation.
*   **When to choose:** When the microflake distribution $D$ can be approximated by trigonometric lobes.
*   **Link:** [Trigonometric Lobes Implementation](52-trigonometric-lobes-implementation.md)

## Operating characteristics
*   **Time Complexity:** Depends on the evaluation of $D(\omega_m)$ and its integrals. If closed-form expressions are available (e.g., via trigonometric lobes), evaluation is $O(1)$ per direction [S1].
*   **Memory Complexity:** Low, as it does not require storing complex shadowing maps or high-resolution distributions [S1].
*   **Performance:** The isotropic assumption simplifies implementation and reduces parameters, making it faster to tune and render than the full anisotropic self-shadowing model [S1].
*   **Strengths:** Simplicity, reduced parameter space, sufficient accuracy for many applications [S1].
*   **Limitations:** Less accurate than the full anisotropic model, especially for highly directional microflake distributions [S1].
*   **When to use:** When computational efficiency is prioritized and the microflake distribution is not strongly directional [S1].

## Background
*   **Origin:** Introduced in "A new microflake model with microscopic self-shadowing for accurate volume downsampling" [S1].
*   **History:** Developed as a simplification of the general self-shadowing microflake model to enable efficient implementations with fewer parameters [S1].
*   **Usage:** Used in volume rendering for participating media [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
