# Microflake Model

Also called: Jakob et al. model, Physically-based microflake model, Standard Microflake Model, anisotropic microflake model

## Overview
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic, perfectly specular flakes [S1]. It solves the problem of accurately rendering anisotropic media (such as fur, hair, or certain fabrics) by deriving direction-dependent attenuation and phase functions directly from a microflake normal distribution function (NDF) [S1]. Unlike generic anisotropic phase functions which often violate Helmholtz reciprocity, the Microflake Model satisfies reciprocity by construction [S1]. It is distinct from the [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md) which typically assumes isotropic or simple Henyey-Greenstein scattering, and from the [Single Scattering Approximation](5264-single-scattering-approximation.md) which may use arbitrary phase functions without physical grounding in surface geometry.

## How it works
The model defines the radiative transfer properties of the medium based on the statistical orientation of its constituent microflakes.

**Inputs and Outputs:**
*   **Input:** Microflake Normal Distribution Function $D(\omega_m)$, density $\rho$, albedo $\alpha(\lambda)$.
*   **Output:** Attenuation coefficient $\sigma_t(\omega)$, scattering coefficient $\sigma_s(\omega)$, and phase function $f(\omega \rightarrow \omega')$ for any incoming direction $\omega$ and outgoing direction $\omega'$.

**Data Structures:**
The implementation requires evaluating the NDF $D(\omega_m)$ for any normal $\omega_m$. Common NDFs include the Power of Sine/Cosine distributions [Jak10] or the Gaussian Fiber Distribution [Gaussian Fiber Distribution](6661-gaussian-fiber-distribution.md). The NDF is typically stored as a function or a precomputed lookup table if analytical evaluation is too costly.

**Parameters:**
*   $\rho$: Total area of flakes per unit volume ($m^{-1}$) [S1].
*   $\alpha(\lambda)$: Wavelength-dependent albedo of the microflakes (unitless) [S1].
*   $D(\omega)$: Microflake normal distribution function ($st^{-1}$) [S1].

**Formulas:**
1.  **Attenuation Coefficient:**
    $$ \sigma_t(\omega) = \rho \int D(\omega_m) \langle \omega \cdot \omega_m \rangle d\omega_m $$
    where $\langle \cdot \rangle$ denotes the clamped dot product ($\max(0, \cdot)$) [S1]. This integral represents the projected area of the flakes in direction $\omega$.

2.  **Scattering Coefficient:**
    $$ \sigma_s(\omega) = \alpha(\lambda) \sigma_t(\omega) $$
    This assumes the microflake albedo is not view-dependent [S1].

3.  **Phase Function:**
    $$ f(\omega \rightarrow \omega') = \frac{\rho \alpha(\lambda) D(\omega_h)}{4 \sigma_s(\omega)} $$
    where $\omega_h$ is the half-vector between the incoming direction $\omega$ and outgoing direction $\omega'$:
    $$ \omega_h = \frac{\omega + \omega'}{\|\omega + \omega'\|} $$
    This formula assumes perfectly specular microflakes [S1].

**Assumptions and Constraints:**
*   The model assumes microflakes reflect on both sides, requiring the NDF to satisfy symmetry: $D(\omega) = D(-\omega)$ [S1].
*   The model satisfies Helmholtz reciprocity by construction: $\sigma_s(\omega)f(\omega \rightarrow \omega') = \sigma_s(\omega')f(\omega' \rightarrow \omega)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_microflake_properties(omega_in, omega_out, D_func, rho, albedo):
    """
    Computes attenuation, scattering coefficients, and phase function
    for the standard Microflake Model.

    Args:
        omega_in (Vector): Incoming light direction.
        omega_out (Vector): Outgoing view direction.
        D_func (Function): Microflake Normal Distribution Function D(omega_m).
        rho (float): Density (surface area per unit volume).
        albedo (float): Albedo of the microflakes.

    Returns:
        sigma_t (float): Attenuation coefficient.
        sigma_s (float): Scattering coefficient.
        phase (float): Phase function value.
    """
    # 1. Compute Attenuation Coefficient sigma_t
    # sigma_t(omega) = rho * integral(D(omega_m) * <omega . omega_m> domega_m)
    # Note: The integral must be evaluated numerically or analytically depending on D_func.
    # For implementation, this is often precomputed or approximated.
    # Here we denote the result of the integral as I_proj.
    I_proj = integrate_projected_area(omega_in, D_func)
    sigma_t = rho * I_proj

    # 2. Compute Scattering Coefficient sigma_s
    sigma_s = albedo * sigma_t

    # 3. Compute Half Vector omega_h
    # omega_h = (omega_in + omega_out) / ||omega_in + omega_out||
    half_vec = omega_in + omega_out
    half_vec_norm = length(half_vec)
    
    # Handle degenerate case where omega_in == -omega_out (backscattering)
    if half_vec_norm < 1e-6:
        # The phase function is undefined or requires special handling for exact backscattering
        # In practice, this is a measure-zero event. Return 0 or handle via limit.
        return sigma_t, sigma_s, 0.0

    omega_h = half_vec / half_vec_norm

    # 4. Compute Phase Function
    # f(omega -> omega') = (rho * alpha * D(omega_h)) / (4 * sigma_s(omega))
    D_h = D_func(omega_h)
    
    # Avoid division by zero if sigma_s is 0
    if sigma_s < 1e-6:
        phase = 0.0
    else:
        phase = (rho * albedo * D_h) / (4.0 * sigma_s)

    return sigma_t, sigma_s, phase

def integrate_projected_area(omega, D_func):
    """
    Computes the integral of D(omega_m) * <omega . omega_m> over the hemisphere.
    This is a placeholder for the specific integration method required by the chosen NDF.
    Many NDFs have closed-form solutions for this integral.
    """
    # Example: If D_func is a specific analytical distribution, use its closed form.
    # Otherwise, use numerical quadrature.
    # Not given in the sources: Specific integration algorithm depends on D_func.
    # [G] For a general implementation, one might use Monte Carlo integration or
    # analytical formulas provided in the NDF's source paper (e.g., [Jak10]).
    pass
```

## Subtopics

### [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md)
Extends the standard model by accounting for microscopic self-shadowing and masking effects through a shadowing function $A(\omega)$, improving accuracy for dense media.

### [Gaussian Fiber Distribution](6661-gaussian-fiber-distribution.md)
A specific NDF used within the Microflake Model for simulating anisotropic media like fur, offering computational efficiency over Power of Sine/Cosine distributions.

## Operating characteristics
*   **Complexity:** The computational cost is dominated by the evaluation of the integral for $\sigma_t(\omega)$ and the NDF $D(\omega_h)$. If analytical solutions for the NDF are available, evaluation is $O(1)$. Otherwise, numerical integration is required.
*   **Strengths:** Physically grounded; satisfies Helmholtz reciprocity; naturally produces anisotropic attenuation and scattering; handles downsampling better than voxel-based methods when combined with self-shadowing extensions [S1].
*   **Limitations:** Assumes perfectly specular microflakes; requires the NDF to be symmetric ($D(\omega) = D(-\omega)$); the standard model ignores microscopic self-shadowing, which can lead to inaccuracies in dense media [S1].
*   **When to use:** Rendering anisotropic participating media such as fur, hair, or cloth where physical accuracy and reciprocity are important.
*   **When not to use:** When the medium consists of non-specular particles or when computational resources are extremely limited and an approximate isotropic model suffices.

## Background
The Microflake Model was introduced by Jakob et al. in 2010 [JAM* 10] as a physically-based model for participating media. It is based on the anisotropic Radiative Transfer Equation (RTE) and derives its properties from a microflake normal distribution function. The model was designed to address the limitations of arbitrary anisotropic phase functions, which often break Helmholtz reciprocity. It has been extended by later work, such as the Microscopic Self-Shadowing Microflake Model, to handle dense media more accurately [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
