# Smoke Column with Helical Displacement


## Overview
Creates a single column of rising smoke by deforming a vertical cylinder, simulating natural dispersion, turbulence, and swirling motion [S1, unofficial]. It differs from its parent [Vortex Flow Function](2538-vortex-flow-function.md) by explicitly combining a height-dependent density falloff and turbulence increase with a vertical helical displacement field to bend and swirl the column, rather than relying solely on rotational flow around a static axis [S1, unofficial].

## How it works
**Inputs:**
- Point position $P = (x, y, z)$ in world or local space [G].
- Cylinder axis direction (typically $Z$ or $Y$) [G].
- Cylinder base radius $R_0$ [G].
- Column height $H$ [G].
- Time parameter $t$ (for animation) [G].

**Outputs:**
- Density value $\rho$ at point $P$ [G].
- Displaced position $P'$ (optional, for visualization or further deformation) [G].

**Data Structures:**
- No complex data structures required; the method is procedural and stateless [G].

**Steps:**
1. **Calculate Height:** Determine the normalized height $h$ of the point relative to the cylinder base, where $h \in [0, 1]$ [G]. If the point is outside the vertical bounds of the column, return zero density [G].
2. **Helical Displacement:** Apply a vertical helix to displace the point's horizontal coordinates $(x, y)$ before calculating the distance from the center [S1, unofficial]. The helix angle $\theta$ increases with height $h$ and time $t$ [G].
   - Let $r = \sqrt{x^2 + y^2}$ be the radial distance from the axis [G].
   - Let $\phi$ be the azimuthal angle of the point [G].
   - The helical displacement rotates the point by an angle $\Delta \theta = k \cdot h + \omega \cdot t$, where $k$ is the helix tightness and $\omega$ is the rotation speed [G].
   - The displaced horizontal position $(x', y')$ is obtained by rotating $(x, y)$ by $\Delta \theta$ [G].
3. **Distance from Center:** Calculate the distance $d$ from the displaced point $(x', y', z)$ to the cylinder axis [S1, unofficial].
4. **Turbulence:** Add turbulence to the displacement or density based on height. Turbulence increases as a function of height to simulate air currents [S1, unofficial]. This can be implemented by adding a noise function $N(x', y', z, t)$ scaled by a height-dependent factor $T(h)$ [G].
5. **Density Calculation:** Compute the density $\rho$ based on the distance $d$ and height $h$. The density decreases as a function of height to simulate dispersion [S1, unofficial]. A common profile is $\rho = \rho_0 \cdot (1 - h) \cdot f(d)$, where $f(d)$ is a radial falloff function (e.g., Gaussian or exponential) [G].
6. **Edge Cases:** Points outside the cylinder radius $R_0$ or outside the height range $[0, H]$ have zero density [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smoke_column_density(P, R0, H, k, omega, t, noise_func):
    x, y, z = P
    if z < 0 or z > H:
        return 0.0
    h = z / H
    r = math.sqrt(x*x + y*y)
    phi = math.atan2(y, x)
    delta_theta = k * h + omega * t
    new_phi = phi + delta_theta
    x_prime = r * math.cos(new_phi)
    y_prime = r * math.sin(new_phi)
    # Rotation preserves radius, so d == r
    d = math.sqrt(x_prime*x_prime + y_prime*y_prime)
    turbulence_factor = h
    noise_val = noise_func(x_prime, y_prime, z, t)
    d_effective = d + turbulence_factor * noise_val
    height_falloff = 1.0 - h
    radial_falloff = math.exp(-(d_effective * d_effective) / (2 * R0 * R0))
    return height_falloff * radial_falloff
```

**Parameters:**
- $R_0$: Base radius. Typical value: Not given in the sources.
- $H$: Column height. Typical value: Not given in the sources.
- $k$: Helix tightness. Typical value: Not given in the sources.
- $\omega$: Rotation speed. Typical value: Not given in the sources.
- Noise function: A procedural noise function (e.g., Perlin noise). Typical value: Not given in the sources.

**Pitfalls:**
- Precision issues at the cylinder axis ($r=0$) when calculating angles. Use `atan2` for robustness [G].
- Boundary conditions: Ensure the density smoothly transitions to zero at the top and bottom of the column [G].

## Subtopics

### [Procedural Flame Density & Color Shader](3033-procedural-flame-density-color-shader.md)
Computes the density and color of a procedural flame volume by deforming a base helical flow path with height-scaled turbulence and applying spatial attenuation ramps. It builds on [Smoke Column with Helical Displacement](2743-smoke-column-with-helical-displacement.md) by adding complex shaping via sine waves and a lookup table, while explicitly handling vertical fading and minimum density thresholds to create a realistic fire appearance. It differs from its parent by introducing a multi-stage attenuation process (horizontal and vertical) and a direct density-to-color mapping that shifts from yellow to red based on density values.

### Synapse Volume Construction
Construction of volumetric emission objects (synapses) connecting an inner sphere to an outer sphere, simulating light-emitting plasma [S2, unofficial]. It defines the base geometry as a straight line from the inner sphere to the outer sphere representing a plasma strand (synapse) taking the shortest distance path [S2, unofficial]. It adds seven layered randomization factors to each synapse: A. Circular randomization function offsetting end points. B. Curve amplitude offset using sinusoidal curves to simulate randomization of the synapse path. C. Width offset to simulate varying intensity and thickness of the synapse along the length. D. Gaussian distribution and color gradients between Electric Blue and Neon Pink along the length of the synapse. E. Endpoint artefacts created by exponential expansion of the synapse when closer to the ends to show surface diffusion effect. F. Surface illumination along the outer sphere to simulate sub-surface emission from the plasma globe [S2, unofficial]. Parameters include 82 instances of the synapse object and a color gradient between Electric Blue and Neon Pink [S2, unofficial].

### Gradient Driven Displacement
A displacement technique requiring a monochrome texture map. The displacement amount is the texture's brightness at each point. Each point of the fluid is shifted along the gradient of the Surface channel, meaning each point in space could have a different displacement direction. Suitable for smoke and fire [S3]. The parameter `displ2` specifies the displacement technique, with 'Gradient driven' being one option [S3].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it is a procedural function [G].
- **Memory Complexity:** $O(1)$, no data structures required [G].
- **Performance:** Fast to evaluate, but the addition of noise functions and helical calculations can increase computational cost compared to simple radial falloffs [G].
- **Strengths:** Simple to implement, produces realistic swirling smoke columns, easily animatable [G].
- **Limitations:** Lacks complex internal structures (e.g., eddies) unless enhanced with noise. The helical displacement may not accurately simulate all types of turbulent flow [G].
- **When to use:** When a simple, animated smoke column with swirling motion is needed [G].
- **When not to use:** When complex, non-helical turbulence or detailed internal structures are required [G].

## Background
Described in procedural modeling of gases and fluids [S1, unofficial]. The method is similar to steam rising procedures but adapted for smoke with added helical displacement for bending and swirling [S1, unofficial]. It is used in computer graphics for simulating smoke columns in animations and visual effects [G].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
[S2] Home, https://graphics.cg.uni-saarland.de/competition/cg1-2018/original/groupR_web/, fetched 2026-10-28 (unofficial)
[S3] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
