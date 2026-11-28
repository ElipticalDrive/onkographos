# Microflake Model

Also called: Jakob et al. model, Physically-based microflake model, Standard Microflake Model, anisotropic microflake model

## Overview
The Microflake Model is a physically-based participating medium model that represents a volume as a homogeneous cloud of microscopic, specular flakes [S1]. It solves the problem of rendering anisotropic media (such as fur, fabrics, or powders) by deriving direction-dependent extinction and phase functions from a statistical distribution of microflake orientations [S1]. Unlike isotropic single-scattering approximations, it produces anisotropic attenuation coefficients and direction-dependent phase functions, satisfying Helmholtz's reciprocity principle by construction [S1].

## How it works
**Inputs:**
*   $\rho$: Density, the total area of microflake surface per unit volume ($m^{-1}$) [S1].
*   $D(\omega_m)$: Microflake Normal Distribution Function (NDF), describing the statistical orientation of microflake normals $\omega_m$ ($st^{-1}$) [S1].
*   $\alpha(\lambda)$: Wavelength-dependent albedo of the microflakes (unitless) [S1].
*   $\omega$: Incoming light direction (or view direction for scattering) [S1].
*   $\omega'$: Outgoing light direction (or view direction for scattering) [S1].

**Outputs:**
*   $\sigma_t(\omega)$: Direction-dependent attenuation coefficient ($m^{-1}$) [S1].
*   $\sigma_s(\omega)$: Direction-dependent scattering coefficient ($m^{-1}$) [S1].
*   $f(\omega \rightarrow \omega')$: Direction-dependent phase function ($st^{-1}$) [S1].

**Data Structures:**
The model stores the density $\rho$ ($m^{-1}$), the wavelength-dependent albedo $\alpha(\lambda)$ (unitless), and the microflake normal distribution function $D(\omega_m)$ ($st^{-1}$) [S1].

**Steps:**

1.  **Compute the Attenuation Coefficient $\sigma_t(\omega)$:**
    Compute the integral of the projected area of the microflakes over all possible orientations [S1].
    $$ \sigma_t(\omega) = \rho \int D(\omega_m) \langle \omega \cdot \omega_m \rangle d\omega_m $$
    where $\langle \cdot \rangle$ denotes the clamped dot product ($\max(0, \cdot)$) [S1].

2.  **Compute the Scattering Coefficient $\sigma_s(\omega)$:**
    Scale the attenuation coefficient by the albedo [S1]. This assumes the microflake albedo is not view-dependent [S1].
    $$ \sigma_s(\omega) = \alpha(\lambda) \sigma_t(\omega) $$
    where $\alpha(\lambda)$ is the albedo at wavelength $\lambda$ [S1].

3.  **Compute the Phase Function $f(\omega \rightarrow \omega')$:**
    For specular microflakes, the phase function is derived from the NDF evaluated at the half-vector [S1].
    $$ f(\omega \rightarrow \omega') = \frac{\rho \alpha(\lambda) D(\omega_h)}{4 \sigma_s(\omega)} $$
    where $\omega_h$ is the normalized half-vector between the incoming and outgoing directions:
    $$ \omega_h = \frac{\omega + \omega'}{\|\omega + \omega'\|} $$
    [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_microflake_properties(rho, albedo, ndf_eval, omega_in, omega_out):
    # Step 1: Compute attenuation coefficient sigma_t
    # sigma_t = rho * integral(D(omega_m) * max(0, omega_in . omega_m) d_omega_m)
    # [G] The sources do not provide a closed-form for the integral;
    # it must be evaluated analytically for the chosen NDF or numerically.
    integral_term = 0.0
    num_samples = 1000
    for _ in range(num_samples):
        phi = random.uniform(0, 2 * math.pi)
        cos_theta = random.random()
        sin_theta = math.sqrt(1.0 - cos_theta * cos_theta)
        omega_m = sample_hemisphere(omega_in, phi, cos_theta, sin_theta)
        dot_val = max(0.0, Vector3_dot(omega_in, omega_m))
        integral_term += ndf_eval(omega_m) * dot_val
    integral_term = (integral_term / num_samples) * (2.0 * math.pi)
    sigma_t = rho * integral_term

    # Step 2: Compute scattering coefficient sigma_s
    sigma_s = albedo * sigma_t

    # Step 3: Compute phase function f(omega_in -> omega_out)
    half_vector = Vector3_add(omega_in, omega_out)
    half_vector_norm = math.sqrt(Vector3_dot(half_vector, half_vector))

    if half_vector_norm < 1e-6:
        return sigma_t, sigma_s, 0.0

    half_vector_normalized = Vector3_scale(half_vector, 1.0 / half_vector_norm)
    d_half = ndf_eval(half_vector_normalized)

    if sigma_s < 1e-6:
        phase = 0.0
    else:
        phase = (rho * albedo * d_half) / (4.0 * sigma_s)

    return sigma_t, sigma_s, phase

def sample_hemisphere(omega, phi, cos_theta, sin_theta):
    # [G] Standard basis construction for a hemisphere around an arbitrary normal
    if abs(omega.z) < 0.999:
        up = Vector3(0.0, 0.0, 1.0)
    else:
        up = Vector3(1.0, 0.0, 0.0)
    tangent = Vector3_normalize(Vector3_cross(omega, up))
    bitangent = Vector3_cross(omega, tangent)
    return Vector3_add(
        Vector3_add(Vector3_scale(tangent, sin_theta * math.cos(phi)),
                    Vector3_scale(bitangent, sin_theta * math.sin(phi))),
        Vector3_scale(omega, cos_theta)
    )

def Vector3_dot(a, b): return a.x * b.x + a.y * b.y + a.z * b.z
def Vector3_add(a, b): return Vector3(a.x + b.x, a.y + b.y, a.z + b.z)
def Vector3_scale(v, s): return Vector3(v.x * s, v.y * s, v.z * s)
def Vector3_cross(a, b): return Vector3(a.y*b.z - a.z*b.y, a.z*b.x - a.x*b.z, a.x*b.y - a.y*b.x)
def Vector3_normalize(v):
    l = math.sqrt(Vector3_dot(v, v))
    return Vector3_scale(v, 1.0 / l) if l > 1e-6 else Vector3(0.0, 0.0, 1.0)

class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
```

**Parameters and Typical Values:**
*   $\rho$: Not given in the sources [S1]. Depends on the specific medium density.
*   $\alpha(\lambda)$: Not given in the sources [S1].
*   $D(\omega)$: Defined by the chosen NDF variant (e.g., Gaussian, Power of Sine/Cosine) [S1].

**Edge Cases and Pitfalls:**
*   **Reciprocity:** Arbitrary phase functions in anisotropic media often break Helmholtz's reciprocity principle ($\sigma_s(\omega)f(\omega \rightarrow \omega') = \sigma_s(\omega')f(\omega' \rightarrow \omega)$) [S1]. The microflake model satisfies reciprocity by construction [S1].
*   **Backscattering Singularity:** When $\omega = -\omega'$, the half-vector $\omega_h$ has zero length and the phase function is undefined [S1].
*   **NDF Symmetry:** The model assumes microflakes reflect on both sides, requiring $D(\omega) = D(-\omega)$ [S1].

## Subtopics

### [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md)
Extends the standard model by introducing correlations of microflake positions at a microscopic scale to account for self-shadowing and masking effects. Chosen when microscopic shadowing significantly affects light transport.

### [Gaussian Fiber Distribution](6661-gaussian-fiber-distribution.md)
A microflake normal distribution function (NDF) for anisotropic media that provides a more efficient implementation than Power of Sine/Cosine distributions. Chosen for fiber-like media where computational efficiency is prioritized over closed-form attenuation expressions.

### [Anisotropic Radiative Transfer with SGGX Phase Functions](16529-anisotropic-radiative-transfer-with-sggx-phase-functions.md)
Extends the RTE to model structured micro-geometries by making scattering parameters direction-dependent using the SGGX representation for $D(m)$. Chosen when fast evaluation and linear interpolation of phase functions are required for complex micro-geometries.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Physically-based, satisfies reciprocity, handles anisotropy naturally [S1].
*   **Limitations:** Assumes specular microflakes, does not account for microscopic self-shadowing (unless using the extended model), requires solving the NDF integral for attenuation [S1].
*   **When to use:** Rendering anisotropic participating media like fur, fabrics, or powders where single scattering is a sufficient approximation [G].
*   **When not to use:** When multiple scattering is significant or when microscopic self-shadowing is critical [G].

## Background
The Microflake Model was introduced by Jakob et al. [JAM* 10] as a physically-based model for participating media [S1]. It is based on the anisotropic Radiative Transfer Equation (RTE) framework [S1]. The model represents the medium as a homogeneous cloud of microscopic, specular flakes [S1]. It was developed to address the limitations of isotropic models in rendering anisotropic materials [S1]. The model satisfies Helmholtz's reciprocity principle by construction, which is a significant advantage over arbitrary anisotropic phase functions [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
