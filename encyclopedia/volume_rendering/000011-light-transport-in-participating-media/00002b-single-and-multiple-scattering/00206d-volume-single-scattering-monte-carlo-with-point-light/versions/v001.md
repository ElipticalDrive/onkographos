# Volume Single Scattering (Monte Carlo with Point Light)

Also called: GetLightEnergy, Light Energy Calculation, PBRT SingleScatteringIntegrator::Li, Single Scattering Volume Integration (PBRT Implementation)

## Overview
Computes the radiance contribution of a single scattering event in a homogeneous participating medium by integrating in-scattered light from a point light source along a view ray. It differs from uniform ray marching variants by using Monte Carlo distance sampling to select scattering positions, which reduces bias from fixed step discretization [S1, unofficial]. It is a specific instance of single scattering approximations [5340-single-scattering-approximations.md] tailored for point lights and homogeneous media.

## How it works
**Inputs:**
- Ray $r$ with origin $o$ and direction $\omega$.
- Homogeneous volume region with total extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, and phase function $p(\omega_i, \omega_o)$.
- Point light source at position $L_{pos}$ with intensity $I$.
- Background radiance $L_{bg}$ (computed via point light model or environment).
- Number of Monte Carlo samples $N_{samples}$.

**Outputs:**
- Accumulated radiance $L$ at the pixel.

**Data Structures:**
- `Volume`: Provides methods `getSigmaS()`, `getPhaseFunctionValue()`, `getTransmittanceHomogeneous(t1, t2)`.
- `Light`: Provides `getLightPosition()`.
- `Sampler`: Provides `sample1DDBG` for distance sampling.

**Steps:**
1. **Background Transmittance:** Calculate the intersection distance $t_{max}$ to the background surface. Compute the transmittance from the ray origin to $t_{max}$:
   $$ T_{bg} = e^{-\sigma_t t_{max}} $$
   Compute the background radiance $L_{bg}$ attenuated by this transmittance:
   $$ L = L_{bg} \cdot T_{bg} $$
   [S1, unofficial]

2. **Initialize Accumulator:** Initialize the scattered radiance accumulator $L_s = 0$.
   [S1, unofficial]

3. **Monte Carlo Integration Loop:** For $i = 0$ to $N_{samples} - 1$:
   a. **Sample Distance:** Sample a distance $t_{sample}$ along the ray from the origin using uniform distance sampling weighted by extinction. The PDF for sampling distance $t$ in a homogeneous medium is:
      $$ p(t) = \sigma_t e^{-\sigma_t t} $$
      [S1, unofficial]
      *Note: The source code uses a helper `sample1DDBG` which returns the sampled distance and updates the PDF.*
   
   b. **Sample Position:** Calculate the 3D position of the scattering event:
      $$ x_{sample} = o + t_{sample} \cdot \omega $$
      [S1, unofficial]

   c. **Eye Transmittance:** Calculate the transmittance from the eye (ray origin) to the sample point:
      $$ T_{eye} = e^{-\sigma_t t_{sample}} $$
      [S1, unofficial]

   d. **Light Radiance:** Calculate the emitted radiance from the point light at $x_{sample}$:
      $$ L_e = \frac{I}{\|x_{sample} - L_{pos}\|^2} $$
      *Note: In the source, this is computed via `getPixelColorPointLight` which handles the inverse square law and attenuation.*
      [S1, unofficial]

   e. **Light Transmittance:** Calculate the transmittance from the sample point to the light source:
      $$ T_{light} = e^{-\sigma_t \|x_{sample} - L_{pos}\|} $$
      [S1, unofficial]

   f. **Phase Function:** Retrieve the phase function value for the scattering direction. For a homogeneous volume, this is often constant or depends on the angle between incoming light and view direction:
      $$ p = p(\omega_{light}, -\omega) $$
      [S1, unofficial]

   g. **Accumulate:** Add the contribution to $L_s$, divided by the PDF to correct for the sampling distribution:
      $$ L_s = L_s + \frac{L_e \cdot T_{eye} \cdot \sigma_s \cdot p \cdot T_{light}}{p(t_{sample})} $$
      [S1, unofficial]

4. **Average and Finalize:** Average the scattered radiance over the number of samples and add to the background term:
   $$ L = L + \frac{L_s}{N_{samples}} $$
   [S1, unofficial]

**Parameters:**
- $t_{max}$: Distance to background intersection [S1, unofficial].
- $\sigma_s$: Scattering coefficient [S1, unofficial].
- $N_{samples}$: Number of Monte Carlo samples [S1, unofficial].

**Edge Cases:**
- If $t_{max}$ is infinity (no background intersection), the background term is 0 or handled as environment map [S1, unofficial].
- If $\sigma_s = 0$, no scattering occurs; return only background [S1, unofficial].
- Precision issues may arise if $t_{sample}$ is very large, causing $T_{eye}$ to underflow to 0.

## Subtopics

### [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md)
A ray marching algorithm that accounts for four light-particle interactions: absorption, out-scattering (attenuation), emission (ignored), and in-scattering (energy gain). It distinguishes between homogeneous (uniform coefficients) and heterogeneous (spatially varying density) participating media. [2318-ray-marching-with-in-out-scattering-and-density-scaling.md]

### [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md)
A rendering approximation that considers only single scattering effects and uses pre-computed direction-independent radiance components to improve efficiency. [4188-single-scattering-approximation-with-precomputed-terms.md]

### [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md)
An efficient analytical approximation for computing in-scattered radiance $J_{ss}$ that avoids repeated ray tracing by decomposing the phase function and incident radiance into direction-independent terms pre-computed per voxel. [4244-analytical-approximation-of-single-scattering-radiance.md]

### [Volume Single Scattering (Monte Carlo with Sphere Light)](4549-volume-single-scattering-monte-carlo-with-sphere-light.md)
Variant of Monte Carlo single scattering using sphere light for background and emission, with equi-angular sampling mode. [4549-volume-single-scattering-monte-carlo-with-sphere-light.md]

### [Volume Single Scattering (Uniform Ray Marching with Point Light Background)](4550-volume-single-scattering-uniform-ray-marching-with-point-lig.md)
Variant of uniform ray marching where the background radiance is computed using a point light model instead of a generic surface object color. [4550-volume-single-scattering-uniform-ray-marching-with-point-lig.md]

### [BSSRDF Approximation with Dipole and Single Scattering](4750-bssrdf-approximation-with-dipole-and-single-scattering.md)
[4750-bssrdf-approximation-with-dipole-and-single-scattering.md]

### [Physical Atmosphere Color Calculation](4820-physical-atmosphere-color-calculation.md)
Calculates the physical color of the sky and sun disk for a given view direction, accounting for Rayleigh and Mie scattering, optical depth, and phase functions. [4820-physical-atmosphere-color-calculation.md]

### Single Scattering with Direction Variance Culling
Traverses a hierarchy of volume samples in a top-down manner, discarding nodes not intersected by the camera ray. At leaf nodes, it tests the direction variance and phase function value. If both are below a threshold, it gathers the contribution from the leaf node; otherwise, it gathers from each sample. The contribution is scaled by a density factor $k_v$:
$$ \text{single} = \alpha \sum_v p(\hat{\boldsymbol{h}}, \hat{\boldsymbol{d}}_v) \frac{I_v}{k_v} (e^{-\sigma_T d_{\text{near}}} - e^{-\sigma_T d_{\text{far}}}) $$
where $d_{\text{near}}$ and $d_{\text{far}}$ are the intersections with the sample volume of influence, $\alpha$ is an albedo/scattering multiplier, and $p$ is the phase function [S4].

## Operating characteristics
- **Time Complexity:** $O(N_{samples})$ per ray, where $N_{samples}$ is the number of Monte Carlo samples. Each sample requires a light visibility check (ray intersection) and transmittance calculation.
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, excluding the scene data structures.
- **Performance:** Convergence is slow due to the high variance of Monte Carlo integration in participating media, especially in dense volumes. Requires many samples for noise-free results.
- **Strengths:** Unbiased result for homogeneous media with point lights. Simple to implement.
- **Limitations:** Computationally expensive for high-quality images. Does not handle multiple scattering. Sensitive to the number of samples.
- **When to use:** Real-time applications with low sample counts or offline rendering where bias is unacceptable for simple homogeneous volumes.
- **When not to use:** Heterogeneous media (requires different sampling), multiple scattering, or when performance is critical and bias is acceptable.

## Background
The method is based on the Monte Carlo integration of the radiative transfer equation for single scattering. The specific implementation details, including the use of `getPixelColorPointLight` and `getTransmittanceHomogeneous`, are derived from the PBRT (Physically Based Rendering Toolkit) single scattering integrator and educational ray tracing tutorials [S1, S3, unofficial]. The approach of using distance sampling for homogeneous media is a standard technique in volume rendering to reduce discretization error compared to uniform ray marching [S1, S3, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S3] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S4] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
