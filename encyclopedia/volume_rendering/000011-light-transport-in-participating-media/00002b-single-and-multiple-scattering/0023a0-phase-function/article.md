# Phase Function

Also called: Angular Distribution of Scattering, Scattering Phase Function

## Overview
A phase function describes the angular distribution of light scattering at any point within a participating medium, specifying the fraction of incoming radiance from direction $\boldsymbol{\omega}'$ that is scattered into direction $-\boldsymbol{\omega}$ [S1, unofficial]. It serves as the probability density function (PDF) for the direction of scattered light, ensuring that the total scattered energy is conserved by integrating to unity over the sphere of directions [S4, unofficial]. Unlike [Isotropic Phase Function](5147-isotropic-phase-function.md), which scatters equally in all directions, phase functions can model anisotropic media where light preferentially scatters forward or backward, such as in clouds or fog [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Position $\mathbf{x}$ (optional, for spatially varying media), incident direction $\boldsymbol{\omega}'$ (pointing from $\mathbf{x}$ to the light source), and outgoing direction $\boldsymbol{\omega}$ (pointing from $\mathbf{x}$ to the eye/camera).
*   **Output:** The phase function value $f_p(\mathbf{x}, \boldsymbol{\omega}, \boldsymbol{\omega}')$, a scalar representing the relative probability density of scattering from $\boldsymbol{\omega}'$ to $\boldsymbol{\omega}$.

**Data Structures**
The phase function is typically implemented as a class or struct with two primary methods:
1.  `evaluate(wo, wi)`: Returns the value of the phase function for given directions.
2.  `sample(wo, u)`: Samples a new outgoing direction $\boldsymbol{\omega}$ given an incident direction $\boldsymbol{\omega}'$ and a uniform random sample $u \in [0,1]^2$.

**Notation and Conventions**
*   $\theta$: The angle between the incident direction $\boldsymbol{\omega}'$ and the outgoing direction $\boldsymbol{\omega}$.
*   **Direction Convention:** In standard radiative transfer literature, $\boldsymbol{\omega}$ points from the scattering point to the eye, and $\boldsymbol{\omega}'$ points from the scattering point to the light source [S1, unofficial].
*   **Implementation Warning:** Many rendering engines (e.g., pbrt) use a convention where both incident and outgoing vectors point *away* from the scattering point [S3]. Code must ensure vectors are oriented correctly relative to the chosen convention before computing $\theta = \arccos(\boldsymbol{\omega} \cdot \boldsymbol{\omega}')$ [S1, unofficial].

**Core Properties**
1.  **Reciprocity:** The phase function is symmetric with respect to its direction arguments:
    $$ f_p(\mathbf{x}, \boldsymbol{\omega}, \boldsymbol{\omega}') = f_p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) $$
    This allows the function to be written simply as $f_p(\mathbf{x}, \theta)$ [S1, unofficial].
2.  **Normalization:** The integral over the unit sphere $\mathbb{S}^2$ must equal 1, ensuring no radiance is created or destroyed by the scattering event itself:
    $$ \int_{\mathbb{S}^2} f_p(\mathbf{x}, \boldsymbol{\omega}, \boldsymbol{\omega}') \, d\omega' = 1 $$
    [S1, unofficial][S4, unofficial]

**Common Models**
The choice of phase function depends on the ratio of particle radius $r$ to wavelength $\lambda$ [S1, unofficial].

1.  **Isotropic:**
    $$ f_p(\theta) = \frac{1}{4\pi} $$
    Used when scattering is uniform in all directions [S3].

2.  **Rayleigh Scattering:**
    Used when $r \ll \lambda$ (e.g., smoke, dust, air molecules) [S1, unofficial].
    $$ P_R(\theta) = \frac{3}{4}(1 + \cos^2 \theta) $$
    Produces a two-lobe distribution favoring forward and backward scattering, with zero intensity at $90^\circ$ [S3].

3.  **Mie Scattering:**
    Used when $r \approx \lambda$ (e.g., fog, water droplets) [S1, unofficial].
    *   **Haziness:** $P_{HM}(\theta) = 1 + 9 \left( \frac{1 + \cos \theta}{2} \right)^8$
    *   **Murkiness:** $P_{MM}(\theta) = 1 + 50 \left( \frac{1 + \cos \theta}{2} \right)^{32}$
    [S1, unofficial]

4.  **Henyey-Greenstein (HG):**
    An approximation to Mie scattering, widely used for its efficiency and single asymmetry parameter $g \in [-1, 1]$ [S1, unofficial].
    $$ P_{HG}(\theta) = \frac{1 - g^2}{(1 + g^2 - 2g \cos \theta)^{\frac{3}{2}}} $$
    $g > 0$ favors forward scattering; $g < 0$ favors backward scattering; $g = 0$ is isotropic [S2, unofficial].

5.  **Schlick's Approximation:**
    A computationally cheaper approximation to HG that avoids fractional exponentiation [S1, unofficial].
    $$ P_S(\theta) = \frac{1 - g^2}{(1 - g \cos \theta)^2} $$
    [S1, unofficial]

6.  **Lambertian Phase Function (Geometric Optics):**
    Derived for spherical particles much larger than the wavelength, scattering diffusely [S2, unofficial].
    $$ p(\boldsymbol{\omega}, \boldsymbol{\omega}') = \frac{8}{3\pi} (\sin \alpha + (\pi - \alpha) \cos \alpha) $$
    where $\alpha$ is the angle between directions [S2, unofficial].

**Pseudocode for Evaluation**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def phase_function_henyey_greenstein(cos_theta, g):
    """
    Evaluates the Henyey-Greenstein phase function.
    
    Args:
        cos_theta (float): Cosine of the angle between incident and outgoing directions.
        g (float): Asymmetry parameter, typically in [-1, 1].
        
    Returns:
        float: Phase function value.
    """
    # Guard against division by zero or invalid inputs
    if abs(g) >= 1.0:
        # Limit case: pure forward or backward scattering
        # Not given in sources for exact limit handling, 
        # but typically treated as a delta function or clamped.
        # For implementation, clamp g slightly away from 1.
        g = math.copysign(0.9999, g)
        
    numerator = 1.0 - g * g
    denominator = (1.0 + g * g - 2.0 * g * cos_theta) ** 1.5
    return numerator / denominator

def phase_function_isotropic():
    """
    Returns the constant value for an isotropic phase function.
    """
    return 1.0 / (4.0 * math.pi)
```

## Subtopics

### [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md)
The Henyey-Greenstein (HG) phase function is a parametric model for the angular distribution of light scattering in participating media, specifically designed to handle anisotropic scattering where light is preferentially scattered in forward or backward directions. It solves the problem of efficiently approximating complex scattering behaviors (such as Mie scattering by dust or water droplets) with a single asymmetry parameter $g$. It is distinguished from its sibling, the [Rayleigh Phase Function](3721-rayleigh-phase-function.md), which models isotropic or weakly anisotropic scattering by small particles, by its ability to produce strong forward or backward peaks, making it the standard choice for clouds, haze, and interstellar dust.

### [Rayleigh Phase Function](3721-rayleigh-phase-function.md)
The Rayleigh phase function models the angular distribution of light scattered by particles much smaller than the wavelength of light, such as air molecules in the atmosphere. It produces a characteristic two-lobe distribution that favors forward and backward scattering while exhibiting zero intensity at 90-degree angles. Unlike the isotropic phase function, it introduces anisotropy, and unlike the Henyey-Greenstein function, it is derived from physical dipole radiation principles rather than empirical fitting.

### [Isotropic Phase Function](5147-isotropic-phase-function.md)
An isotropic phase function describes equal scattering in all directions and is independent of either of the two directions. Because phase functions are normalized, there is only one such function.

### Lambertian Phase Function (Geometric Optics)
A simpler phase function derived by geometric optics for a spherical particle much larger than the light wavelength, scattering diffusely by Lambert's law [S2, unofficial]. Its formula is $p(\boldsymbol{\omega}, \boldsymbol{\omega}') = (8/3\pi) (\sin \boldsymbol{\alpha} + (\pi - \boldsymbol{\alpha}) \cos \boldsymbol{\alpha})$, where $\alpha$ is the angle between directions [S2, unofficial].

### Phase Function Interface
The abstract base class defining the interface for phase functions in pbrt. It requires implementations for evaluating the phase function value and sampling a new direction [S3]. The layout includes a class `PhaseFunction` with two pure virtual methods: `virtual Float p(const Vector3f &wo, const Vector3f &wi) const = 0;` and `virtual Float Sample_p(const Vector3f &wo, Vector3f *wi, const Point2f &u) const = 0;` [S3]. A key pitfall is the direction convention: pbrt uses the convention that both the incident direction ($\omega_i$) and the outgoing direction ($\omega_o$) point away from the point where scattering occurs, which differs from scattering literature where the incident direction generally points toward the scattering point [S3].

### In-scattering Model
In-scattering is the increase in radiance along an incoming direction due to scattering from other directions into the current ray's path [S4, unofficial]. The phase function $f_p (\mathbf{x}, \omega, \omega^\prime)$ acts as a probability density function, representing the fraction of the intensity along direction $\omega^\prime$ relative to the normalized integral of scattered intensity at all angles [S4, unofficial]. The normalization condition is $\forall \omega \in \Omega: \; \int_{S^2} f_p (\mathbf{x}, \omega, \omega^\prime) \; d\omega^\prime = 1$ [S4, unofficial]. The differential change in radiance due to in-scattering is $\mathrm{d}L_\mathrm{i} (\mathrm{p}, \omega) = \sigma_{\mathrm{s}} (\mathrm{p}, \omega) \int_{S^2} f_p (\mathrm{p}, \omega, \omega^\prime) L_{\mathrm{i}} (\mathrm{p}, \omega^\prime) \; \mathrm{d} \omega^\prime$, where $L_{\mathrm{i}} (\mathrm{p}, \omega^\prime)$ is the incident radiance from direction $\omega^\prime$ [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Evaluation of standard phase functions (Isotropic, Rayleigh, HG, Schlick) is $O(1)$, involving basic arithmetic operations and potentially a square root or power function [S1, unofficial].
*   **Memory Complexity:** $O(1)$ for parametric models. No storage is required beyond the parameters (e.g., $g$ for HG) [S1, unofficial].
*   **Strengths:** Parametric models like HG are computationally efficient and require minimal memory. They allow for easy tuning of scattering behavior via parameters like $g$ [S1, unofficial].
*   **Limitations:** Parametric models are approximations. They may not accurately represent complex, real-world scattering data (e.g., measured phase functions for specific materials) without significant error [S1, unofficial]. Sampling from anisotropic phase functions (like HG) can be non-trivial and may require rejection sampling or inversion methods, adding overhead compared to isotropic sampling [S1, unofficial].
*   **When to use:** Use HG or Schlick for general-purpose volumetric rendering where performance is critical and the medium is anisotropic (clouds, fog). Use Rayleigh for atmospheric effects with small particles. Use Isotropic for simple, uniform scattering or as a baseline.

## Background
The concept of the phase function originates from radiative transfer theory in physics and atmospheric science [S1, unofficial][S4, unofficial]. It was adapted for computer graphics to model light transport in participating media [S1, unofficial]. The Henyey-Greenstein function was originally developed for neutron transport and later adopted for graphics due to its simplicity [S1, unofficial]. The pbrt implementation of the phase function interface standardizes the evaluation and sampling of these functions in modern path tracers [S3].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S3] 11.2 Phase Functions - pbr-book.org, https://pbr-book.org/3ed-2018/Volume_Scattering/Phase_Functions, fetched 2026-10-27
[S4] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
