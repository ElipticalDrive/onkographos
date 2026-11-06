# Homogeneous Media

Also called: Homogeneous Volume, Homogeneous volume representation, Uniform Participating Medium

## Overview
Homogeneous Media is an optical model for participating media where volumetric properties are uniform at every point within a defined geometric boundary [S1, S2, unofficial]. It simplifies the [Emission-Absorption Model](927-emission-absorption-model.md) by assuming constant density, enabling closed-form transmittance calculations [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $x_o$ and direction $\vec{d}$.
- Geometric boundary defining the medium's extent [S2, unofficial].
- Uniform scattering factor $\sigma_s$ [S1, unofficial].
- Uniform absorption factor $\sigma_a$ [S1, unofficial].

**Outputs:**
- Transmittance $T_r$ along a ray segment [S1, unofficial].

**Data Structures:**
The medium is represented by a class `HomogeneousVolume` inheriting from `Object` [S2, unofficial].
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
Not given in the sources.

## Background
The homogeneous media model is a fundamental concept in volume rendering, documented in educational materials such as CMU's 15-468 course slides [S1, unofficial].

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
