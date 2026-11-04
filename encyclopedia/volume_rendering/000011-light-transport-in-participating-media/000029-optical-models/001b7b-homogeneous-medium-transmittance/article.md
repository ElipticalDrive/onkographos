# Homogeneous Medium Transmittance

Also called: Beer's Law, Beer's Law Transmittance, Beer-Lambert Law, Beer-Lambert Transmittance, Beer-Lambert voxel absorption, Ray Transmittance Accumulation

## Overview
Homogeneous Medium Transmittance computes the fraction of radiance that survives transmission through a participating medium with constant optical properties along a ray segment [S2, S5, S7, unofficial]. It solves the problem of light attenuation in transparent or semi-transparent volumes by modeling the exponential decay of intensity due to absorption and out-scattering [S1, S6, unofficial]. It sets itself apart from [Heterogeneous Medium Transmittance — Beer's Law](4793-heterogeneous-medium-transmittance-beer-s-law.md) by assuming a spatially constant extinction coefficient, and from [Single Scattering Model](1053-single-scattering-model.md) by ignoring in-scattering contributions, focusing solely on the loss of radiance from the primary ray path [S1, S2, unofficial].

## How it works
**Inputs:**
- $s_{start}$: Entry point of the ray segment into the medium.
- $s_{end}$: Exit point of the ray segment.
- $\sigma_t$: Extinction coefficient of the homogeneous medium.

**Outputs:**
- $T$: Transmittance, the fraction of light transmitted from $s_{start}$ to $s_{end}$.

**Data Structures:**
No complex data structures are required for the core calculation. The method operates on scalar values. In ray-marching implementations, it is applied iteratively per step [S6, unofficial].

**Definitions:**
- $\sigma_t$: Total extinction coefficient, defined as the sum of the absorption coefficient $\sigma_a$ and the scattering coefficient $\sigma_s$ ($\sigma_t = \sigma_a + \sigma_s$) [S2, S7, unofficial].
- $d$: Geometric distance between two points in the medium, $d = \|s_{end} - s_{start}\|$ [S5, S7, unofficial].
- $\tau$: Optical depth, representing the integrated extinction along the path. For a homogeneous medium, $\tau = \sigma_t d$ [S2, S7, unofficial].

**Formulas:**
The transmittance $T$ is calculated using the Beer-Lambert law [S2, S5, S7, unofficial]:
$$ T = e^{-\sigma_t d} $$
Alternatively, expressed via optical depth $\tau$:
$$ T = e^{-\tau} $$
where $\tau = \sigma_t d$ [S2, S7, unofficial].

The radiance $L(s)$ at distance $s$ from the entry point, given initial radiance $L(0)$, is:
$$ L(s) = L(0) e^{-\sigma_t s} $$
This is the solution to the first-order homogeneous linear ordinary differential equation:
$$ \frac{dL(s)}{ds} = -\sigma_t L(s) $$
[S2, unofficial].

**Parameters:**
- $\sigma_t$: Extinction coefficient. Typical values depend on the medium density. In demos, an absorption coefficient $\alpha$ (often used interchangeably with $\sigma_a$ or $\sigma_t$ in simplified contexts) is set to 0.9 [S6, unofficial].
- $d$: Step size or ray segment length. In ray marching, this is `MARCH_SIZE` [S6, unofficial].

**Algorithm (Ray Marching Accumulation):**
When integrating transmittance through a volume via ray marching, the total transmittance is accumulated multiplicatively [S6, unofficial].

1. Initialize `totalTransmittance` to 1.0 [S6, unofficial].
2. Initialize `lightEnergy` (accumulated emitted/scattered light) to 0.0 [S6, unofficial].
3. For each step $i$ from 0 to $N-1$:
    a. Evaluate density $\rho$ at current ray position $p_i$.
    b. Calculate geometric step distance $\Delta d$ (e.g., `MARCH_SIZE`) [S6, unofficial].
    c. If $\rho > 0$:
        i. Calculate local extinction $\sigma_t = \rho \times \text{absorption\_coefficient}$ [S6, unofficial].
        ii. Calculate step transmittance $T_{step} = \exp(-\sigma_t \Delta d)$ [S6, unofficial].
        iii. Update `totalTransmittance` = `totalTransmittance` $\times T_{step}$ [S6, unofficial].
        iv. Calculate luminance contribution $L_{step} = \rho$ (or sampled emission) [S6, unofficial].
        v. Accumulate light: `lightEnergy` = `lightEnergy` + `totalTransmittance` $\times L_{step}$ [S6, unofficial].
    d. Advance ray position $p_{i+1} = p_i + \Delta d \cdot \text{direction}$ [S6, unofficial].
4. Return `lightEnergy` (and optionally `totalTransmittance` for background blending) [S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def homogeneous_transmittance(extinction_coefficient: float, distance: float) -> float:
    """
    Computes the transmittance of light through a homogeneous medium.
    
    Args:
        extinction_coefficient: The total extinction coefficient sigma_t (sigma_a + sigma_s).
        distance: The geometric distance d traveled through the medium.
        
    Returns:
        The fraction of light transmitted T.
    """
    if distance < 0.0:
        return 1.0 # Symmetric transmittance for negative distance is physically 1.0 in magnitude, 
                   # but typically distance is positive.
    if extinction_coefficient <= 0.0:
        return 1.0 # No extinction means full transmission.
        
    optical_depth = extinction_coefficient * distance
    return math.exp(-optical_depth)

def raymarch_homogeneous_volume(ray_origin, ray_direction, step_size, max_steps, 
                                extinction_coefficient_func, emission_func):
    """
    Ray marches through a volume using homogeneous Beer's Law for transmittance.
    Note: extinction_coefficient_func returns sigma_t for the current position.
    If the medium is truly homogeneous, this function returns a constant.
    """
    total_transmittance = 1.0
    accumulated_light = 0.0
    
    current_pos = ray_origin + ray_direction * step_size
    
    for i in range(max_steps):
        # Evaluate medium properties at current position
        sigma_t = extinction_coefficient_func(current_pos)
        emission = emission_func(current_pos)
        
        if sigma_t > 0.0:
            # Calculate transmittance for this small step
            # T_step = exp(-sigma_t * step_size)
            step_transmittance = math.exp(-sigma_t * step_size)
            
            # Update accumulated transmittance
            total_transmittance *= step_transmittance
            
            # Accumulate emitted/scattered light
            # The light emitted at this step is attenuated by the transmittance 
            # of all subsequent steps (which is already in total_transmittance 
            # if we were doing front-to-back, but here total_transmittance 
            # represents transmittance FROM origin TO current_pos.
            # Standard front-to-back accumulation:
            # color += emission * total_transmittance * (1 - step_transmittance) / sigma_t ?
            # Simplified from [S6, unofficial]: lightEnergy += totalTransmittance * luminance
            # Here, totalTransmittance is the transmittance FROM origin TO current_pos.
            # The light emitted at current_pos travels through the REST of the volume.
            # However, [S6, unofficial] accumulates lightEnergy += totalTransmittance * luminance.
            # This implies totalTransmittance is the transmittance FROM current_pos TO END?
            # No, [S6, unofficial] updates totalTransmittance *= transmittance BEFORE adding light.
            # So totalTransmittance is transmittance FROM origin TO current_pos.
            # This is physically incorrect for front-to-back accumulation unless 
            # 'luminance' is defined differently or it's a specific simplified model.
            # Standard volume rendering equation:
            # C = C_bg * T_total + \int L_e * T(s) * sigma_t ds
            # Discretized: C += L_e * T_current * sigma_t * delta_s
            # Where T_current is transmittance from origin to s.
            
            # Following [S6, unofficial] strictly:
            luminance = emission # Assuming emission represents the light added
            accumulated_light += total_transmittance * luminance
            
        current_pos += ray_direction * step_size
        
    return accumulated_light, total_transmittance
```

**Edge Cases and Pitfalls:**
- **Zero Extinction:** If $\sigma_t = 0$, transmittance is 1.0. Division by zero errors may occur if calculating opacity $\alpha = 1 - T$ and then dividing by $\sigma_t$ in scattering terms [S2, unofficial].
- **Numerical Underflow:** For large optical depths ($\sigma_t d \gg 1$), $T$ approaches 0. Floating-point underflow to 0 is acceptable but may cause gradient issues in differentiable rendering [S2, unofficial].
- **View Direction Alignment:** In slice-based rendering, opacity $\alpha = 1 - \exp(-\sigma_t \Delta x)$ must be adjusted if the viewing direction is not orthogonal to the slice planes, as the effective path length $\Delta x$ changes [S4, unofficial].
- **Lighting Artifacts:** Using only Beer's Law without a proper lighting model (e.g., sampling light direction) results in a "white blob" in the center of clouds because the volume does not interact with light directionally [S6, unofficial].

## Subtopics

### [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md)
Exponential Fog (Beer-Lambert) is a depth-cueing technique that simulates atmospheric scattering by fading scene colors toward a background (sky) color based on the viewer's distance from the camera. It solves the problem of visual discontinuity at the far clipping plane and hides tiling artifacts by creating a physically plausible exponential decay of visibility. It sets itself apart from [Linear Distance Fog](4409-linear-distance-fog.md) by using an exponential decay function rather than a linear blend, and from [Heterogeneous Medium Transmittance — Beer's Law](4793-heterogeneous-medium-transmittance-beer-s-law.md) by assuming a constant density homogeneous medium.

### [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md)
The Non-Exponential Free Flight Model describes light transport in participating media where scattering particles are positively correlated, resulting in free path distributions with longer tails than the standard exponential distribution. It allows for artistic control over the depth of light bleeding into subsurface media by introducing a shape parameter that modifies the probability of interaction based on distance traveled. Unlike the Exponential Fog model, which assumes uncorrelated particle distribution, this model gives photons "memory," where past non-interactions influence future interaction probabilities.

### [Heterogeneous Medium Transmittance — Beer's Law](4793-heterogeneous-medium-transmittance-beer-s-law.md)
Heterogeneous Medium Transmittance — Beer's Law computes the fraction of radiance transmitted between two points through a participating medium where the extinction coefficient varies spatially. It generalizes the homogeneous case by integrating the extinction coefficient along the ray path to determine the optical depth. Unlike [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md), which assumes a constant extinction coefficient, this method handles spatially varying media by evaluating the integral of $\sigma_t$ along the ray segment. It serves as the foundational transmittance model for [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) and [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md).

### Sun Transmittance Marching
Sun Transmittance Marching is a raymarching technique that calculates the transmittance of sunlight to a specific sample point by integrating light density along a ray directed toward the sun, using a simplified Beer-Lambert equation [S9, unofficial]. It initializes light density to 0, loops from 0 to `stepLimit` (exclusive), updates ray position by adding `_SunDirection * stepSize`, and checks if the normalized UVW position is within the unit cube [S9, unofficial]. If inside the box, it accumulates light density by adding the red channel of a 3D noise texture sampled at `rayPosition * _CloudScale` multiplied by `stepSize` [S9, unofficial]. The final transmittance is calculated as $\exp(-\text{lightDensity} \times \text{_AbsorptionCoefficient})$ [S9, unofficial]. Typical parameters are `stepSize` = 20.0 and `stepLimit` = 200 [S9, unofficial].

### Absorption Adjustment
In slice-based rendering, the absorption parameter $\alpha$ must be adjusted for view directions not aligned with the axis orthogonal to the viewing planes [S4, unofficial]. The absorption coefficient also needs adjustment when the distance between adjacent smoke planes changes or viewing planes are skipped [S4, unofficial]. For a simulation with grid dimensions of $100 \times 100 \times 100$, displaying smoke with corrected $\alpha$'s at 10 frames per second requires ten million exponential operations per second [S4, unofficial].

### Beer-Lambert Extinction with Powder Sugar Effect
Beer-Lambert Extinction with Powder Sugar Effect is an empirical correction for standard Beer-Lambert attenuation to prevent unrealistic darkening of cloud centers by accounting for intense forward multiple-scattering near boundaries [S10, unofficial]. It modifies the transmittance using a powder factor. The powder sugar multiplier is $T_{\text{powder}}(\tau) = 1.0 - \exp(-2 \tau)$ [S10, unofficial]. The effective transmittance is $T_{\text{effective}}(\tau) = \exp(-\tau) \cdot \max(T_{\text{powder}}(\tau), 0.05)$ [S10, unofficial]. The `powderFactor` parameter defaults to 1.0 [S10, unofficial].

## Operating characteristics
**Time Complexity:** $O(1)$ for a single ray segment calculation. In ray marching, $O(N)$ where $N$ is the number of steps [S6, unofficial]. Slice-based rendering requires $O(N^3)$ precomputation for opacity if done naively, but optimizations like run-length encoding reduce this [S4, unofficial]. Real-time performance requires significant computational power; for example, 10 million exponential operations per second are needed for 10 FPS on a $100^3$ grid [S4, unofficial].

**Memory Complexity:** $O(1)$ for the core transmittance calculation. Ray marching requires storage for the ray state and volume samples [S6, unofficial].

**Strengths:** Physically based model for light absorption; simple to implement; computationally efficient for homogeneous media [S6, S7, unofficial].

**Limitations:** Does not account for in-scattering (light entering the ray from other directions); assumes constant density; can produce unrealistic dark centers if not combined with scattering models [S6, unofficial].

**When to use:** Ideal for simple fog, smoke, or homogeneous participating media where in-scattering is negligible or approximated [S5, S6, unofficial].

**When not to use:** For heterogeneous media with significant spatial variation in density (use [Heterogeneous Medium Transmittance — Beer's Law](4793-heterogeneous-medium-transmittance-beer-s-law.md)) or when accurate scattering effects are required (use [Single Scattering Model](1053-single-scattering-model.md)) [S1, S2, unofficial].

## Background
The Beer-Lambert law describes the radiance $L(s)$ of a light beam at distance $s$ through a homogeneous medium, accounting for absorption and out-scattering (extinction) [S2, unofficial]. It is the solution to the first-order homogeneous linear ordinary differential equation for radiance change [S2, unofficial]. The law is fundamental in physics and chemistry for describing light attenuation [S4, S7, unofficial]. In computer graphics, it is widely used for volume rendering, fog effects, and cloud rendering [S1, S6, S7, unofficial]. The concept of optical depth $\tau$ generalizes this to heterogeneous media [S2, S7, unofficial].

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S3] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S4] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S5] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S6] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S7] Volumetric Lighting (God Rays) — How It Works - Moonjump, https://moonjump.com/game-dev-mechanics-volumetric-lighting-god-rays-how-it-works/, fetched 2026-10-28 (unofficial)
[S8] 体渲染简明教程 - BimAnt, https://bimant.com/blog/volume-rendering-brief-intro/, fetched 2026-10-30 (unofficial)
[S9] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S10] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
