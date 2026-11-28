# Volumetric Path Tracing with BSSRDF and Medium Sampling

Also called: BSSRDF integration, Path Tracing with Subsurface Scattering and Participating Media, Subsurface Scattering Integration in Volumetric Path Tracing, Subsurface scattering in VolPathIntegrator, VolPathIntegrator with BSSRDF

## Overview
This method extends volumetric path tracing to handle participating media coupled with subsurface scattering on surfaces. It solves the problem of light transport where rays enter a surface, scatter within a subsurface layer, and exit at a different location, all while potentially traversing participating media. It differs from [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by explicitly sampling BSSRDF probe segments to find exit points on the same material, treating the subsurface interaction as a surface scattering event with direct and indirect illumination, rather than relying solely on phase function sampling within homogeneous volumes.

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera ray, scene geometry, materials (including BSSRDFs), participating media properties, random number generator state, path state (`beta`, `r_u`, `depth`, `maxDepth`, `regularize`).
- **Outputs:** Radiance contribution `L`.

**Data Structures:**
- `BSSRDFProbeSegment`: Contains entry point `p0` and exit point `p1` on the surface.
- `WeightedReservoirSampler<T>`: Reservoir sampling structure to select a random intersection along the probe segment.
- `BSSRDFSample`: Contains subsurface scattering weight `Sp`, PDF `pdf`, outgoing direction `wo`, and the associated BSDF `Sw`.
- `SubsurfaceInteraction`: Intersection data on the surface.

**Parameters:**
- `maxDepth`: Maximum path depth. If -1, infinite. Otherwise, max bounces = `maxDepth + 1` vertices [S1].
- `regularize`: Boolean flag to regularize the subsurface BSDF [S2].

**Steps:**

1. **Initialize Path:**
   - Set `L = 0`, `beta = 1`, `r_u = 1`, `depth = 0`.
   - Sample primary ray from camera.

2. **Main Loop:**
   - Intersect ray with scene. Let `si` be the intersection (or null if in medium).
   - **Sample Medium:**
     - If the ray is in a medium, sample the distance to the next event using the majorant transmittance `T_maj`.
     - Compute medium event probabilities:
       $$p_{\text{Absorb}} = \frac{\sigma_a[0]}{\sigma_{\text{maj}}[0]}, \quad p_{\text{Scatter}} = \frac{\sigma_s[0]}{\sigma_{\text{maj}}[0]}, \quad p_{\text{Null}} = \max(0, 1 - p_{\text{Absorb}} - p_{\text{Scatter}})$$
     - Sample mode `m` from `{p_Absorb, p_Scatter, p_Null}`.
     - **Mode 0 (Absorption):** Set `terminated = true`. Return `L`.
     - **Mode 1 (Scattering):**
       - If `depth >= maxDepth`, set `terminated = true`. Return `L`.
       - Update `beta` and `r_u`:
         $$\text{pdf} = T_{\text{maj}}[0] \cdot \sigma_s[0]$$
         $$\beta \leftarrow \beta \cdot \frac{T_{\text{maj}} \cdot \sigma_s}{\text{pdf}}, \quad r_u \leftarrow r_u \cdot \frac{T_{\text{maj}} \cdot \sigma_s}{\text{pdf}}$$
       - Add emission: If `mp.Le` exists, compute `betap` and `r_e`. Add to `L`:
         $$L \leftarrow L + \text{betap} \cdot \sigma_a \cdot L_e / r_e.\text{Average}()$$
       - Sample direct lighting: `L += SampleLd(intr, nullptr, lambda, sampler, beta, r_u)`.
       - Sample phase function: `ps = intr.phase.Sample_p(-ray.d, u)`.
       - If `ps` valid:
         $$\beta \leftarrow \beta \cdot \frac{ps.p}{ps.pdf}, \quad r_l \leftarrow \frac{r_u}{ps.pdf}$$
         $$\text{scattered} = \text{true}$$
         $$\text{ray}.o = p, \quad \text{ray}.d = ps.wi$$
         $$\text{specularBounce} = \text{false}, \quad \text{anyNonSpecularBounces} = \text{true}$$
       - Return `L` if terminated or invalid `beta`/`r_u`.
     - **Mode 2 (Null Scattering):**
       - $\sigma_n = \text{ClampZero}(\sigma_{\text{maj}} - \sigma_a - \sigma_s)$.
       - $\text{pdf} = T_{\text{maj}}[0] \cdot \sigma_n[0]$.
       - $\beta \leftarrow \beta \cdot \frac{T_{\text{maj}} \cdot \sigma_n}{\text{pdf}}$. If `pdf == 0`, $\beta = 0$.
       - $r_u \leftarrow r_u \cdot \frac{T_{\text{maj}} \cdot \sigma_n}{\text{pdf}}$.
       - $r_l \leftarrow r_l \cdot \frac{T_{\text{maj}} \cdot \sigma_{\text{maj}}}{\text{pdf}}$.
       - Return `beta && r_u`.
     - If `scattered`, continue to next iteration.
     - If not scattered, update `beta`, `r_u`, `r_l` by $T_{\text{maj}} / T_{\text{maj}}[0]$.

   - **Surface Interaction:**
     - If `si` is null or medium sampling terminated, break.
     - Get BSDF sample `bs` from surface.
     - Update path state:
       $$\text{specularBounce} = bs.\text{IsSpecular}()$$
       $$\text{anyNonSpecularBounces} \leftarrow \text{anyNonSpecularBounces} \lor \neg bs.\text{IsSpecular}()$$
       If transmission, update `etaScale`.
       Spawn new ray.

     - **BSSRDF Handling:**
       - Get `bssrdf = isect.GetBSSRDF(ray, lambda, camera, scratchBuffer)`.
       - If `bssrdf` exists and `bs.IsTransmission()`:
         - Sample probe segment: `probeSeg = bssrdf.SampleSp(uc, up)`. If null, break.
         - Reservoir Sampling:
           - `seed = MixBits(FloatToBits(sampler.Get1D()))`.
           - `interactionSampler = WeightedReservoirSampler<SubsurfaceInteraction>(seed)`.
           - `base = Interaction(probeSeg->p0, ray.time, Medium())`.
           - Loop:
             - `r = base.SpawnRayTo(probeSeg->p1)`.
             - `si = Intersect(r, 1)`. If null, break.
             - `base = si->intr`.
             - If `si->intr.material == isect.material`, `interactionSampler.Add(SubsurfaceInteraction(si->intr), 1.f)`.
           - If `!interactionSampler.HasSample()`, break.
         - Convert to sample:
           - `ssi = interactionSampler.GetSample()`.
           - `bssrdfSample = bssrdf.ProbeIntersectionToSample(ssi, scratchBuffer)`.
           - If `!bssrdfSample.Sp` or `!bssrdfSample.pdf`, break.
         - Update path state:
           $$\text{pdf} = \text{interactionSampler.SampleProbability}() \cdot bssrdfSample.pdf[0]$$
           $$\beta \leftarrow \beta \cdot \frac{bssrdfSample.Sp}{\text{pdf}}, \quad r_u \leftarrow r_u \cdot \frac{bssrdfSample.pdf}{bssrdfSample.pdf[0]}$$
           - `pi = ssi`, `pi.wo = bssrdfSample.wo`.
           - `prevIntrContext = LightSampleContext(pi)`.
         - Regularize BSDF if `regularize`: `Sw.Regularize()`. `anyNonSpecularBounces = true`.
         - Direct Illumination: `L += SampleLd(pi, &Sw, lambda, sampler, beta, r_u)`.
         - Indirect Subsurface Scattering:
           - `bs = Sw.Sample_f(pi.wo, u, sampler.Get2D())`. If null, break.
           - $\beta \leftarrow \beta \cdot \frac{bs.f \cdot \text{AbsDot}(bs.wi, pi.shading.n)}{bs.pdf}$.
           - $r_l \leftarrow \frac{r_u}{bs.pdf}$.
           - `specularBounce = bs.IsSpecular()`.
           - `ray = RayDifferential(pi.SpawnRay(bs.wi))`.

     - **Russian Roulette:**
       - `rrBeta = beta * etaScale / r_u.Average()`.
       - `uRR = sampler.Get1D()`.
       - If `rrBeta.MaxComponentValue() < 1 && depth > 1`:
         - `q = \max(0, 1 - rrBeta.MaxComponentValue())`.
         - If `uRR < q`, break.
         - $\beta \leftarrow \beta / (1 - q)$.

     - Increment `depth`.

3. **Return:** `L`.

**Pitfalls:**
- Do not cancel out $T_{\text{maj}}$ factors between `betap` and `r_e` before MIS weighting, because the final estimator divides by the average of rescaled probabilities, not component-wise [S2].
- Index-matched surfaces (`material_id == -1`) do not reflect/refract but change the medium. The ray passes through, counting as a bounce [S1].
- Volumetric emission is added at every sampled point in the medium, not only when absorption is sampled [S2].

## Subtopics

### [BSDF Sampling for Volumetric Paths](1995-bsdf-sampling-for-volumetric-paths.md)
Handles surface interaction component, determining new volumetric path direction upon surface intersection.

### [Reference Path Tracing](3170-reference-path-tracing.md)
Unbiased reference renderer treating subsurface layer as volume, performing full random walks.

### [Chromatic Effects with MIS](5305-chromatic-effects-with-mis.md)
Handles non-monochromatic media using MIS between RGB channels during distance sampling.

## Operating characteristics
- **Time Complexity:** Depends on the number of bounces and the cost of BSSRDF sampling (reservoir sampling requires intersecting the probe segment with scene geometry).
- **Memory Complexity:** Low, primarily storing path state and scratch buffers.
- **Strengths:** Handles complex subsurface scattering coupled with participating media accurately.
- **Limitations:** BSSRDF sampling can be expensive due to multiple intersections. Reservoir sampling may fail if no valid intersections are found along the probe segment.

## Background
Originated in the context of advanced volumetric path tracing implementations, specifically integrating subsurface scattering into the path tracer. The method is detailed in the PBRT v4 book [S2] and related course materials [S1].

## Sources
[S1] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
[S2] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
