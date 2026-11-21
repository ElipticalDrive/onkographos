# Heterogeneous Medium Transmittance — Beer's Law

Also called: Beam Transmittance, Beer's Law (Heterogeneous), Beer's Law with Optical Depth, Beer-Lambert Law, General Transmittance Equation, Transmittance (Beer-Lambert Law)

## Overview
Heterogeneous Medium Transmittance calculates the fraction of radiance transmitted between two points through a participating medium where the extinction coefficient varies spatially. It generalizes the homogeneous case by integrating the extinction coefficient along the ray path to compute optical depth, then applying the exponential attenuation model. Unlike [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md), which assumes a constant extinction coefficient, this method handles spatially varying density and composition, serving as the foundational attenuation term for light transport in complex participating media.

## How it works
**Inputs:**
- Start point $\mathrm{p}$ (3D position).
- End point $\mathrm{p}^\prime$ (3D position).
- Extinction coefficient function $\sigma_{\mathrm{t}}(\mathbf{x}, \omega)$, which returns the total extinction coefficient at position $\mathbf{x}$ for direction $\omega$. This function may depend on position and direction, though often $\omega$ is fixed for a single ray segment.

**Outputs:**
- Transmittance $\mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime)$: A scalar in $[0, 1]$ representing the fraction of light surviving attenuation.
- Optical depth $\tau (\mathrm{p} \to \mathrm{p}^\prime)$: A scalar representing the integrated extinction along the path.

**Data Structures:**
- No specific complex data structures are required for the definition itself, but implementation typically involves a numerical integration routine (e.g., trapezoidal rule, Simpson's rule, or adaptive quadrature) over the ray segment. The extinction coefficient function $\sigma_{\mathrm{t}}$ is usually accessed via a 3D texture, voxel grid, or analytical function.

**Steps:**
1. Calculate the ray direction $\omega$ and total distance $d$:
   $$ \omega = \frac{\mathrm{p}^\prime - \mathrm{p}}{\lVert \mathrm{p}^\prime - \mathrm{p} \rVert} $$
   $$ d = \lVert \mathrm{p}^\prime - \mathrm{p} \rVert $$
   If $d = 0$, return $\mathcal{T} = 1$ and $\tau = 0$.

2. Define the position along the ray at distance $t$ from $\mathrm{p}$:
   $$ \mathbf{x}(t) = \mathrm{p} + t \omega, \quad t \in [0, d] $$

3. Compute the optical depth $\tau$ by integrating the extinction coefficient along the ray:
   $$ \tau (\mathrm{p} \to \mathrm{p}^\prime) = \int_0^d \sigma_{\mathrm{t}} (\mathbf{x}(t), \omega) \; \mathrm{d}t $$
   In practice, this integral is approximated numerically. For a discrete set of samples $t_i$ with spacing $\Delta t$, a simple Riemann sum or trapezoidal rule is used:
   $$ \tau \approx \sum_{i} \sigma_{\mathrm{t}} (\mathbf{x}(t_i), \omega) \Delta t_i $$

4. Compute the transmittance using the exponential of the negative optical depth:
   $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) = \exp \left( - \tau (\mathrm{p} \to \mathrm{p}^\prime) \right) $$

**Formulas:**
- **General Transmittance:**
  $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) \equiv \exp \left( - \int_0^d \sigma_{\mathrm{t}} (\mathrm{p} + \omega t, \omega) \; \mathrm{d}t \right) $$
  where $d = \lVert \mathrm{p}^\prime - \mathrm{p} \rVert$ and $\omega = (\mathrm{p}^\prime - \mathrm{p}) / d$ [S3, unofficial].

- **Optical Depth:**
  $$ \tau (\mathrm{p} \to \mathrm{p}^\prime) \equiv \int_0^d \sigma_{\mathrm{t}} (\mathrm{p} + \omega t, \omega) \; \mathrm{d}t $$
  [S3, unofficial]. This is also written as $\tau = \int_{s=0}^d \sigma_t(x_s) ds$ where $x_s = x + s\omega$ [S1, unofficial].

- **Extinction Coefficient Composition:**
  The total extinction coefficient $\sigma_{\mathrm{t}}$ combines absorption $\sigma_a$ and out-scattering $\sigma_s$:
  $$ \sigma_{\mathrm{t}} = \sigma_a + \sigma_s $$
  [S4, unofficial]. Both absorption and out-scattering cause attenuation of the beam, and they are combined into a single term unless separate tracking is required [S4, unofficial].

- **Multiplicative Property:**
  Transmittance is multiplicative along a ray:
  $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^{\prime \prime}) = \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) \mathcal{T} (\mathrm{p}^\prime \to \mathrm{p}^{\prime \prime}) $$
  [S3, unofficial]. This follows from the linearity of the integral in the exponent.

**Parameters:**
- $\tau$: Optical depth (or optical thickness). Not given in the sources as a fixed parameter, but a computed quantity.
- $d$: Distance traveled by the ray through the volume.

**Edge Cases and Pitfalls:**
- **Zero Distance:** If $\mathrm{p} = \mathrm{p}^\prime$, $d=0$, transmittance is 1.
- **Precision:** For very large optical depths, $\exp(-\tau)$ may underflow to 0. This is physically correct (opaque medium).
- **Numerical Integration:** The accuracy of $\tau$ depends on the sampling density relative to the variation of $\sigma_{\mathrm{t}}$. If $\sigma_{\mathrm{t}}$ varies rapidly, insufficient sampling leads to inaccurate transmittance.
- **Direction Dependence:** The extinction coefficient $\sigma_{\mathrm{t}}$ may depend on direction $\omega$ (anisotropic media). The formula accounts for this by evaluating $\sigma_{\mathrm{t}}(\mathbf{x}, \omega)$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def heterogeneous_transmittance(p_start, p_end, sigma_t_func):
    """
    Computes transmittance through a heterogeneous medium using Beer's Law.
    
    Args:
        p_start: 3D vector, start point of the ray.
        p_end: 3D vector, end point of the ray.
        sigma_t_func: Function that takes (position, direction) and returns 
                      the total extinction coefficient sigma_t.
    
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
    
    # Step 2: Numerical integration of optical depth
    # Note: The specific integration method (e.g., number of samples, 
    # quadrature rule) is not specified in the sources. 
    # A simple midpoint rule with N samples is provided as a common implementation.
    N = 100  # Number of samples. Not given in the sources.
    tau = 0.0
    dt = d / N
    
    for i in range(N):
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
This variant treats light as electromagnetic radiation with a spectral power distribution $F(\lambda)$, typically limited to the visible spectrum $\lambda \in [380\text{nm}, 770\text{nm}]$ or $[400\text{nm}, 700\text{nm}]$ [S5, book]. It extends Beer's Law to wavelength-dependent extinction. For a homogeneous isotropic absorption filter, transmission follows Lambert's law: $L(\lambda) = L_0(\lambda) \cdot \exp(-s \cdot \tau(\lambda))$, where $\tau(\lambda)$ is the spectral extinction coefficient [S5, book]. Beer's extension for a liquid solution with solute concentration $\nu$ and solute extinction coefficient $\kappa(\lambda)$ is $L(\lambda) = L_0(\lambda) \cdot \exp(-s \cdot \nu \cdot \kappa(\lambda))$ [S5, book]. For depth correction, if transparency $T(\lambda, \Delta)$ is known for a standard path length $\Delta$, the transparency for an arbitrary path length $\delta$ is $T(\lambda, \delta) = T(\lambda, \Delta)^{\frac{\delta}{\Delta}}$, which can be rewritten for opacity as $\alpha(\lambda, \delta) = 1 - (1 - \alpha(\lambda, \Delta))^{\frac{\delta}{\Delta}}$ [S5, book]. This is chosen when color accuracy and wavelength-dependent attenuation are critical.

## Operating characteristics
- **Time Complexity:** Depends on the numerical integration method. For $N$ samples along the ray, it is $O(N)$ evaluations of $\sigma_{\mathrm{t}}$.
- **Memory Complexity:** $O(1)$ additional memory if $\sigma_{\mathrm{t}}$ is accessed on-the-fly (e.g., from a texture or analytical function). If precomputed optical depth is stored, it depends on the volume resolution.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with ray length and sampling density.
- **Strengths:** Physically accurate for heterogeneous media; generalizes homogeneous case; multiplicative property allows efficient composition of transmittance over multiple segments.
- **Limitations:** Requires evaluation of $\sigma_{\mathrm{t}}$ at multiple points; numerical integration errors if sampling is insufficient; does not account for in-scattering (only attenuation).
- **When to use:** When the medium's extinction coefficient varies spatially and in-scattering is handled separately or ignored.
- **When not to use:** When the medium is homogeneous (use [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md) for efficiency); when in-scattering is significant and needs to be coupled with transmittance in a single step (requires full RTE solution).

## Background
The Beer-Lambert law originates from the work of Bouguer (1727) and Lambert (1760) on absorption in filters, and Beer (1983) extended it to liquid solutions [S5, book]. In computer graphics, it is a fundamental component of the volume rendering equation, describing the attenuation of light as it travels through participating media. The general form for heterogeneous media is derived from the radiative transfer equation (RTE) [S1, S3, unofficial]. It is widely used in offline renderers and real-time volume rendering for simulating fog, smoke, clouds, and biological tissues.

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S3] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S4] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
[S5] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
