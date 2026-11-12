# Transparency and Opacity

Also called: Opacity Calculation

## Overview
Transparency and opacity quantify light attenuation in the absorption-only model [S1, unofficial]. Transparency (or transmittance) $T(s)$ represents the fraction of incident radiance surviving absorption over distance $s$, while opacity $O(s)$ represents the fraction blocked [S1, unofficial]. This model considers only absorption, serving as the foundational component for compositing in volume rendering and radiography [S1, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Extinction coefficient field $\kappa(x, \omega)$ (units: inverse length), ray origin $x$, ray direction $\omega$, and integration distance $s$ [S1, unofficial].
*   **Output:** Transparency $T(s)$ and Opacity $O(s)$ [S1, unofficial].

**Data Structures:**
*   **Extinction Coefficient:** Stored as a scalar field [S1, unofficial]. In radiography contexts, coefficients are pre-computed and stored in a table indexed by material number and energy group number [S3, unofficial].

**Formulas:**
The optical depth $\tau(s)$ along a ray segment of length $s$ is defined as:
$$ \tau(s) = \int_0^s \kappa(x + t\omega, \omega) dt $$
[S1, unofficial]

If $\kappa$ is constant, this simplifies to $\tau(s) = \kappa s$ [S1, unofficial].

Transparency $T(s)$ is the exponential decay of light:
$$ T(s) = e^{-\tau(s)} = e^{-\int_0^s \kappa(x + t\omega) dt} $$
[S1, unofficial]

Opacity $O(s)$ is the complement of transparency:
$$ O(s) = 1 - T(s) $$
[S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_transparency_and_opacity(extinction_func, ray_origin, ray_direction, distance):
    """
    Computes transparency T(s) and opacity O(s) for a ray segment of length s.
    Args:
        extinction_func: Callable returning kappa(x, omega) at position x.
        ray_origin: 3D vector x.
        ray_direction: 3D vector omega (assumed normalized).
        distance: Scalar s > 0.
    Returns:
        T: Transparency in [0, 1].
        O: Opacity in [0, 1].
    """
    num_samples = 100  # Not given in the sources; implementation detail [G]
    step = distance / num_samples
    optical_depth = 0.0
    for i in range(num_samples):
        t = (i + 0.5) * step
        sample_point = ray_origin + t * ray_direction
        kappa = extinction_func(sample_point, ray_direction)
        optical_depth += kappa * step
    transparency = math.exp(-optical_depth)
    opacity = 1.0 - transparency
    return transparency, opacity
```

**Parameters:**
*   **Integration Steps:** Not given in the sources.
*   **Precision:** Standard floating-point precision is assumed [G].

**Edge Cases and Pitfalls:**
*   **High Opacity:** If $\tau(s) \gg 1$, $T(s)$ underflows to 0 [G].
*   **Zero Extinction:** If $\kappa = 0$, $T(s) = 1$ and $O(s) = 0$ [S1, unofficial].
*   **Boundary Artifacts:** In fuzzy surface rendering, naive opacity calculation can cause visual artifacts at object boundaries [S2].

## Subtopics

### Transparency Attenuation
A technique introduced to reduce visual artifacts at object boundaries in fuzzy surface rendering [S2]. In ablation studies, using transparency-decay resulted in a 2.13 dB PSNR gain on the Shelly scene compared to not using it [S2].

### Extinction Coefficient Computation
A procedure for obtaining extinction coefficients from calculated atomic cross-sections for X-rays, used to define material opacity for radiography [S3, unofficial].
*   **Elemental Opacity:** Computed from tabulated atomic cross-sections $\sigma_E$ (units: barns/atom, 1 barn = $10^{-24}$ cm$^2$):
    $$ \kappa_E = \frac{\sigma_E}{uA} $$
    where $u$ is the atomic mass unit ($1 \text{ amu} = 1.6605402 \times 10^{-24}$ g) and $A$ is the atomic mass of the element in amu [S3, unofficial].
*   **Energy Group Approximation:** Opacity is divided into energy groups $[E_g, E_{g+1}]$. The constant approximation for group $g$ is:
    $$ \mathbf{k}_g = \frac{1}{E_{g+1} - E_g} \int_{E_g}^{E_{g+1}} \mathbf{k}_{E'} dE' $$
    [S3, unofficial]
*   **Mixture Opacity:** For mixtures or compounds, opacity is estimated by atomic mixing:
    $$ \mathbf{K}_{mat,g} = \sum_{i=0}^{N-1} m_i \mathbf{K}_{i,g} $$
    where $m_i$ is the mass fraction of element $i$ [S3, unofficial].
*   **Storage:** Extinction coefficients are stored in a table indexed by material number and energy group number [S3, unofficial].
*   **Constants:** Planck's constant $h = 6.6263 \times 10^{-27}$ erg/s. Photon energy $E = h\nu$. Photon energies are typically given in electron volts (1 eV = $1.60206 \times 10^{-12}$ erg) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Computationally inexpensive; forms the basis for volume compositing [S1, unofficial][S3, unofficial].
*   **Limitations:** Does not account for scattering or emission [S1, unofficial].

## Background
The concepts of transparency and opacity in volume rendering originate from the solution to the radiative transfer equation in the absence of scattering [S1, unofficial]. The differential form $dL/ds = -\kappa L$ describes the loss of radiance due to absorption [S1, unofficial]. This model is fundamental to the "Absorption-only model" and is widely used in medical imaging (radiography) and computer graphics for compositing [S1, unofficial][S3, unofficial].

## Sources
[S1] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S2] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
[S3] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
