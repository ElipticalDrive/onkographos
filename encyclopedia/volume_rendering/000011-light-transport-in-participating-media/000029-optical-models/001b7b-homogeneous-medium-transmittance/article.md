# Homogeneous Medium Transmittance

Also called: Beer's Law, Beer's Law Transmittance, Beer-Lambert Law, Beer-Lambert Transmittance, Direct Transmission, Exponential Attenuation

## Overview
Homogeneous Medium Transmittance computes the fraction of radiance that survives transmission through a participating medium with a constant extinction coefficient [S1, S2, S6, S8, unofficial]. It solves the problem of light attenuation (absorption and out-scattering) along a straight path, serving as the fundamental building block for opacity calculation in volume rendering [S5, S8, unofficial]. It differs from [Heterogeneous Medium Transmittance](4793-heterogeneous-medium-transmittance-beer-s-law.md) by assuming a spatially uniform medium, allowing the optical depth to be calculated via simple multiplication rather than integration [S1, S2, S6, unofficial]. It is the core transmittance model used by [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) and [Absorption-only model](839-absorption-only-model.md) [S1, S3, S7, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Entry point $\mathbf{p}_{in}$, exit point $\mathbf{p}_{out}$, total extinction coefficient $\sigma_t$ (constant scalar).
*   **Output:** Transmittance $T$, a scalar in $[0, 1]$ representing the ratio of outgoing radiance to incoming radiance.

**Data Structures**
No complex data structures are required. The method operates on scalar values. If implemented within a ray-marching loop (as in [Emission-Absorption Ray Marching](3171-emission-absorption-ray-marching.md)), it accumulates transmittance across discrete steps.

**Formulas**
The differential change in radiance $L(s)$ along a path $s$ in a homogeneous medium is governed by the first-order linear ordinary differential equation [S2, unofficial]:
$$ \frac{dL(s)}{ds} = -\sigma_t L(s) $$
where $\sigma_t = \sigma_a + \sigma_s$ is the total extinction coefficient, sum of absorption $\sigma_a$ and scattering $\sigma_s$ [S2, S8, unofficial].

The solution to this ODE yields the transmittance $T$ over a distance $d$ between two points:
$$ T = e^{-\sigma_t d} $$
where $d$ is the distance traveled along the ray [S2, S6, S8, unofficial].

Alternatively, using optical depth $\tau = \sigma_t d$:
$$ T = e^{-\tau} $$
[S3, S8, unofficial].

**Parameters**
*   $\sigma_t$: Total extinction coefficient. Units: $m^{-1}$ or density units depending on implementation. Typical values vary by medium; e.g., 0.9 for cloud demos [S7, unofficial].
*   $d$: Geometric distance.

**Algorithm (Ray Marching Context)**
When integrating transmittance through a volume discretized into steps (e.g., for [Emission-Absorption Volume Rendering](4040-emission-absorption-volume-rendering.md)), the method is applied iteratively [S7, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_transmittance(extinction_coefficient, distance):
    """Computes transmittance T = exp(-sigma_t * d) for a homogeneous medium."""
    return exp(-extinction_coefficient * distance)

def raymarch_homogeneous(ray_origin, ray_direction, step_size, max_steps, extinction_coefficient):
    """
    Accumulates transmittance and emitted light through a homogeneous medium.
    Based on [S7, unofficial].
    """
    total_transmittance = 1.0
    accumulated_light_energy = 0.0
    current_pos = ray_origin

    for _ in range(max_steps):
        # For homogeneous media, density is constant; here assumed 1.0
        density = 1.0

        if density > 0.0:
            step_transmittance = compute_transmittance(extinction_coefficient, step_size)
            total_transmittance *= step_transmittance
            luminance = density
            accumulated_light_energy += total_transmittance * luminance

        if total_transmittance < 1e-6:
            break

        current_pos = current_pos + ray_direction * step_size

    return accumulated_light_energy
```

**Edge Cases and Pitfalls**
*   **Numerical Underflow:** For large optical depths ($\tau \gg 1$), $e^{-\tau}$ underflows to 0, correctly representing a fully opaque medium [S8, unofficial].
*   **Directional Lighting:** Using only Beer's Law without a proper lighting model (in-scattering) results in a "white blob" in the center of clouds because the cloud does not interact with light directionally [S7, unofficial].
*   **Scattering Assumption:** In many implementations, scattering effects are folded into the extinction coefficient $\sigma_t$, assuming light loss is from both absorption and out-scattering [S4, S5, unofficial].

## Subtopics

### [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md)
Exponential fog simulates atmospheric scattering by fading scene colors toward a background color based on viewer distance. It uses the Beer-Lambert law for physically plausible decay, differing from linear fog by its exponential nature, and from heterogeneous models by assuming uniform density.

### [Absorption Adjustment](4020-absorption-adjustment.md)
Corrects opacity $\alpha$ in layered smoke rendering for non-orthogonal viewing angles and variable plane spacing [4020-absorption-adjustment.md]. It dynamically scales the absorption coefficient based on geometric projection factors, unlike standard homogeneous transmittance which assumes a fixed path length [4020-absorption-adjustment.md].

### [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md)
Simulates light transport in homogeneous media with spatially correlated particles, resulting in free path distributions with longer tails than standard exponential decay. It introduces "memory" into photon transport for artistic control over light bleeding, differing from the uncorrelated assumption of [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md).

### [Heterogeneous Medium Transmittance — Beer's Law](4793-heterogeneous-medium-transmittance-beer-s-law.md)
Calculates the fraction of radiance transmitted between two points through a participating medium where the extinction coefficient varies spatially. It generalizes the homogeneous case by integrating the extinction coefficient along the ray path to compute optical depth, then applying the exponential attenuation model. Unlike [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md), which assumes a constant extinction coefficient, this method handles spatially varying density and composition, serving as the foundational attenuation term for light transport in complex participating media.

### [Maximum-Value Projection Shading Handling](6358-maximum-value-projection-shading-handling.md)
In maximum-value projections, shading effects (except transmission) are turned off, and the index of refraction is set to 1. Cells are temporarily turned "off" during transmission ray casting to avoid recursive intersections within the same object.

### [Transmittance Calculation via Ratio Tracking](7102-transmittance-calculation-via-ratio-tracking.md)
Ratio Tracking is an unbiased Monte Carlo estimator for the transmittance function $T(x_i, x_{i+1})$ in heterogeneous media, proposed by Novák et al. [2014]. It is used within Pseudo-Marginal Metropolis Light Transport (PMLT) to provide a noisy but unbiased estimate of transmission where exact integration is intractable. Unlike deterministic methods such as [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) which assume homogeneous density, Ratio Tracking handles spatially varying extinction coefficients by sampling distances and evaluating the ratio of samples that exceed the target distance.

### Sun Transmittance Marching
A raymarching technique calculating sunlight transmittance to a sample point by integrating light density along a sun-directed ray using a simplified Beer-Lambert equation [S12, unofficial]. It samples a 3D noise texture for density and accumulates light density, applying `exp(-lightDensity * absorptionCoefficient)` for final transmittance [S12, unofficial]. Parameters include `stepSize` (20.0) and `stepLimit` (200) [S12, unofficial].

### Beer-Lambert Extinction with Powder Sugar Effect
An empirical correction preventing unrealistic darkening of cloud centers by accounting for intense forward multiple-scattering near boundaries [S13, unofficial]. It modifies standard transmittance: $T_{\text{effective}}(\tau) = \exp(-\tau) \cdot \max(1.0 - \exp(-2 \tau \cdot \text{powderFactor}), 0.05)$, where `powderFactor` defaults to 1.0 [S13, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for a single segment. $O(N)$ for ray marching with $N$ steps, where each step requires one exponential evaluation [S7, unofficial].
*   **Memory Complexity:** $O(1)$, requiring only scalar storage for transmittance and optical depth [S2, unofficial].
*   **Strengths:** Physically accurate for uniform media; computationally inexpensive; foundation for more complex models [S7, S8, unofficial].
*   **Limitations:** Cannot handle spatially varying density (requires [Heterogeneous Medium Transmittance](4793-heterogeneous-medium-transmittance-beer-s-law.md)); ignores in-scattering (requires [Single Scattering Model](1053-single-scattering-model.md) or RTE); assumes straight-line paths [S2, S5, unofficial].
*   **When to use:** Uniform fog, smoke, or clouds where density is constant; as a component in ray marching for emission-absorption models [S7, S9, unofficial].
*   **When not to use:** Heterogeneous media, media with significant in-scattering effects, or when artistic control over particle clustering is needed ([Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md)) [S2, S11, unofficial].

## Background
The Beer-Lambert law originates from the work of August Beer and Johann Heinrich Lambert, describing the attenuation of light through a medium [S2, S8, unofficial]. In computer graphics, it was adopted as the standard model for volumetric attenuation, replacing earlier "hacky" transparency hacks with a physically based exponential decay model [S7, unofficial]. It is a special case of the Radiative Transfer Equation (RTE) where the source term (in-scattering) is zero and the extinction coefficient is constant [S2, S8, unofficial]. It is widely used in real-time rendering for fog, clouds, and smoke [S1, S7, S10, unofficial].

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
[S12] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S13] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
