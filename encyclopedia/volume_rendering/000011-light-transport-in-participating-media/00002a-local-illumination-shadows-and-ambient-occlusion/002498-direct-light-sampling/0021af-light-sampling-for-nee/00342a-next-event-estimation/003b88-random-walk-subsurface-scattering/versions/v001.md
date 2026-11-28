# Random-Walk Subsurface Scattering

Also called: Chiang 2016 SSS, Disney Subsurface Approximation, Subsurface Scattering (Random Walk)

## Overview
Random-walk subsurface scattering is a Monte Carlo technique for rendering translucent materials like skin, wax, or marble by simulating light transport within the mesh volume [S1][S2, unofficial]. It is distinguished from standard volume rendering by its efficiency and controllability for materials with well-defined surfaces, avoiding the high cost of volumetric path tracing for multiple scattering events [S1].

## How it works
**Inputs:** Surface intersection point, material subsurface coefficients (weight, radius, scale, anisotropy), and scene geometry [S2, unofficial].
**Outputs:** Radiance contribution from subsurface scattering.

**Data Structures:**
The method relies on the `SurfaceInteraction` structure, which carries the path wavelength and material properties [S2, unofficial]. Specific material hooks include `subsurfaceCoeffs` and `isVolumeBoundary` [S2, unofficial].

**Steps:**
1. **Entry:** Upon shading a surface point, if the material has a subsurface lobe (e.g., Disney Principled BSDF), the random walk is initiated inside the mesh geometry [S2, unofficial].
2. **Walk:** The photon or path segment performs a random walk through the interior volume. This is not a surface bounce but a distinct scattering event within the material [S2, unofficial].
3. **Termination:** The walk terminates when it reaches a "white Lambertian vertex" [S2, unofficial].
4. **Exit and NEE:** At the exit vertex, full Next Event Estimation (NEE) is applied to estimate direct illumination from light sources [S2, unofficial].

**Parameters:**
The Disney Principled BSDF defines the subsurface behavior using four key parameters: subsurface weight, radius, scale, and anisotropy [S2, unofficial].

**Pitfalls:**
Implementers must distinguish the random walk from a standard surface bounce; it represents internal scattering [S2, unofficial].

## Subtopics

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Performance:** The method is described as more efficient and easier to control than volume rendering for materials requiring multiple scattering [S1].
**Strengths:** Accurately renders materials like skin or milk [S1].
**Limitations:** Not suitable for materials without a well-defined surface, such as clouds or smoke, which require full volume rendering [S1].

## Background
The method is associated with Chiang 2016 and is implemented in renderers like Cycles as the "subsurface" approximation [S2, unofficial]. It is used in Blender's Principled BSDF material system [S2, unofficial].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
