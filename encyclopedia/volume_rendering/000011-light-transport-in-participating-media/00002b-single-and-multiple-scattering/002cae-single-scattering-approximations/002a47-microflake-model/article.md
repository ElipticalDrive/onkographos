# Microflake Model

Also called: Jakob et al. model, Physically-based microflake model, Standard Microflake Model, anisotropic microflake model

## Overview
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic flakes, where orientations are statistically described by a normal distribution function $D(\boldsymbol{\omega})$ [S1]. It solves the problem of rendering anisotropic media by deriving direction-dependent attenuation coefficients and phase functions that satisfy Helmholtz's reciprocity principle by construction [S1]. Unlike the Microscopic Self-Shadowing variant [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md), it assumes uncorrelated microflake positions at a microscopic scale [S1].

## How it works
**Inputs:**
*   $\boldsymbol{\omega}$: Incoming light direction or view direction for attenuation [S1].
*   $\boldsymbol{\omega}'$: Outgoing light direction [S1].
*   $D(\boldsymbol{\omega}_m)$: Microflake normal distribution function defined over the unit sphere [S1].
*   $\rho$: Density representing the total area of microflake surface per unit volume [S1].
*   $\alpha(\lambda)$: Wavelength-dependent albedo of the microflakes [S1].

**Outputs:**
*   $\sigma_t(\boldsymbol{\omega})$: Attenuation coefficient [S1].
*   $\sigma_s(\boldsymbol{\omega})$: Scattering coefficient [S1].
*   $f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}')$: Phase function [S1].

**Data Structures:**
The model requires a microflake normal distribution function $D(\boldsymbol{\omega}_m)$ defined over the unit sphere [S1]. The NDF must satisfy the symmetry condition $D(\boldsymbol{\omega}) = D(-\boldsymbol{\omega})$ to model microflakes that reflect on both sides [S1].

**Steps:**

1.  **Compute the Attenuation Coefficient $\sigma_t(\boldsymbol{\omega})$**:
    Calculate the integral of the projected areas over all microflake orientations [S1].
    $$ \sigma_t(\boldsymbol{\omega}) = \rho \int_{S^2} D(\boldsymbol{\omega}_m) \langle \boldsymbol{\omega} \cdot \boldsymbol{\omega}_m \rangle d\boldsymbol{\omega}_m $$
    where $\langle x \rangle = \max(0, x)$ is the clamped dot product [S1].

2.  **Compute the Scattering Coefficient $\sigma_s(\boldsymbol{\omega})$**:
    Derive the scattering coefficient from the attenuation coefficient and the albedo [S1].
    $$ \sigma_s(\boldsymbol{\omega}) = \alpha(\lambda) \sigma_t(\boldsymbol{\omega}) $$

3.  **Compute the Phase Function $f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}')$**:
    For specular microflakes, the phase function is determined by the NDF evaluated at the half-vector between the incoming and outgoing directions [S1].
    $$ f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}') = \frac{\rho \alpha(\lambda) D(\boldsymbol{\omega}_h)}{4 \sigma_s(\boldsymbol{\omega})} $$
    where $\boldsymbol{\omega}_h$ is the normalized half-vector:
    $$ \boldsymbol{\omega}_h = \frac{\boldsymbol{\omega} + \boldsymbol{\omega}'}{\|\boldsymbol{\omega} + \boldsymbol{\omega}'\|} $$

**Parameters:**
*   $\rho$: Density in $m^{-1}$ [S1].
*   $\alpha(\lambda)$: Albedo, unitless and wavelength-dependent [S1].
*   $D(\boldsymbol{\omega})$: NDF in $sr^{-1}$ [S1].

**Edge Cases and Pitfalls:**
*   **Reciprocity**: Arbitrary phase functions in anisotropic media often break Helmholtz's reciprocity principle ($\sigma_s(\boldsymbol{\omega})f(\boldsymbol{\omega} \rightarrow \boldsymbol{\omega}') = \sigma_s(\boldsymbol{\omega}')f(\boldsymbol{\omega}' \rightarrow \boldsymbol{\omega})$) [S1]. The Microflake Model satisfies reciprocity by construction because the phase function and scattering coefficient are derived from the same physical principles [S1].
*   **NDF Symmetry**: The model assumes $D(\boldsymbol{\omega}) = D(-\boldsymbol{\omega})$ [S1]. If the NDF does not satisfy this, the model's physical validity is compromised [G].
*   **Singularity**: If $\sigma_s(\boldsymbol{\omega}) = 0$, the phase function is undefined [S1]. This occurs if $\sigma_t(\boldsymbol{\omega}) = 0$ or $\alpha(\lambda) = 0$ [S1]. In practice, a small epsilon should be added to the denominator or the result clamped to zero [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def dot_product(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/length, v[1]/length, v[2]/length)

def sample_uniform_sphere():
    theta = random.uniform(0, 2 * math.pi)
    cos_theta = random.uniform(-1, 1)
    sin_theta = math.sqrt(1 - cos_theta**2)
    return (math.cos(theta) * sin_theta, math.sin(theta) * sin_theta, cos_theta)

def microflake_model(omega_in, omega_out, D_func, rho, albedo, num_samples=1024):
    # Step 1: Compute Attenuation Coefficient sigma_t via Monte Carlo integration
    sigma_t_sum = 0.0
    for _ in range(num_samples):
        omega_m = sample_uniform_sphere()
        clamped_dot = max(0.0, dot_product(omega_in, omega_m))
        sigma_t_sum += D_func(omega_m) * clamped_dot
    sigma_t = rho * (sigma_t_sum / num_samples) * (4.0 * math.pi)

    # Step 2: Compute Scattering Coefficient sigma_s
    sigma_s = albedo * sigma_t

    # Step 3: Compute Phase Function
    if sigma_s < 1e-8:
        return sigma_t, sigma_s, 0.0

    half_vec = normalize((omega_in[0] + omega_out[0], omega_in[1] + omega_out[1], omega_in[2] + omega_out[2]))
    D_half = D_func(half_vec)
    phase = (rho * albedo * D_half) / (4.0 * sigma_s)

    return sigma_t, sigma_s, phase
```

## Subtopics

### [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md)
Extends the standard model by introducing correlations of microflake positions at a microscopic scale, using a shadowing function $A(\omega)$ to account for self-shadowing and masking. Choose this for dense or clustered media where occlusion between flakes is significant.

### [Gaussian Fiber Distribution](6661-gaussian-fiber-distribution.md)
A specific NDF used within the Microflake Model for simulating anisotropic media like fur. It offers improved computational efficiency over Power of Sine/Cosine distributions but lacks a closed-form expression for the attenuation coefficient, relying on approximations. Choose this for efficient fur rendering.

## Operating characteristics
Time and memory complexity are not given in the sources. Measured performance, scaling, strengths, limitations, and usage conditions are not given in the sources.

## Background
The Microflake Model was introduced by Jakob et al. as a physically-based model for anisotropic participating media [S1]. It is based on the anisotropic Radiative Transfer Equation and derives attenuation coefficients and phase functions from a microflake normal distribution function [S1]. The model assumes that microflakes are perfectly specular and reflect on both sides, satisfying the symmetry condition $D(\boldsymbol{\omega}) = D(-\boldsymbol{\omega})$ [S1]. It was developed to address the limitations of arbitrary phase functions in anisotropic media, which often break reciprocity [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
