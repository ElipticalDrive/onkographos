# Particle Tracing for Global Illumination
Also called: Photon Beam Tracing phase, Photon Mapping (Photon Beam Tracing phase), Volumetric Global Illumination (Stochastic/Monte-Carlo), Volumetric radiosity

## Overview
Particle tracing for global illumination is a technique that tracks paths of scattered light energy through space starting at light sources to solve the illumination transport equation [S2, unofficial][S3, book]. It models reflected illumination as an integral of incident radiation, incorporating global light visibility such as shadows and scattering, thereby ignoring local illumination assumptions [S3, book]. It is related to instant radiosity and photon mapping, which also splat energy around a scene [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, light source positions and intensities, participating media properties (scattering and absorption coefficients), and phase functions.
*   **Outputs:** Irradiance or radiance distribution within the volume or on surfaces.

**Data Structures:**
*   **Voxels:** Discrete elements in the volume used for radiosity calculations [S2, unofficial].
*   **Particles/Photons:** Carriers of light energy traversing the scene [S2, unofficial].

**Steps:**
1.  **Emission:** Generate particles from light sources [S2, unofficial].
2.  **Transport:** Track paths of scattered light energy through space [S2, unofficial]. This involves solving the illumination transport equation where the reflected illumination $I(\gamma, \omega)$ in direction $\omega$ at any voxel $\gamma$ is described as the integral of all incident radiation from directions $\omega'$ [S3, book].
    $$I(\gamma, \omega) = \int_{VT} q(\omega, \omega^\dagger) I(\gamma, \omega^\dagger) d\omega^\dagger dv$$
    where $\Gamma$ is the set of all directions and $V$ is the set of all voxels $v$ [S3, book].
3.  **Interaction:** Modulate the incident radiation by the phase function $q(\omega, \omega^\dagger)$ [S3, book].
4.  **Storage/Splatting:** Store the energy in voxels or splat it around the scene [S1, S2, unofficial]. In volumetric radiosity, voxels are regarded as discrete elements, computing on a voxel-by-voxel basis [S2, unofficial].
5.  **Accumulation:** The illumination at any voxel is dependent upon the illumination at every other voxel. In practice, this integral equation is solved by finite repeated projection of energy among voxels [S3, book].

**Parameters:**
*   Number of particles/photons.
*   Voxel resolution.
*   Phase function parameters.

**Edge Cases and Pitfalls:**
*   **Bias:** Simply terminating rays (e.g., early ray termination) results in biased images and an overall decrease in illumination [S1, unofficial]. Russian roulette termination helps remove this bias [S1, unofficial].
*   **Complexity:** The illumination at any voxel depends on every other voxel, leading to complex energy transport paths [S3, book].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Incorporates global light visibility (shadows, scattering) [S3, book].
*   **Limitations:** Not given in the sources.

## Background
*   **Origin:** The method is related to instant radiosity [S1, unofficial] and photon mapping [S1, unofficial]. It is also associated with volumetric radiosity calculations [S2, unofficial].
*   **History:** Research has been done on acceleration of ray tracing for decades [S1, unofficial]. Kajiya and Von Herzen calculated light intensity of cloud-like volumes one horizontal slice at a time [S2, unofficial]. The Heidelberg ray-tracing model demonstrated shadow rays propagated simultaneously slice-by-slice [S2, unofficial]. Dachille et al. described a backprojection approach [S2, unofficial].
*   **Usage:** Used for simulating participating media such as clouds, skin, and stone [S3, book].

## Subtopics

### Photon-based Estimation
Estimates illumination using photon densities. [Link](2521-photon-based-estimation.md)

### Photon Beam Tracing
Uses beams instead of particles for efficiency. [Link](5401-photon-beam-tracing.md)

### Grid-based Importance Sampling
Uses grids to sample boundary segments. [Link](7643-grid-based-importance-sampling-for-boundary-segments.md)

### Photon Beam Mapping
Maps photon beams to a grid. [Link](7874-photon-beam-mapping.md)

### Adaptive Beam Tracing
Adapts beam tracing for volume caustics. [Link](8406-adaptive-beam-tracing-for-volume-caustics.md)

### Instant Radiosity / Virtual Point Lights (VPLs)
Approximates scene illumination using a small number of virtual point lights. [Link](9319-instant-radiosity-virtual-point-lights-vpls.md)

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
