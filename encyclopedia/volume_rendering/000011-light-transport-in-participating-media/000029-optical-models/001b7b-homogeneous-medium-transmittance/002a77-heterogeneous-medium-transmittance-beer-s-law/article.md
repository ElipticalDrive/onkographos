# Heterogeneous Medium Transmittance — Beer's Law

Also called: Beam Transmittance, Beer's Law (Heterogeneous), Beer's Law with Optical Depth, Beer-Lambert Law, General Transmittance Equation, Optical Depth and Transparency, Transmittance (Beer-Lambert Law)

## Overview
Heterogeneous Medium Transmittance calculates the fraction of radiance transmitted between two points through a participating medium where the extinction coefficient varies spatially [S4, unofficial]. It generalizes the homogeneous case by integrating the extinction coefficient along the ray path to compute optical depth, then applying the exponential attenuation model [S1, unofficial]. Unlike [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md), which assumes a constant extinction coefficient, this method handles spatially varying density and composition, serving as the foundational attenuation term for light transport in complex participating media [S4, unofficial].

## How it works
**Inputs:**
- Start point $\mathrm{p}$ (3D position) [S4, unofficial].
- End point $\mathrm{p}^\prime$ (3D position) [S4, unofficial].
- Extinction coefficient function $\sigma_{\mathrm{t}}(\mathbf{x}, \omega)$, which returns the total extinction coefficient at position $\mathbf{x}$ for direction $\omega$ [S4, unofficial]. This function may depend on position and direction, though often $\omega$ is fixed for a single ray segment [G].

**Outputs:**
- Transmittance $\mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime)$: A scalar in $[0, 1]$ representing the fraction of light surviving attenuation [S4, unofficial].
- Optical depth $\tau (\mathrm{p} \to \mathrm{p}^\prime)$: A scalar representing the integrated extinction along the path [S1, unofficial].

**Data Structures:**
- Not given in the sources. Implementation typically involves a numerical integration routine over the ray segment [G]. The extinction coefficient function $\sigma_{\mathrm{t}}$ is usually accessed via a 3D texture, voxel grid, or analytical function [G].

**Steps:**
1. Calculate the ray direction $\omega$ and total distance $d$:
   $$ \omega = \frac{\mathrm{p}^\prime - \mathrm{p}}{\lVert \mathrm{p}^\prime - \mathrm{p} \rVert} $$
   $$ d = \lVert \mathrm{p}^\prime - \mathrm{p} \rVert $$
   If $d = 0$, return $\mathcal{T} = 1$ and $\tau = 0$ [G].

2. Define the position along the ray at distance $t$ from $\mathrm{p}$:
   $$ \mathbf{x}(t) = \mathrm{p} + t \omega, \quad t \in [0, d] $$
   [S4, unofficial].

3. Compute the optical depth $\tau$ by integrating the extinction coefficient along the ray:
   $$ \tau (\mathrm{p} \to \mathrm{p}^\prime) = \int_0^d \sigma_{\mathrm{t}} (\mathbf{x}(t), \omega) \; \mathrm{d}t $$
   [S4, unofficial]. In practice, this integral is approximated numerically [G]. For a discrete set of samples $t_i$ with spacing $\Delta t$, a simple Riemann sum or trapezoidal rule is used:
   $$ \tau \approx \sum_{i} \sigma_{\mathrm{t}} (\mathbf{x}(t_i), \omega) \Delta t_i $$
   [G].

4. Compute the transmittance using the exponential of the negative optical depth:
   $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) = \exp \left( - \tau (\mathrm{p} \to \mathrm{p}^\prime) \right) $$
   [S4, unofficial].

**Formulas:**
- **General Transmittance:**
  $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) \equiv \exp \left( - \int_0^d \sigma_{\mathrm{t}} (\mathrm{p} + \omega t, \omega) \; \mathrm{d}t \right) $$
  where $d = \lVert \mathrm{p}^\prime - \mathrm{p} \rVert$ and $\omega = (\mathrm{p}^\prime - \mathrm{p}) / d$ [S4, unofficial].

- **Optical Depth:**
  $$ \tau (\mathrm{p} \to \mathrm{p}^\prime) \equiv \int_0^d \sigma_{\mathrm{t}} (\mathrm{p} + \omega t, \omega) \; \mathrm{d}t $$
  [S4, unofficial]. This is also written as $\tau = \int_{s=0}^d \sigma_t(x_s) ds$ where $x_s = x + s\omega$ [S1, unofficial].

- **Extinction Coefficient Composition:**
  The total extinction coefficient $\sigma_{\mathrm{t}}$ combines absorption $\sigma_a$ and out-scattering $\sigma_s$:
  $$ \sigma_{\mathrm{t}} = \sigma_a + \sigma_s $$
  [S5, unofficial]. Both absorption and out-scattering cause attenuation of the beam, and they are combined into a single term unless separate tracking is required [S5, unofficial].

- **Multiplicative Property:**
  Transmittance is multiplicative along a ray:
  $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^{\prime \prime}) = \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) \mathcal{T} (\mathrm{p}^\prime \to \mathrm{p}^{\prime \prime}) $$
  [S4, unofficial]. This follows from the linearity of the integral in the exponent [G].

**Parameters:**
- $\tau$: Optical depth (or optical thickness). Not given in the sources as a fixed parameter, but a computed quantity [S1, unofficial].
- $d$: Distance traveled by the ray through the volume [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Distance:** If $\mathrm{p} = \mathrm{p}^\prime$, $d=0$, transmittance is 1 [G].
- **Precision:** For very large optical depths, $\exp(-\tau)$ may underflow to 0. This is physically correct (opaque medium) [G].
- **Numerical Integration:** The accuracy of $\tau$ depends on the sampling density relative to the variation of $\sigma_{\mathrm{t}}$ [G]. If $\sigma_{\mathrm{t}}$ varies rapidly, insufficient sampling leads to inaccurate transmittance [G].
- **Direction Dependence:** The extinction coefficient $\sigma_{\mathrm{t}}$ may depend on direction $\omega$ (anisotropic media) [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def heterogeneous_transmittance(p_start, p_end, sigma_t_func, num_samples=100):
    """
    Computes transmittance through a heterogeneous medium using Beer's Law.
    
    Args:
        p_start: 3D vector, start point of the ray.
        p_end: 3D vector, end point of the ray.
        sigma_t_func: Function that takes (position, direction) and returns 
                      the total extinction coefficient sigma_t.
        num_samples: Number of integration samples. Not given in the sources.
    
    Returns:
        T: Transmittance (scalar in [0, 1]).
        tau: Optical depth (scalar).
    """
    # Step 1: Calculate direction and distance
    direction = p_end - p_start
    d = length(direction)
    
    # Edge case: zero distance
    if d == 0:
        return 1.0, 0.0
    
    # Normalize direction
    omega = direction / d
    
    # Step 2: Numerical integration of optical depth (midpoint rule)
    dt = d / num_samples
    tau = 0.0
    
    for i in range(num_samples):
        # Sample at the midpoint of each interval
        t = (i + 0.5) * dt
        position = p_start + t * omega
        
        # Evaluate extinction coefficient at this position and direction
        sigma_t = sigma_t_func(position, omega)
        
        # Accumulate optical depth
        tau += sigma_t * dt
    
    # Step 3: Compute transmittance
    T = exp(-tau)
    
    return T, tau
```

## Subtopics

### Spectral Volume Rendering (Lambert-Bouguer Basis)
This variant treats light as electromagnetic radiation with a spectral power distribution $F(\lambda)$, typically limited to the visible spectrum $\lambda \in [380\text{nm}, 770\text{nm}]$ or $[400\text{nm}, 700\text{nm}]$ [S6, book]. It extends Beer's Law to wavelength-dependent extinction [S6, book]. For a homogeneous isotropic absorption filter, transmission follows Lambert's law: $L(\lambda) = L_0(\lambda) \cdot \exp(-s \cdot \tau(\lambda))$, where $\tau(\lambda)$ is the spectral extinction coefficient [S6, book]. Beer's extension for a liquid solution with solute concentration $\nu$ and solute extinction coefficient $\kappa(\lambda)$ is $L(\lambda) = L_0(\lambda) \cdot \exp(-s \cdot \nu \cdot \kappa(\lambda))$ [S6, book]. For depth correction, if transparency $T(\lambda, \Delta)$ is known for a standard path length $\Delta$, the transparency for an arbitrary path length $\delta$ is $T(\lambda, \delta) = T(\lambda, \Delta)^{\frac{\delta}{\Delta}}$, which can be rewritten for opacity as $\alpha(\lambda, \delta) = 1 - (1 - \alpha(\lambda, \Delta))^{\frac{\delta}{\Delta}}$ [S6, book]. This is chosen when color accuracy and wavelength-dependent attenuation are critical [G].

### Extinction Coefficient Calculation
This variant computes the spatially varying extinction coefficient from particle properties [S2]. The extinction coefficient $\beta_{ex}$ is the probability per meter that light intersects a particle, calculated as particle density times extinction cross-section [S2]. At a point in space $\tilde{s}$, it is calculated as $\beta_{ex}(\tilde{s}) = \eta(\tilde{s})\sigma_{ex}$, where $\eta$ is the particle concentration in particles per cubic meter and $\sigma_{ex}$ is the extinction cross-section [S2]. For multiple particle fields, the extinction coefficients linearly combine: $\sum_{\text{all fields } i} \beta_{ex}^{(i)}(\tilde{s})$ [S2]. This is used when simulating particulate media like clouds or fog where microphysical properties are known [G].

## Operating characteristics
- **Time Complexity:** Depends on the numerical integration method. For $N$ samples along the ray, it is $O(N)$ evaluations of $\sigma_{\mathrm{t}}$ [G].
- **Memory Complexity:** $O(1)$ additional memory if $\sigma_{\mathrm{t}}$ is accessed on-the-fly (e.g., from a texture or analytical function) [G]. If precomputed optical depth is stored, it depends on the volume resolution [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with ray length and sampling density [G].
- **Strengths:** Physically accurate for heterogeneous media; generalizes homogeneous case; multiplicative property allows efficient composition of transmittance over multiple segments [G].
- **Limitations:** Requires evaluation of $\sigma_{\mathrm{t}}$ at multiple points; numerical integration errors if sampling is insufficient; does not account for in-scattering (only attenuation) [G].
- **When to use:** When the medium's extinction coefficient varies spatially and in-scattering is handled separately or ignored [G].
- **When not to use:** When the medium is homogeneous (use [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md) for efficiency); when in-scattering is significant and needs to be coupled with transmittance in a single step (requires full RTE solution) [G].

## Background
The Beer-Lambert law originates from the work of Bouguer (1727) and Lambert (1760) on absorption in filters, and Beer extended it to liquid solutions [S6, book]. In computer graphics, it is a fundamental component of the volume rendering equation, describing the attenuation of light as it travels through participating media [G]. It is widely used in offline renderers and real-time volume rendering for simulating fog, smoke, clouds, and biological tissues [G].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
[S3] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S4] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S5] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
[S6] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
