# Scan-line Volume Rendering

Also called: Geometrically-Based Light Shaft (Li), Max's Scan-line Method

## Overview
Scan-line Volume Rendering is a method for rendering light shafts (volumetric lighting) in participating media by exploiting the single-scattering assumption [S2, unofficial]. It computes the radiance reaching the viewer by integrating scattered light contributions only within "illumination volumes" (regions of the medium directly lit by the source) [S1][S2, unofficial]. Unlike general ray marching which samples the entire view ray, this method uses a scan-line or ray-casting approach combined with shadow volumes to determine lit segments, significantly reducing computation by ignoring unlit regions [S1][S2, unofficial]. It is distinct from Nishita’s ray-casting variant by its specific scan-line execution context and from general path tracing by its restriction to single scattering and geometric volume integration [S1].

## How it works
**Inputs:**
- Camera parameters (view direction, scan-line position).
- Light source definition (position, type: point or directional/infinity).
- Participating medium properties (density $\sigma_t$, assumed constant; phase function $p(\theta)$, assumed isotropic) [S1][S2, unofficial].
- Geometry defining the scene boundaries and occluders (polygons) to construct shadow and illumination volumes [S1][S2, unofficial].

**Outputs:**
- Pixel radiance value for the current scan-line or ray.

**Data Structures:**
- **Shadow Volume:** A geometric representation of the region occluded from the light source by scene geometry.
- **Illumination Volume:** The intersection of the participating medium and the region directly illuminated by the light source. This is the complement of the shadow volume within the medium bounds [S1][S2, unofficial].
- **Scan-line/Ray:** A 1D or 3D path through the medium.

**Steps:**
1. **Construct Volumes:** Determine the shadow volume cast by scene geometry from the light source. Define the illumination volume as the lit portion of the participating medium [S1][S2, unofficial].
2. **Intersect Ray:** For each pixel (or scan-line), cast a ray from the camera into the scene [S1][S2, unofficial].
3. **Identify Lit Segments:** Calculate the intersection of the view ray with the illumination volume. This yields one or more line segments along the ray where scattering occurs [S1][S2, unofficial].
4. **Integrate Scattering:** For each lit segment, compute the single-scattering contribution. The radiance $L$ is the sum of contributions from these segments [S1][S2, unofficial].
   - The integral is solved analytically or via hardware-accelerated surface integration rather than discrete sampling [S2, unofficial].
   - For a **point light** with constant density $\sigma_t$ and isotropic scattering, the contribution along a segment from $t_1$ to $t_2$ involves the phase function $p(\theta)$ and the exponential attenuation $e^{-\sigma_t (t - t_{light})}$ [S1].
   - For a **light at infinity** (directional), the method supports constant or layered density and any phase function [S1].
5. **Composite:** Sum the contributions from all lit segments to form the final pixel color [S1][S2, unofficial].

**Formulas:**
The radiance $L$ reaching the eye is given by:
$$ L = \sum_{\text{lit segments}} \int_{t_{start}}^{t_{end}} \sigma_s \, p(\theta) \, L_{source} \, e^{-\sigma_t (t - t_{light})} \, e^{-\sigma_t (t_{eye} - t)} \, dt $$
Where:
- $\sigma_s$: Scattering coefficient (part of $\sigma_t$).
- $p(\theta)$: Phase function (isotropic $p(\theta) = 1/4\pi$) [S1][S2, unofficial].
- $L_{source}$: Light intensity.
- $\sigma_t$: Total extinction coefficient (constant) [S1][S2, unofficial].
- $t$: Distance along the view ray.
- $t_{light}$: Distance to the light source along the light ray.
- $t_{eye}$: Distance to the eye.

**Parameters:**
- **Scattering Assumption:** Isotropic [S2, unofficial].
- **Density:** Constant [S1][S2, unofficial].
- **Light Types:** Point light or Light at infinity (directional) [S1].

**Edge Cases and Pitfalls:**
- **Multiple Scattering:** The method does not account for multiple scattering, leading to inaccuracies in dense media [S2, unofficial].
- **Complex Geometry:** Constructing accurate shadow and illumination volumes for complex polygonal scenes can be computationally expensive [S1].
- **Precision:** Analytical integration requires careful handling of exponential terms to avoid precision loss in deep scenes [G].

## Subtopics

### Nishita’s Ray-Casting Variant
Uses similar shadow/illumination volume concepts but relies on ray casting rather than scan-line processing, and extends to spotlights with angular distributions [S1].

### Consolidated Volumes (Li)
Introduces a consolidated volume representation for shafts of light and shadow to simulate atmospheric scattering effects, using hardware-accelerated integral calculation on volume surfaces [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the complexity of the shadow volume construction and the number of lit segments per ray. Generally faster than full ray marching for sparse lighting because it skips unlit regions [S1][S2, unofficial].
- **Memory Complexity:** Requires storage for shadow and illumination volumes, which can be significant for complex scenes [S1][S2, unofficial].
- **Strengths:** Efficient for rendering light shafts in scenes with clear light sources and occluders; exploits hardware for integral calculation [S2, unofficial].
- **Limitations:** Limited to single scattering; assumes constant density and isotropic scattering; not suitable for dynamic, highly scattering media [S1][S2, unofficial].
- **When to Use:** Real-time or interactive rendering of light shafts (god rays) in scenes with known light sources and static geometry [S1][S2, unofficial].
- **When Not to Use:** Scenes requiring multiple scattering, variable density, or complex phase functions beyond isotropic [S1][S2, unofficial].

## Background
The method was introduced by Max [54] as a scan-line based approach to deal with shadow volumes to account for glow in haze [S1]. It was later extended by Nishita et al. [62], who adopted the use of shadow and illumination volumes but switched to a ray-casting approach and extended it to spotlights [S1]. Li [45] further proposed consolidated volume representations and analytical integration approaches [S2, unofficial]. The method is part of the broader family of single-scattering volume rendering techniques [S1][S2, unofficial].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] A Review On Light Shafts Rendering For Indoor Scenes, https://doi.org/10.5281/zenodo.1125510, fetched 2026-10-26 (unofficial)
