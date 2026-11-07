# Microflake Model

Also called: Jakob et al. model, Physically-based microflake model, Standard Microflake Model, anisotropic microflake model

## Overview
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic flakes, where orientations are statistically described by a Normal Distribution Function (NDF) $D(\boldsymbol{\omega})$ [S1]. It solves the problem of accurately simulating anisotropic light transport in media like fur, hair, or paper, where standard isotropic models fail to capture direction-dependent attenuation and scattering [S1]. Unlike isotropic single-scattering approximations, the Microflake Model produces anisotropic attenuation coefficients and direction-dependent phase functions that satisfy Helmholtz's reciprocity principle by construction [S1].

## How it works
The model defines the optical properties of the medium based on the statistical distribution of microflake normals. The core assumption is that microflakes are perfectly specular and reflect on both sides, requiring the NDF to satisfy $D(\boldsymbol{\omega}) = D(-\boldsymbol{\omega})$ [S1].

**Inputs:**
*   $\boldsymbol{\omega}$: Incoming light direction (or view direction for attenuation).
*   $\boldsymbol{\omega}'$: Outgoing direction.
*   $D(\boldsymbol{\omega}_m)$: Microflake Normal Distribution Function, unit $\text{sr}^{-1}$ [S1].
*   $\rho$: Density, defined as the total area of microflake surface per unit volume, unit $\text{m}^{-1}$ [S1].
*   $\alpha(\lambda)$: Wavelength-dependent albedo of the microflakes, unitless [S1].

**Outputs:**
*   $\sigma_t(\boldsymbol{\omega})$: Attenuation coefficient, unit $\text{m}^{-1}$ [S1].
*   $\sigma_s(\boldsymbol{\omega})$: Scattering coefficient, unit $\text{m}^{-1}$ [S1].
*   $f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}')$: Phase function, unit $\text{sr}^{-1}$ [S1].

**Data Structures:**
The implementation requires evaluating the NDF $D(\boldsymbol{\omega}_m)$ for arbitrary directions. Common implementations use analytical distributions (e.g., Power of Sine/Cosine, Gaussian) or precomputed lookup tables. The clamped dot product $\langle \mathbf{a} \cdot \mathbf{b} \rangle$ is defined as $\max(0, \mathbf{a} \cdot \mathbf{b})$ [S1].

**Formulas:**

1.  **Attenuation Coefficient:**
    Derived from the integral of projected areas over microflake orientations:
    $$ \sigma_t(\boldsymbol{\omega}) = \rho \int D(\boldsymbol{\omega}_m) \langle \boldsymbol{\omega} \cdot \boldsymbol{\omega}_m \rangle d\boldsymbol{\omega}_m $$
    where $\boldsymbol{\omega}_m$ is the microflake normal [S1].

2.  **Scattering Coefficient:**
    Assuming view-independent albedo:
    $$ \sigma_s(\boldsymbol{\omega}) = \alpha(\lambda) \sigma_t(\boldsymbol{\omega}) $$
    [S1]

3.  **Phase Function:**
    For specular microflakes:
    $$ f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}') = \frac{\rho \alpha(\lambda) D(\boldsymbol{\omega}_h)}{4 \sigma_s(\boldsymbol{\omega})} $$
    where $\boldsymbol{\omega}_h$ is the half-vector between incoming and outgoing directions:
    $$ \boldsymbol{\omega}_h = \frac{\boldsymbol{\omega} + \boldsymbol{\omega}'}{\|\boldsymbol{\omega} + \boldsymbol{\omega}'\|} $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_microflake_properties(omega_in, omega_out, D_func, rho, albedo):
    """
    Computes attenuation, scattering coefficients, and phase function
    for the standard Microflake Model.
    
    Args:
        omega_in (vec3): Incoming direction (normalized).
        omega_out (vec3): Outgoing direction (normalized).
        D_func (function): Evaluates the Normal Distribution Function D(omega_m).
        rho (float): Density (surface area per unit volume).
        albedo (float): Wavelength-dependent albedo.
        
    Returns:
        sigma_t (float): Attenuation coefficient.
        sigma_s (float): Scattering coefficient.
        phase (float): Phase function value.
    """
    # 1. Compute Attenuation Coefficient sigma_t
    # sigma_t(omega) = rho * integral(D(omega_m) * max(0, omega . omega_m) d omega_m)
    # Note: The integral over the hemisphere must be evaluated numerically or 
    # analytically depending on the specific form of D_func.
    # For a general implementation, one might use Monte Carlo integration or 
    # a precomputed table for D_func.
    
    # Example: Monte Carlo estimation for sigma_t
    num_samples = 1024
    integral_sum = 0.0
    for i in range(num_samples):
        # Sample omega_m from a uniform distribution on the hemisphere 
        # or importance sample from D_func if available.
        # Here we assume a simple uniform hemisphere sample for demonstration,
        # though importance sampling D is preferred for efficiency.
        omega_m = sample_hemisphere_uniform()
        
        # Clamped dot product <omega . omega_m>
        dot_product = max(0.0, dot(omega_in, omega_m))
        
        # Evaluate D at omega_m
        d_val = D_func(omega_m)
        
        integral_sum += d_val * dot_product
        
    # Average over samples and scale by 2*pi (area of hemisphere) / N
    # Or more precisely: integral = (2*pi / N) * sum(...)
    sigma_t = rho * (2.0 * 3.14159265358979 / num_samples) * integral_sum
    
    # 2. Compute Scattering Coefficient sigma_s
    sigma_s = albedo * sigma_t
    
    # 3. Compute Phase Function
    # omega_h = (omega_in + omega_out) / ||omega_in + omega_out||
    half_vec = omega_in + omega_out
    half_vec_norm = length(half_vec)
    
    if half_vec_norm < 1e-6:
        # Degenerate case: omega_in == -omega_out
        # Phase function is undefined or requires limit analysis.
        # For specular reflection, energy is concentrated in the mirror direction.
        return sigma_t, sigma_s, 0.0 # Or handle as delta function in renderer

    omega_h = half_vec / half_vec_norm
    
    # f(omega -> omega') = (rho * alpha * D(omega_h)) / (4 * sigma_s(omega))
    d_h = D_func(omega_h)
    
    if sigma_s < 1e-6:
        return sigma_t, sigma_s, 0.0
        
    phase = (rho * albedo * d_h) / (4.0 * sigma_s)
    
    return sigma_t, sigma_s, phase
```

**Edge Cases and Pitfalls:**
*   **Reciprocity:** Arbitrary phase functions in anisotropic media often break Helmholtz's reciprocity principle ($\sigma_s(\boldsymbol{\omega})f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}') = \sigma_s(\boldsymbol{\omega}')f(\boldsymbol{\omega}' \rightarrow \boldsymbol{\omega})$). The Microflake Model satisfies reciprocity by construction [S1].
*   **Degenerate Half-Vector:** If $\boldsymbol{\omega} = -\boldsymbol{\omega}'$, the half-vector is zero. This corresponds to back-scattering. In specular models, this is often a delta function or handled via specific limits [S1].
*   **NDF Symmetry:** The model assumes $D(\boldsymbol{\omega}) = D(-\boldsymbol{\omega})$ (double-sided flakes) [S1].

## Subtopics

### [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md)
Extends the standard model by introducing correlations of microflake positions at a microscopic scale to handle self-shadowing in dense media.

### [Gaussian Fiber Distribution](6661-gaussian-fiber-distribution.md)
A specific NDF used within the Microflake Model for simulating fur/hair, offering computational efficiency over Power of Sine/Cosine distributions.

## Operating characteristics
*   **Complexity:** The computational cost is dominated by the evaluation of the integral for $\sigma_t(\boldsymbol{\omega})$. If $D(\boldsymbol{\omega})$ has a closed-form solution for the integral, evaluation is $O(1)$. Otherwise, numerical integration (e.g., Monte Carlo) is required, scaling with the number of samples [S1].
*   **Memory:** Requires storage for the NDF parameters and potentially lookup tables if precomputed [S1].
*   **Strengths:** Physically based; satisfies reciprocity; handles anisotropy naturally through the NDF [S1].
*   **Limitations:** The standard model assumes uncorrelated microflake positions, leading to inaccuracies in dense or clustered media where self-shadowing occurs [S1]. It does not account for microscopic self-shadowing [S1].

## Background
The Microflake Model was introduced by Jakob et al. in [JAM* 10] as a physically-based model for participating media. It is based on anisotropic Radiative Transfer Equation (RTE) frameworks. The model treats the medium as a homogeneous cloud of microflakes, deriving optical properties from the statistical distribution of their normals [S1]. It is widely used for rendering anisotropic materials like fur, hair, and paper [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
