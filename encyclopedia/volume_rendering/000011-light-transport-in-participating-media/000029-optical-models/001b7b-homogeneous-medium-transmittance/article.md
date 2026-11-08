# Homogeneous Medium Transmittance

Also called: Beer's Law, Beer-Lambert Law, Beer's Law Transmittance, Beer-Lambert Transmittance

## Overview
Homogeneous Medium Transmittance computes the fraction of radiance that survives transmission through a participating medium of uniform density, attenuating light due to absorption and out-scattering [S1, S5, S6, S7, unofficial]. It solves the problem of determining visibility through a volume by modeling the exponential decay of light intensity with distance [S2, S8, unofficial]. Unlike [Absorption-only model](839-absorption-only-model.md), which considers only energy conversion to heat, this method includes out-scattering in the extinction term, making it the standard for general volumetric attenuation [S2, S5, unofficial]. It serves as the foundational transmittance calculation for [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) and [Emission-Absorption Ray Marching](3171-emission-absorption-ray-marching.md) [S3, S7, unofficial].

## How it works
**Inputs:**
*   $\sigma_t$: Total extinction coefficient ($m^{-1}$), constant throughout the medium [S2, S6, unofficial].
*   $d$: Geometric distance between the entry point (Point A) and exit point (Point B) along the ray ($m$) [S6, S8, unofficial].

**Outputs:**
*   $T$: Transmittance, a scalar in $[0, 1]$ representing the fraction of incident radiance transmitted [S2, S6, unofficial].

**Data Structures:**
No complex data structures are required for the homogeneous case; the extinction coefficient is a single scalar value [S6, unofficial].

**Steps:**
1.  Calculate the optical depth $\tau$ by multiplying the total extinction coefficient by the distance:
    $$ \tau = \sigma_t \cdot d $$
2.  Compute the transmittance using the exponential function:
    $$ T = e^{-\tau} $$
    Alternatively, expressed directly:
    $$ T = e^{-\sigma_t d} $$
    [S2, S6, S8, unofficial]

**Parameters:**
*   $\sigma_t$: The sum of the absorption coefficient $\sigma_a$ and the scattering coefficient $\sigma_s$ ($\sigma_t = \sigma_a + \sigma_s$) [S2, S8, unofficial].
*   Typical values: In demos, an absorption coefficient $\alpha$ (used as $\sigma_t$) of 0.9 is used for cloud rendering [S7, unofficial].

**Pitfalls:**
*   **Lighting Interaction:** Using only Beer's Law for transmittance without a proper lighting model (in-scattering) results in a "white blob" appearance because the volume does not interact directionally with light sources [S7, unofficial].
*   **Scattering Accounting:** Scattering effects are only accounted for in the value of the total mass extinction coefficient; light losses are assumed to be from both absorption and scattering [S4, unofficial].
*   **Boundary Checks:** When implementing ratio tracking or step-based approximations, changing transmittance parameters can affect the number of steps required to exit the volume bounding box if boundary checks are performed per step [S13, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def homogeneous_transmittance(sigma_t: float, distance: float) -> float:
    """
    Computes the transmittance of light through a homogeneous medium.
    
    Args:
        sigma_t: Total extinction coefficient (absorption + out-scattering).
        distance: Geometric distance traveled through the medium.
        
    Returns:
        Transmittance T in [0, 1].
    """
    if distance < 0.0:
        return 1.0 # No distance, full transmission [G]
    
    optical_depth = sigma_t * distance
    transmittance = math.exp(-optical_depth)
    return transmittance
```

## Subtopics

### [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md)
Exponential fog simulates atmospheric scattering by fading scene colors toward a background color based on viewer distance, using the Beer-Lambert law for physically plausible decay. It differs from linear distance fog by using an exponential function and from heterogeneous models by assuming uniform density. Choose this for simple atmospheric depth cues.

### [Absorption Adjustment](4020-absorption-adjustment.md)
Absorption Adjustment corrects the opacity parameter $\alpha$ in layered smoke rendering to account for non-orthogonal viewing angles and variable plane spacing. It differs from standard homogeneous transmittance by dynamically scaling the absorption coefficient based on geometric projection factors rather than assuming a fixed path length. Choose this when rendering slice-based smoke volumes with varying view angles.

### [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md)
The Non-Exponential Free Flight Model simulates light transport in homogeneous media with spatially correlated particles, resulting in free path distributions with longer tails than standard exponential decay. Unlike [Exponential Fog](3968-exponential-fog-beer-lambert.md), it introduces "memory" into photon transport for artistic control over light bleeding. Choose this for subsurface scattering or volumetric effects with natural clustering/voids.

### [Heterogeneous Medium Transmittance — Beer's Law](4793-heterogeneous-medium-transmittance-beer-s-law.md)
This method computes transmittance through media where the extinction coefficient varies spatially by integrating $\sigma_t$ along the ray path. Unlike the homogeneous case, it handles spatially varying density by evaluating $\tau = \int_a^b \sigma_t(s) ds$. It serves as the foundation for [Exponential Fog](3968-exponential-fog-beer-lambert.md) and [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md).

### Sun Transmittance Marching
A raymarching technique that calculates sunlight transmittance to a sample point by integrating light density along a sun-directed ray using a simplified Beer-Lambert equation [S10, unofficial]. It initializes light density to 0, loops through steps, accumulates density from a noise texture, and computes transmittance as $\exp(-\text{lightDensity} \cdot \text{AbsorptionCoefficient})$ [S10, unofficial]. Typical parameters: stepSize=20.0, stepLimit=200 [S10, unofficial].(Note: No dedicated article exists, described here per instructions)

### Beer-Lambert Extinction with Powder Sugar Effect
An empirical correction for standard Beer-Lambert attenuation to prevent unrealistic darkening of cloud centers by accounting for intense forward multiple-scattering [S11, unofficial]. It modifies transmittance to $T_{\text{effective}}(\tau) = \exp(-\tau) \cdot \max(1.0 - \exp(-2 \tau \cdot \text{powderFactor}), 0.05)$ [S11, unofficial]. The powderFactor defaults to 1.0 [S11, unofficial].(Note: No dedicated article exists, described here per instructions)

### Maximum-Value Projection Shading Handling
For maximum-value projections, shading effects except transmission are turned off, with the index of refraction set to 1 and transmission coefficient set to $1 - O_1$ (where $O_1$ is the max-value opacity) [S12, unofficial]. To avoid recursive intersections, the cell is turned "off" during transmission ray casting and "on" after intersection [S12, unofficial].(Note: No dedicated article exists, described here per instructions)

### Transmittance Calculation via Ratio Tracking
Transmittance represents the fraction of energy reaching a point, decreasing exponentially based on extinction [S13, unofficial]. It uses ratio tracking, multiplying the transmittance distance by a `transmittance depth` parameter (range 0.1 to 10) to adjust attenuation without changing density [S13, unofficial]. Changing this parameter affects the number of steps to exit the bounding box [S13, unofficial].(Note: No dedicated article exists, described here per instructions)

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the closed-form homogeneous solution [S6, unofficial]. For heterogeneous approximations (marching), it is $O(N)$ where $N$ is the number of samples [S7, unofficial].
*   **Memory Complexity:** $O(1)$ for homogeneous; $O(N)$ for heterogeneous if storing density samples [S7, unofficial].
*   **Strengths:** Physically based, computationally inexpensive for uniform media, forms the basis for most volumetric visibility calculations [S2, S7, unofficial].
*   **Limitations:** Assumes uniform density; does not account for in-scattering (light gained) or directional lighting interactions on its own [S7, S11, unofficial].
*   **When to use:** Uniform fog, simple smoke attenuation, or as the transmittance component in more complex volume rendering integrals [S6, S9, unofficial].
*   **When not to use:** Heterogeneous media (use [Heterogeneous Medium Transmittance](4793-heterogeneous-medium-transmittance-beer-s-law.md)), or when accurate in-scattering/lighting interaction is required without additional models [S7, unofficial].

## Background
The Beer-Lambert law describes the radiance change as a first-order homogeneous linear ordinary differential equation [S2, unofficial]. It was originally formulated to describe light absorption in chemical solutions and has been adapted for computer graphics to model light diminution through participating media [S2, S8, unofficial]. It is widely used in volume rendering for fog, smoke, and cloud shading, often combined with ray marching techniques [S7, S10, unofficial].

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S3] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S4] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S5] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S6] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S7] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S8] Volumetric Lighting (God Rays) — How It Works - Moonjump, https://moonjump.com/game-dev-mechanics-volumetric-lighting-god-rays-how-it-works/, fetched 2026-10-28 (unofficial)
[S9] 体渲染简明教程 - BimAnt, https://bimant.com/blog/volume-rendering-brief-intro/, fetched 2026-10-30 (unofficial)
[S10] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S11] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S12] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S13] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
