# Microflake Model

Also called: Jakob et al. model, Physically-based microflake model, Standard Microflake Model, anisotropic microflake model

## Overview
The Microflake Model is a physically-based participating medium model where the medium is treated as a homogeneous cloud of microscopic flakes [S1]. It solves the problem of accurately modeling anisotropic light transport in fibrous or layered media (such as hair, fur, or paper) by deriving attenuation coefficients and phase functions directly from a microflake normal distribution function [S1]. Unlike standard isotropic phase functions, this model produces direction-dependent scattering and attenuation, ensuring that Helmholtz's reciprocity principle is satisfied by construction, which arbitrary anisotropic phase functions often break [S1].

## How it works
The model defines the optical properties of the medium based on the statistical orientation of its constituent microflakes.

**Inputs and Outputs**
*   **Inputs:**
    *   $\rho$: Density, the total area of microflake surface per unit volume ($m^{-1}$) [S1].
    *   $D(\omega_m)$: Microflake normal distribution function, describing the statistical orientation of flakes ($st^{-1}$) [S1].
    *   $\alpha(\lambda)$: Wavelength-dependent albedo of the microflakes (unitless) [S1].
    *   $\omega$: Incident ray direction.
    *   $\omega'$: Scattered ray direction.
*   **Outputs:**
    *   $\sigma_t(\omega)$: Anisotropic attenuation coefficient ($m^{-1}$) [S1].
    *   $\sigma_s(\omega)$: Anisotropic scattering coefficient ($m^{-1}$) [S1].
    *   $f(\omega \rightarrow \omega')$: Phase function ($st^{-1}$) [S1].

**Data Structures**
The core data structure is the Normal Distribution Function (NDF) $D(\omega_m)$. Common implementations include the Gaussian Fiber Distribution [S1] or the SGGX distribution [S1]. The function maps a normal vector $\omega_m$ to a density value.

**Formulas**
1.  **Attenuation Coefficient:** Derived from the integral of projected areas over all microflake orientations [S1]:
    $$ \sigma_t(\omega) = \rho \int_{\Omega} D(\omega_m) \langle \omega \cdot \omega_m \rangle d\omega_m $$
    where $\langle \cdot \rangle$ denotes the clamped dot product (i.e., $\max(0, \omega \cdot \omega_m)$) [S1].

2.  **Scattering Coefficient:** Assuming the albedo is not view-dependent, the scattering coefficient is proportional to the attenuation coefficient [S1]:
    $$ \sigma_s(\omega) = \alpha(\lambda) \sigma_t(\omega) $$
    where $\alpha(\lambda)$ is the albedo at wavelength $\lambda$ [S1].

3.  **Phase Function:** For specular microflakes, the phase function is defined using the half-vector $\omega_h$ [S1]:
    $$ f(\omega \rightarrow \omega') = \frac{\rho \alpha(\lambda) D(\omega_h)}{4 \sigma_s(\omega)} $$
    where the half-vector is calculated as:
    $$ \omega_h = \frac{\omega + \omega'}{\|\omega + \omega'\|} $$
    [S1].

**Pseudocode**
The following Python-like pseudocode computes the optical properties given the NDF $D$, density $\rho$, and albedo $\alpha$. Note that the integral for $\sigma_t$ must be evaluated numerically or via a closed-form approximation specific to the chosen $D$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_microflake_properties(D_func, rho, alpha, omega_in, omega_out):
    """
    Computes attenuation, scattering coefficients, and phase function
    for the standard microflake model.
    
    Args:
        D_func: Function evaluating the microflake normal distribution D(omega_m).
        rho: Density (total area of flakes per unit volume).
        alpha: Albedo (wavelength dependent, scalar for this step).
        omega_in: Incident direction vector (unit length).
        omega_out: Scattered direction vector (unit length).
        
    Returns:
        sigma_t: Attenuation coefficient.
        sigma_s: Scattering coefficient.
        phase: Phase function value.
    """
    
    # 1. Compute Attenuation Coefficient sigma_t
    # sigma_t = rho * integral(D(omega_m) * <omega_in . omega_m> domega_m)
    # This integral represents the projected area of the flakes.
    # Implementation depends on the specific D_func (e.g., closed-form for SGGX,
    # approximation for Gaussian Fiber).
    # Let's assume a helper function exists for this specific D.
    projected_area_factor = integrate_projected_area(D_func, omega_in)
    sigma_t = rho * projected_area_factor
    
    # 2. Compute Scattering Coefficient sigma_s
    sigma_s = alpha * sigma_t
    
    # 3. Compute Phase Function
    # Calculate half-vector
    half_vec = omega_in + omega_out
    half_vec_norm = math.sqrt(half_vec[0]**2 + half_vec[1]**2 + half_vec[2]**2)
    
    # Avoid division by zero if omega_in == -omega_out (backscattering)
    if half_vec_norm < 1e-8:
        return 0.0, 0.0, 0.0
        
    omega_h = (half_vec[0]/half_vec_norm, 
               half_vec[1]/half_vec_norm, 
               half_vec[2]/half_vec_norm)
               
    # Evaluate D at the half-vector
    D_h = D_func(omega_h)
    
    # Phase function formula: f = (rho * alpha * D(omega_h)) / (4 * sigma_s)
    # Note: sigma_s = alpha * sigma_t, so rho * alpha / sigma_s = rho / sigma_t
    # However, using the explicit formula from sources:
    if sigma_s < 1e-8:
        phase = 0.0
    else:
        phase = (rho * alpha * D_h) / (4.0 * sigma_s)
        
    return sigma_t, sigma_s, phase

def integrate_projected_area(D_func, omega):
    """
    Computes the integral of D(omega_m) * <omega . omega_m> over the hemisphere.
    This is the core computational step that varies by distribution D.
    For SGGX, this has a closed-form solution. For Gaussian Fiber, an approximation is used.
    [G] Placeholder for distribution-specific integration logic.
    """
    # Not given in the sources: Specific integration algorithms for D_func.
    # This function must be implemented according to the chosen NDF.
    raise NotImplementedError("Integration depends on the specific Normal Distribution Function D.")
```

**Parameters and Typical Values**
*   $\rho$ (Density): Not given in the sources.
*   $\alpha$ (Albedo): Wavelength-dependent, unitless. Not given in the sources.
*   $D(\omega)$: Unitless ($st^{-1}$). Shape depends on the chosen distribution (e.g., Gaussian Fiber, SGGX) [S1].

**Edge Cases and Pitfalls**
*   **Reciprocity:** Arbitrary phase functions in anisotropic media often break Helmholtz's reciprocity principle ($\sigma_s(\omega)f(\omega \rightarrow \omega') = \sigma_s(\omega')f(\omega' \rightarrow \omega)$). The microflake model satisfies reciprocity by construction [S1].
*   **Sampling Efficiency:** Sampling the distribution of visible normals using a rejection algorithm based on the sampling of the normal distribution can be inefficient in some configurations [S1].
*   **Closed-Form Availability:** Some distributions, like the Gaussian Fiber Distribution, do not provide a closed-form expression for the attenuation coefficient, requiring efficient approximations [S1].

## Subtopics

### Gaussian Fiber Distribution
A microflake normal distribution used by Zhao et al. [ZJMB11]. It is more efficient than the original implementation of Jakob et al. [JAM* 10], but does not provide a closed-form expression for the attenuation coefficient (using an efficient approximation [Jak10]). Sampling the distribution of visible normals using a rejection algorithm based on the sampling of the normal distribution is inefficient in some configurations [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the evaluation of the integral for $\sigma_t(\omega)$. For distributions like SGGX, this can be evaluated in closed form, offering high efficiency. For others, numerical integration or approximations are required [S1].
*   **Memory Complexity:** Requires storage for the parameters of the Normal Distribution Function $D$ and the density $\rho$ [S1].
*   **Strengths:** Physically-based; satisfies Helmholtz's reciprocity by construction; capable of modeling anisotropic scattering and attenuation [S1].
*   **Limitations:** Requires careful implementation of the NDF evaluation and integration; some distributions lack closed-form attenuation solutions [S1].
*   **When to use:** When rendering media with fibrous or layered structures where anisotropy is significant and physical accuracy is required [S1].

## Background
The Microflake Model was introduced by Jakob et al. in 2010 [JAM* 10] as a physically-based participating medium model. It is based on the anisotropic Radiative Transfer Equation (RTE) framework. The model treats the medium as a homogeneous cloud of microflakes whose orientations are statistically described by a microflake normal distribution function $D(\omega)$ [S1]. Anisotropic microflake normal distributions result in anisotropic attenuation coefficients and direction-dependent phase functions [S1]. The model assumes microflakes reflect on both sides and considers normal distributions $D$ that satisfy $D(\omega) = D(-\omega)$ [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
