# Volumetric Path Tracing with BSSRDF and Medium Sampling

Also called: BSSRDF integration, Path Tracing with Subsurface Scattering and Participating Media, Subsurface Scattering Integration in Volumometric Path Tracing, Subsurface scattering in VolPathIntegrator, VolPathIntegrator with BSSRDF

## Overview
This method extends unidirectional path tracing to handle light transport through participating media that also exhibit subsurface scattering (SSS) via BSSRDFs. It solves the problem of accurately rendering translucent materials (like skin, wax, or marble) embedded in or surrounded by volumetric fog by treating subsurface transport as a surface-to-surface scattering event within the path. Unlike pure volume integrators, it bridges surface geometry and participating media by sampling a BSSRDF probe segment to find an exit point on the same material, then continuing the path with BSDF sampling [S1][S3]. It differs from [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by explicitly handling the attenuation and geometry traversal of subsurface layers, and from [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md) by integrating direct lighting estimation at both surface and volume interaction points.

## How it works
**Inputs:**
- Ray `ray` (origin, direction, time, medium).
- Intersection `isect` (point, normal, material, BSSRDF if available).
- Path state: throughput `beta`, rescaled probabilities `r_u`, `r_l`, depth `depth`, max depth `maxDepth`.
- Sampler `sampler` (random number generator).
- Scene geometry and medium properties.

**Outputs:**
- Radiance contribution `L`.
- Updated path state for continuation or termination.

**Data Structures:**
- `BSSRDFProbeSegment`: Contains entry point `p0` and exit point `p1` of the subsurface probe.
- `WeightedReservoirSampler<T>`: Reservoir sampling structure to select a random intersection along the probe segment.
- `BSSRDFSample`: Contains subsurface scattering coefficient `Sp`, PDF `pdf`, outgoing direction `wo`, and associated BSDF `Sw`.
- `MediumProperties`: Contains absorption `sigma_a`, scattering `sigma_s`, extinction `sigma_e`, phase function, and emission `Le`.

**Steps:**

1. **Retrieve BSSRDF:**
   Get the BSSRDF from the intersection: `BSSRDF bssrdf = isect.GetBSSRDF(ray, lambda, camera, scratchBuffer)`. If no BSSRDF exists or it is not transmission, skip subsurface scattering.

2. **Sample BSSRDF Probe Segment:**
   Generate random numbers: `Float uc = sampler.Get1D()`, `Point2f up = sampler.Get2D()`.
   Sample the probe segment: `pstd::optional<BSSRDFProbeSegment> probeSeg = bssrdf.SampleSp(uc, up)`.
   If no probe segment is returned, break the path.

3. **Sample Random Intersection along Probe Segment:**
   Initialize reservoir sampler: `uint64_t seed = MixBits(FloatToBits(sampler.Get1D()))`, `WeightedReservoirSampler<SubsurfaceInteraction> interactionSampler(seed)`.
   Start from entry point: `Interaction base(probeSeg->p0, ray.time, Medium())`.
   Loop to intersect the scene along the segment:
   ```python
   while True:
       r = base.SpawnRayTo(probeSeg->p1)
       si = Intersect(r, 1) # Intersect ray against scene geometry
       if not si:
           break
       base = si->intr
       if si->intr.material == isect.material:
           interactionSampler.Add(SubsurfaceInteraction(si->intr), 1.0)
   ```
   If no sample was found (`!interactionSampler.HasSample()`), break the path.

4. **Convert Probe Intersection to BSSRDFSample:**
   Get the sampled subsurface interaction: `SubsurfaceInteraction ssi = interactionSampler.GetSample()`.
   Convert to sample: `BSSRDFSample bssrdfSample = bssrdf.ProbeIntersectionToSample(ssi, scratchBuffer)`.
   If no sample point or PDF (`!bssrdfSample.Sp || !bssrdfSample.pdf`), break the path.

5. **Update Path State for Subsurface Scattering:**
   Calculate PDF: `Float pdf = interactionSampler.SampleProbability() * bssrdfSample.pdf[0]`.
   Update throughput and rescaled probabilities:
   `beta *= bssrdfSample.Sp / pdf`
   `r_u *= bssrdfSample.pdf / bssrdfSample.pdf[0]`
   Set surface interaction: `SurfaceInteraction pi = ssi`, `pi.wo = bssrdfSample.wo`, `prevIntrContext = LightSampleContext(pi)`.

6. **Regularize Subsurface BSDF (Optional):**
   Get BSDF: `BSDF &Sw = bssrdfSample.Sw`.
   If `regularize` flag is set: `Sw.Regularize()`.
   Set `anyNonSpecularBounces = true`.

7. **Account for Attenuated Direct Illumination:**
   `L += SampleLd(pi, &Sw, lambda, sampler, beta, r_u)`.

8. **Sample Indirect Subsurface Scattering:**
   Generate random number: `Float u = sampler.Get1D()`.
   Sample BSDF: `pstd::optional<BSDFSample> bs = Sw.Sample_f(pi.wo, u, sampler.Get2D())`.
   If no sample, break the path.
   Update state:
   `beta *= bs->f * AbsDot(bs->wi, pi.shading.n) / bs->pdf`
   `r_l = r_u / bs->pdf`
   `specularBounce = bs->IsSpecular()`
   `ray = RayDifferential(pi.SpawnRay(bs->wi))`

9. **Russian Roulette Termination:**
   Calculate termination probability: `SampledSpectrum rrBeta = beta * etaScale / r_u.Average()`.
   Generate random number: `Float uRR = sampler.Get1D()`.
   If `rrBeta.MaxComponentValue() < 1 && depth > 1`:
   `Float q = std::max<Float>(0, 1 - rrBeta.MaxComponentValue())`
   If `uRR < q`, break the path.
   Else, `beta /= 1 - q`.

10. **Sample the Participating Medium:**
    Determine max distance: `Float tMax = si ? si->tHit : Infinity`.
    Initialize RNG: `uint64_t hash0 = Hash(sampler.Get1D())`, `uint64_t hash1 = Hash(sampler.Get1D())`, `RNG rng(hash0, hash1)`.
    Call `SampleT_maj` with a lambda function that handles medium events.

11. **Medium Event Handling (Lambda Function):**
    For each sampled point `p` with medium properties `mp` and majorant transmittance `T_maj`:
    
    a. **Emission Estimator:**
       If `depth < maxDepth && mp.Le`:
       Compute PDF: `Float pdf = sigma_maj[0] * T_maj[0]`.
       Compute beta prime: `SampledSpectrum betap = beta * T_maj / pdf`.
       Compute rescaled probability for absorption: `SampledSpectrum r_e = r_u * sigma_maj * T_maj / pdf`.
       Add emission: `if (r_e) L += betap * mp.sigma_a * mp.Le / r_e.Average()`.
       *Note: Do not cancel `T_maj` factors between `betap` and `r_e` before MIS weighting, as the final estimator divides by the average of rescaled probabilities, not component-wise [S1].*

    b. **Compute Medium Event Probabilities:**
       `Float pAbsorb = mp.sigma_a[0] / sigma_maj[0]`
       `Float pScatter = mp.sigma_s[0] / sigma_maj[0]`
       `Float pNull = std::max<Float>(0, 1 - pAbsorb - pScatter)`
       Sample mode: `Float um = rng.Uniform<Float>()`, `int mode = SampleDiscrete({pAbsorb, pScatter, pNull}, um)`.

    c. **Handle Absorption (Mode 0):**
       `terminated = true`, `return false`.

    d. **Handle Scattering (Mode 1):**
       If `depth++ >= maxDepth`, `terminated = true`, `return false`.
       Update beta and r_u: `Float pdf = T_maj[0] * mp.sigma_s[0]`, `beta *= T_maj * mp.sigma_s / pdf`, `r_u *= T_maj * mp.sigma_s / pdf`.
       If `beta && r_u`:
       Sample direct lighting: `MediumInteraction intr(p, -ray.d, ray.time, ray.medium, mp.phase)`, `L += SampleLd(intr, nullptr, lambda, sampler, beta, r_u)`.
       Sample new direction: `Point2f u = sampler.Get2D()`, `pstd::optional<PhaseFunctionSample> ps = intr.phase.Sample_p(-ray.d, u)`.
       If `!ps || ps->pdf == 0`, `terminated = true`.
       Else:
       `beta *= ps->p / ps->pdf`
       `r_l = r_u / ps->pdf`
       `prevIntrContext = LightSampleContext(intr)`
       `scattered = true`
       `ray.o = p`, `ray.d = ps->wi`
       `specularBounce = false`, `anyNonSpecularBounces = true`.
       `return false`.

    e. **Handle Null Scattering (Mode 2):**
       `SampledSpectrum sigma_n = ClampZero(sigma_maj - mp.sigma_a - mp.sigma_s)`
       `Float pdf = T_maj[0] * sigma_n[0]`
       `beta *= T_maj * sigma_n / pdf`
       If `pdf == 0`, `beta = SampledSpectrum(0.f)`
       `r_u *= T_maj * sigma_n / pdf`
       `r_l *= T_maj * sigma_maj / pdf`
       `return beta && r_u`.

12. **Handle Terminated, Scattered, and Unscattered Medium Rays:**
    If `terminated || !beta || !r_u`, `return L`.
    If `scattered`, `continue` (loop continues to next surface interaction).
    Else (unscattered):
    `beta *= T_maj / T_maj[0]`
    `r_u *= T_maj / T_maj[0]`
    `r_l *= T_maj / T_maj[0]`

**Edge Cases and Pitfalls:**
- **MIS Weighting:** Do not cancel out $T_{maj}$ factors between `betap` and `r_e` before MIS weighting. The final estimator divides by the average of rescaled probabilities, not component-wise. Cancelling leads to incorrect results [S1].
- **Emission Sampling:** Volumetric emission is added at every sampled point in the medium, not only when the absorption case is sampled, because emission is available via `MediumProperties` [S1].
- **Reservoir Sampling:** Ensure the reservoir sampler is seeded correctly and that intersections are only added if they belong to the same material as the entry point.

## Subtopics

### [BSDF Sampling for Volumetric Paths](1995-bsdf-sampling-for-volumetric-paths.md)
BSDF Sampling for Volumetric Paths handles the surface interaction component of path tracing within participating media, calculating indirect lighting contributions based on surface BRDFs and sampling strategies. It determines the new direction of a volumetric path upon hitting a surface, updating throughput and rescaled probabilities to account for the surface scattering event. Unlike pure volume integrators, this method bridges surface geometry and participating media by treating surface interactions as scattering events that redirect rays back into the medium or to other surfaces.

### [Reference Path Tracing](3170-reference-path-tracing.md)
Reference Path Tracing is an unbiased volumetric path tracer used as a ground-truth reference renderer for subsurface scattering and participating media. It solves the problem of accurate light transport in high-curvature surfaces and complex media by treating the subsurface layer as a volume, performing full random walks from entry to exit points. It sets itself apart from diffusion-based models by explicitly tracking geometric features for accurate shadowing and handling zero- and single-scatter events, while differing from sibling methods like BSDF Sampling for Volumetric Paths by relying on a pure random walk with Next-Event Estimation (NEE) rather than specialized BSDF sampling strategies.

### Chromatic Effects with MIS
A technique to handle non-monochromatic media efficiently by using Multiple Importance Sampling (MIS) between RGB channels during distance sampling, avoiding the inefficiency of picking a single wavelength [S2, unofficial]. It chooses $P(\lambda)$ using the current throughput instead of single scatter albedo to bound throughput and prevent fireflies from throughput spiking in deep paths with highly tinted extinction [S2, unofficial]. The total probability density is $P(s) = P(\lambda) * P_\lambda(s)$, where $P_\lambda(s) = \sigma_\lambda e^{-s\sigma_\lambda}$ [S2, unofficial]. The MIS weight is $W_\lambda(s) = P(\lambda)/(P(\lambda_R) + P(\lambda_G) + P(\lambda_B))$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear in the number of scattering events and surface interactions. Subsurface scattering adds overhead due to reservoir sampling and multiple ray intersections along the probe segment.
- **Memory Complexity:** Low, primarily storing path state and medium properties. Reservoir sampling requires constant memory.
- **Performance:** Efficient for homogeneous media. Subsurface scattering can be expensive if the probe segment intersects many geometry elements. Chromatic MIS reduces variance in tinted media compared to single-wavelength sampling.
- **Strengths:** Handles complex interactions between surface subsurface scattering and volumetric effects. Unbiased.
- **Limitations:** Computationally expensive for deep subsurface scattering or complex media. Reservoir sampling may fail if no valid intersections are found along the probe segment.

## Background
Originated in the context of physically based rendering (PBR) for film and high-end graphics. The method is described in "14.2 Volume Scattering Integrators" from the PBR Book [S1] and "Path Traced Subsurface Scattering Model" from Pixar [S2, unofficial]. It builds on unidirectional path tracing and volume scattering integrators, extending them to handle BSSRDFs and chromatic effects.

## Sources
[S1] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S2] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
