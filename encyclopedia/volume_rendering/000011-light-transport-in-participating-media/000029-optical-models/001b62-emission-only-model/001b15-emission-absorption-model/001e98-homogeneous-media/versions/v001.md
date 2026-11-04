# Homogeneous Media

Also called: Homogeneous Volume, Homogeneous volume representation, Uniform Participating Medium

## Overview
Homogeneous Media is a specific optical model for participating media where volumetric properties (density, absorption, scattering) are constant throughout a defined geometric boundary [S1, S2, unofficial]. It simplifies the general Emission-Absorption Model by removing spatial variation, allowing for closed-form transmittance calculations [S1, unofficial]. Unlike the Particle Model or Point-Sampled Spectrum Model, which handle discrete or sparse data, this method assumes a continuous, uniform medium [S2, unofficial].

## How it works
**Inputs:**
- Ray origin $x_o$ and direction $\vec{d}$.
- Geometric boundary defining the medium's extent [S2, unofficial].
- Constant extinction coefficient $\sigma_t$ [S1, unofficial].
- Constant scattering coefficient $\sigma_s$ [S1, unofficial].
- Constant absorption coefficient $\sigma_a$ [S1, unofficial].

**Outputs:**
- Transmittance $T_r$ from a point to another along the ray [S1, unofficial].
- Hit point and accumulated radiance [S2, unofficial].

**Data Structures:**
The medium is represented by a class `HomogeneousVolume` which inherits from `Object` [S2, unofficial].
- Fields:
  - `Object *boundary_`: Pointer to the enclosing geometric boundary [S2, unofficial].
  - `double density_`: Constant density value used in intersection and scattering calculations [S2, unofficial].
  - `Material *material`: Associated material properties [S2, unofficial].
- Constructor: `HomogeneousVolume(Object *boundary, double density, Material *material)` [S2, unofficial].

**Steps:**
1. **Intersection:** Intersect the ray with `boundary_` to find entry and exit points $x_{in}$ and $x_{out}$ [S2, unofficial]. If no intersection, the ray does not pass through the medium.
2. **Distance Calculation:** Compute the distance $t$ along the ray within the medium: $t = \| x_{out} - x_{in} \|$ [S1, unofficial].
3. **Extinction Calculation:** Calculate the total extinction coefficient $\sigma_t$ [S1, unofficial]:
   $$ \sigma_t = \sigma_s + \sigma_a $$
   where $\sigma_s$ is the scattering factor and $\sigma_a$ is the absorption factor [S1, unofficial].
4. **Transmittance:** Compute the transmittance $T_r(x, x_t)$ between two points separated by distance $t$ [S1, unofficial]:
   $$ T_r(x, x_t) = \exp(- \sigma_t t) $$
   where $T_r$ is transmittance, $\sigma_t$ is the extinction factor, and $t$ is the distance between points [S1, unofficial].
5. **Scattering/Absorption:** Use `density_` for further scattering or absorption calculations as required by the specific rendering algorithm [S2, unofficial].

**Parameters:**
- $\sigma_s$: Scattering factor [S1, unofficial].
- $\sigma_a$: Absorption factor [S1, unofficial].
- $\sigma_t$: Extinction factor, defined as $\sigma_s + \sigma_a$ [S1, unofficial].
- `density_`: Constant density value [S2, unofficial].

**Edge Cases:**
- If the ray does not intersect the boundary, the medium is ignored [S2, unofficial].
- Precision issues may arise if $t$ is very small or $\sigma_t$ is very large, leading to underflow in transmittance calculation [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for transmittance calculation given intersection points [S1, unofficial]. Intersection with boundary depends on the geometry type [S2, unofficial].
- **Memory Complexity:** $O(1)$ per volume instance, storing only boundary pointer, density, and material [S2, unofficial].
- **Strengths:** Simple implementation, fast evaluation due to constant properties [S1, S2, unofficial].
- **Limitations:** Cannot represent spatial variations in density or material properties [S1, unofficial].
- **When to use:** For simple, uniform fog, smoke, or clouds where spatial variation is negligible [S1, unofficial].
- **When not to use:** For complex volumetric effects requiring density gradients or anisotropic scattering [S1, unofficial].

## Background
The homogeneous media model is a fundamental concept in volume rendering, often introduced in educational contexts such as CMU's 15-468 course [S1, unofficial]. It serves as a baseline for understanding light transport in participating media before introducing more complex spatial variations [S1, unofficial].

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
