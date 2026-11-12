# Multiple Scattering Volume Path Tracing with Phase Function Sampling

Also called: VolPathIntegrator, Volumetric Path Integrator

## Overview
A path tracing technique for simulating light transport in participating media where a ray enters a material, applies a BTDF, travels a distance determined by an exponential distribution, scatters according to a phase function, and repeats until exiting the material with another BTDF [S1, unofficial][S2, S3]. It solves the inefficiency of pure phase function sampling when light sources are small or distant by explicitly connecting path vertices to lights [S1, unofficial]. Unlike its parent method, which only samples the phase function, this method changes the integration domain from the hemisphere to the light surface area, requiring the computation of transmittance through volumes and the handling of index-matched boundaries [S1, unofficial][S3]. It differs from [Volumetric Path Tracing with BSSRDF and Medium Sampling](1867-volumetric-path-tracing-with-bssrdf-and-medium-sampling.md) by focusing on direct light connections via shadow rays rather than medium sampling for indirect illumination [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray, scene geometry, participating media properties (scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function $\rho$), light sources, maximum depth `maxDepth`, Russian roulette depth `rr_depth`, and a sampler.
*   **Output:** Radiance $L$ for the pixel.

**Data Structures:**
*   `SampledSpectrum`: Represents spectral radiance/throughput.
*   `Interaction`: Stores position, normal, time, medium, and phase function.
*   `BSDFSample`: Contains scattered direction `wi`, BSDF value `f`, PDF `pdf`, and flags.
*   `PhaseFunctionSample`: Contains scattered direction `wi`, phase function value `p`, and PDF `pdf`.
*   `MediumProperties`: Contains $\sigma_a$, $\sigma_s$, $\sigma_t = \sigma_a + \sigma_s$, and phase function.

**Parameters:**
*   `maxDepth`: Integer limiting the maximum number of bounces/scattering events [S3]. If -1, infinite bounces [S2]. `maxDepth=2` corresponds to single scattering [S2].
*   `rr_depth`: Controls when Russian roulette begins. RR starts when a light path has `rr_depth + 1` vertices [S2].
*   `regularize`: Boolean flag; if true, BSDFs are regularized after non-specular bounces to prevent numerical instability [S3].
*   `lightSampleStrategy`: String specifying the light sampling strategy (default "bvh") [S3].

**Method Steps:**

1.  **Initialization:**
    *   Initialize state variables: $L = 0$, $\beta = 1$, $r_u = 1$, $r_l = 1$ [S3].
    *   Initialize flags: `specularBounce = false`, `anyNonSpecularBounces = false` [S3].
    *   Initialize counters: `depth = 0`, `etaScale = 1` [S3].
    *   Initialize context: `prevIntrContext` [S3].

2.  **Ray Loop:**
    *   Intersect the current ray with the scene to get intersection `si` [S3].
    *   Determine `tMax` as the distance to the intersection or infinity if no intersection [S3].

3.  **Volumetric Sampling (if ray is in a medium):**
    *   Initialize RNG for sampling the majorant transmittance using two 1D samples from the sampler, hashed to `uint64_t` [S3].
    *   Sample majorant transmittance $T_{maj}$ using a callback function [S3].
    *   Inside the callback, at point $p$:
        *   Compute medium event probabilities:
            $$p_{Absorb} = \frac{\sigma_a[0]}{\sigma_{maj}[0]}$$
            $$p_{Scatter} = \frac{\sigma_s[0]}{\sigma_{maj}[0]}$$
            $$p_{Null} = \max(0, 1 - p_{Absorb} - p_{Scatter})$$
            [S3]
        *   Sample event type `mode` using a uniform random number `um` and `SampleDiscrete({pAbsorb, pScatter, pNull}, um)` [S3].
        *   **If `mode == 0` (Absorption):** Set `terminated = true` and return `false` [S3].
        *   **If `mode == 1` (Scattering):**
            *   If `depth >= maxDepth`, set `terminated = true` and return `false` [S3].
            *   Increment `depth` [S3].
            *   Compute PDF: $pdf = T_{maj}[0] \cdot \sigma_s[0]$ [S3].
            *   Update throughput: $\beta \leftarrow \beta \cdot \frac{T_{maj} \cdot \sigma_s}{pdf}$ [S3].
            *   Update rescaled unidirectional probability: $r_u \leftarrow r_u \cdot \frac{T_{maj} \cdot \sigma_s}{pdf}$ [S3].
            *   If $\beta$ and $r_u$ are non-zero, sample direct lighting at the volume-scattering event using `SampleLd(intr, nullptr, lambda, sampler, beta, r_u)` and add to $L$ [S3].
            *   Sample a new direction from the phase function by generating a 2D uniform sample `u` and calling `intr.phase.Sample_p(-ray.d, u)` [S3].
            *   If the result is null or has zero PDF, set `terminated = true` [S3].
            *   Else:
                *   Update $\beta \leftarrow \beta \cdot \frac{ps \to p}{ps \to pdf}$ [S3].
                *   Set $r_l = \frac{r_u}{ps \to pdf}$ [S3].
                *   Store the light sample context in `prevIntrContext` [S3].
                *   Mark `scattered = true` [S3].
                *   Update ray origin `ray.o = p` and direction `ray.d = ps \to wi` [S3].
                *   Set `specularBounce = false` and `anyNonSpecularBounces = true` [S3].
            *   Return `false` [S3].
        *   **If `mode == 2` (Null Scattering):**
            *   Compute $\sigma_n = \text{ClampZero}(\sigma_{maj} - \sigma_a - \sigma_s)$ [S3].
            *   Compute PDF: $pdf = T_{maj}[0] \cdot \sigma_n[0]$ [S3].
            *   If PDF is 0, set $\beta = 0$ [S3].
            *   Update $\beta \leftarrow \beta \cdot \frac{T_{maj} \cdot \sigma_n}{pdf}$ [S3].
            *   Update $r_u \leftarrow r_u \cdot \frac{T_{maj} \cdot \sigma_n}{pdf}$ [S3].
            *   Update $r_l \leftarrow r_l \cdot \frac{T_{maj} \cdot \sigma_{maj}}{pdf}$ [S3].
            *   Return $\beta \land r_u$ [S3].
    *   After the medium callback:
        *   If `terminated` or $\beta$ or $r_u$ is zero, return $L$ [S3].
        *   If `scattered`, continue the loop [S3].
        *   If unscattered, update $\beta$, $r_u$, and $r_l$ by multiplying by $\frac{T_{maj}}{T_{maj}[0]}$ to account for transmittance from the last event or origin to the max distance [S3].

4.  **Surface Interaction:**
    *   If no intersection (`!si`), accumulate contributions from infinite light sources [S3].
    *   Get the BSDF: `bsdf = isect.GetBSDF(...)` [S3].
    *   If BSDF is null, skip the intersection and continue [S3].
    *   If `depth == 0` and `visibleSurf` is set, estimate the BSDF's albedo using 16 pre-defined samples `ucRho` and `uRho` via `bsdf.rho(isect.wo, ucRho, uRho)` and store it in `*visibleSurf` [S3].
    *   If `depth >= maxDepth`, return $L$ [S3].
    *   If `regularize` and `anyNonSpecularBounces`, call `bsdf.Regularize()` [S3].
    *   If the BSDF is non-specular, sample direct lighting using `SampleLd(isect, &bsdf, lambda, sampler, beta, r_u)` and add to $L$ [S3].
    *   Store the surface interaction context in `prevIntrContext` [S3].
    *   Sample the BSDF to get a new direction: Generate 1D sample `u` and 2D sample. Call `bsdf.Sample_f(wo, u, sampler.Get2D())` [S3].
    *   If null, break the loop [S3].
    *   Update $\beta \leftarrow \beta \cdot \frac{bs \to f \cdot |\cos \theta|}{bs \to pdf}$ [S3].
    *   If `bs->pdfIsProportional` is true, set $r_l = \frac{r_u}{bsdf.PDF(wo, bs \to wi)}$, else $r_l = \frac{r_u}{bs \to pdf}$ [S3].
    *   Update path state: `specularBounce = bs->IsSpecular()`, `anyNonSpecularBounces |= !bs->IsSpecular()` [S3].
    *   If transmission, `etaScale *= Sqr(bs->eta)` [S3].
    *   Spawn the new ray [S3].
    *   If BSSRDF is present and the bounce is transmission, sample a BSSRDF probe segment using 1D sample `uc` and 2D sample `up` [S3]. If null, break [S3].
    *   Sample a random intersection along the probe segment using a seeded 1D sample [S3].
    *   Sample indirect subsurface scattering: Get `u`. Sample BSDF `bs = Sw.Sample_f(pi.wo, u, sampler.Get2D())` [S3]. If null, break [S3].
    *   Update $\beta \leftarrow \beta \cdot \frac{bs \to f \cdot |\cos \theta|}{bs \to pdf}$ [S3].
    *   Set $r_l = \frac{r_u}{bs \to pdf}$ [S3].
    *   Set `specularBounce = bs->IsSpecular()` [S3].
    *   Spawn new ray [S3].

5.  **Russian Roulette Termination:**
    *   Compute $rrBeta = \frac{\beta \cdot etaScale}{r_u.Average()}$ [S3].
    *   Get `uRR` [S3].
    *   If $rrBeta.MaxComponentValue() < 1$ and `depth > 1`:
        *   Compute $q = \max(0, 1 - rrBeta.MaxComponentValue())$ [S3].
        *   If $uRR < q$, break [S3].
        *   Else, $\beta \leftarrow \frac{\beta}{1 - q}$ [S3].

6.  **Return:**
    *   Return $L$ [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def VolPathIntegrator_Li(ray, lambda, sampler, scratchBuffer, visibleSurf, maxDepth, rr_depth, regularize, lights, lightSampler):
    L = SampledSpectrum(0.0)
    beta = SampledSpectrum(1.0)
    r_u = SampledSpectrum(1.0)
    r_l = SampledSpectrum(1.0)
    specularBounce = False
    anyNonSpecularBounces = False
    depth = 0
    etaScale = 1.0
    prevIntrContext = LightSampleContext()

    while True:
        si = Intersect(ray)
        tMax = si.tHit if si else Infinity

        if ray.medium:
            hash0 = Hash(sampler.Get1D())
            hash1 = Hash(sampler.Get1D())
            rng = RNG(hash0, hash1)
            
            scattered = False
            terminated = False
            
            T_maj = SampleT_maj(ray, tMax, sampler.Get1D(), rng, lambda, 
                lambda p, mp, sigma_maj, T_maj: HandleMediumEvent(p, mp, sigma_maj, T_maj, beta, r_u, r_l, depth, maxDepth, lambda, sampler, L, prevIntrContext, ray))
            
            if terminated or not beta or not r_u:
                return L
            
            if scattered:
                continue
            
            beta *= T_maj / T_maj[0]
            r_u *= T_maj / T_maj[0]
            r_l *= T_maj / T_maj[0]

        if not si:
            for light in infiniteLights:
                Le = light.Le(ray, lambda)
                if Le:
                    if depth == 0 or specularBounce:
                        L += beta * Le / r_u.Average()
                    else:
                        lightPDF = lightSampler.PMF(prevIntrContext, light) * light.PDF_Li(prevIntrContext, ray.d, True)
                        r_l *= lightPDF
                        L += beta * Le / (r_u + r_l).Average()
            continue

        bsdf = si.GetBSDF(ray, lambda, camera, scratchBuffer, sampler)
        if not bsdf:
            si.SkipIntersection(&ray, si.tHit)
            continue

        if depth == 0 and visibleSurf:
            nRhoSamples = 16
            ucRho = [0.75741637, 0.37870818, 0.7083487, 0.18935409, 0.9149363, 0.35417435, 0.5990858, 0.09467703, 0.8578725, 0.45746812, 0.686759, 0.17708716, 0.9674518, 0.2995429, 0.5083201, 0.047338516]
            uRho = [Point2f(0.855985, 0.570367), Point2f(0.381823, 0.851844), Point2f(0.285328, 0.764262), Point2f(0.733380, 0.114073), Point2f(0.542663, 0.344465), Point2f(0.127274, 0.414848), Point2f(0.964700, 0.947162), Point2f(0.594089, 0.643463), Point2f(0.095109, 0.170369), Point2f(0.825444, 0.263359), Point2f(0.65, 0.152852), Point2f(0.180888, 0.214174), Point2f(0.898579, 0.503897)] # Truncated for brevity, see [S3]
            albedo = bsdf.rho(si.wo, ucRho, uRho)
            visibleSurf = VisibleSurface(si, albedo, lambda)

        depth += 1
        if depth > maxDepth:
            return L

        if regularize and anyNonSpecularBounces:
            bsdf.Regularize()

        if IsNonSpecular(bsdf.Flags()):
            L += SampleLd(si, &bsdf, lambda, sampler, beta, r_u)
            prevIntrContext = LightSampleContext(si)

        wo = si.wo
        u = sampler.Get1D()
        bs = bsdf.Sample_f(wo, u, sampler.Get2D())
        if not bs:
            break

        beta *= bs.f * AbsDot(bs.wi, si.shading.n) / bs.pdf
        if bs.pdfIsProportional:
            r_l = r_u / bsdf.PDF(wo, bs.wi)
        else:
            r_l = r_u / bs.pdf

        specularBounce = bs.IsSpecular()
        anyNonSpecularBounces |= not bs.IsSpecular()
        if bs.IsTransmission():
            etaScale *= Sqr(bs.eta)
        
        ray = si.SpawnRay(ray, bsdf, bs.wi, bs.flags, bs.eta)

        bssrdf = si.GetBSSRDF(ray, lambda, camera, scratchBuffer)
        if bssrdf and bs.IsTransmission():
            uc = sampler.Get1D()
            up = sampler.Get2D()
            probeSeg = bssrdf.SampleSp(uc, up)
            if not probeSeg:
                break
            
            seed = MixBits(FloatToBits(sampler.Get1D()))
            interactionSampler = WeightedReservoirSampler<SubsurfaceInteraction>(seed)
            
            base = Interaction(probeSeg.p0, ray.time, Medium())
            while True:
                r = base.SpawnRayTo(probeSeg.p1)
                si_sub = Intersect(r, 1)
                if not si_sub:
                    break
                base = si_sub.intr
                if si_sub.intr.material == si.material:
                    interactionSampler.Add(SubsurfaceInteraction(si_sub.intr), 1.0)
            
            if not interactionSampler.HasSample():
                break
            
            ssi = interactionSampler.GetSample()
            bssrdfSample = bssrdf.ProbeIntersectionToSample(ssi, scratchBuffer)
            if not bssrdfSample.Sp or not bssrdfSample.pdf:
                break
            
            pdf = interactionSampler.SampleProbability() * bssrdfSample.pdf[0]
            beta *= bssrdfSample.Sp / pdf
            r_u *= bssrdfSample.Sp / pdf
            r_l = r_u / pdf
            specularBounce = False
            anyNonSpecularBounces = True
            if regularize:
                Sw.Regularize()
            L += SampleLd(pi, &Sw, lambda, sampler, beta, r_u)
            
            u = sampler.Get1D()
            bs = Sw.Sample_f(pi.wo, u, sampler.Get2D())
            if not bs:
                break
            beta *= bs.f * AbsDot(bs.wi, pi.shading.n) / bs.pdf
            r_l = r_u / bs.pdf
            specularBounce = bs.IsSpecular()
            ray = RayDifferential(pi.SpawnRay(bs.wi))

        rrBeta = beta * etaScale / r_u.Average()
        uRR = sampler.Get1D()
        if rrBeta.MaxComponentValue() < 1 and depth > 1:
            q = max(0.0, 1.0 - rrBeta.MaxComponentValue())
            if uRR < q:
                break
            beta /= 1.0 - q

    return L

def HandleMediumEvent(p, mp, sigma_maj, T_maj, beta, r_u, r_l, depth, maxDepth, lambda, sampler, L, prevIntrContext, ray):
    pAbsorb = mp.sigma_a[0] / sigma_maj[0]
    pScatter = mp.sigma_s[0] / sigma_maj[0]
    pNull = max(0.0, 1.0 - pAbsorb - pScatter)
    
    um = rng.Uniform()
    mode = SampleDiscrete([pAbsorb, pScatter, pNull], um)
    
    if mode == 0:
        return False # Absorbed
    
    if mode == 1:
        if depth >= maxDepth:
            return False # Max depth reached
        
        depth += 1
        pdf = T_maj[0] * mp.sigma_s[0]
        beta *= T_maj * mp.sigma_s / pdf
        r_u *= T_maj * mp.sigma_s / pdf
        
        if beta and r_u:
            intr = MediumInteraction(p, -ray.d, ray.time, ray.medium, mp.phase)
            L += SampleLd(intr, None, lambda, sampler, beta, r_u)
            
            u = sampler.Get2D()
            ps = intr.phase.Sample_p(-ray.d, u)
            if not ps or ps.pdf == 0:
                return False # Terminated
            
            beta *= ps.p / ps.pdf
            r_l = r_u / ps.pdf
            prevIntrContext = LightSampleContext(intr)
            scattered = True
            ray.o = p
            ray.d = ps.wi
            specularBounce = False
            anyNonSpecularBounces = True
        
        return False
    
    if mode == 2:
        sigma_n = ClampZero(sigma_maj - mp.sigma_a - mp.sigma_s)
        pdf = T_maj[0] * sigma_n[0]
        beta *= T_maj * sigma_n / pdf
        if pdf == 0:
            beta = SampledSpectrum(0.0)
        r_u *= T_maj * sigma_n / pdf
        r_l *= T_maj * sigma_maj / pdf
        return beta and r_u

    return False
```

## Subtopics

### [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md)
Next Event Estimation (NEE) for volumetric path tracing is an optimization that adds direct lighting estimation to multiple scattering volume path tracing by sampling points on light sources and tracing shadow rays through participating media and index-matching surfaces. It solves the inefficiency of pure phase function sampling when light sources are small or distant by explicitly connecting path vertices to lights. Unlike its parent method, which only samples the phase function, this method changes the integration domain from the hemisphere to the light surface area, requiring the computation of transmittance through volumes and the handling of index-matched boundaries. It differs from [Volumetric Path Tracing with BSSRDF and Medium Sampling](1867-volumetric-path-tracing-with-bssrdf-and-medium-sampling.md) by focusing on direct light connections via shadow rays rather than medium sampling for indirect illumination.

### [Volumetric Path Tracing with BSSRDF and Medium Sampling](1867-volumetric-path-tracing-with-bssrdf-and-medium-sampling.md)
This method handles attenuated subsurface scattering by sampling a BSSRDF probe segment, finding an exit point on the same material, and treating it as a surface scattering event with direct and indirect illumination. It extends unidirectional path tracing to integrate subsurface transport alongside participating media sampling.

## Operating characteristics
*   **Time Complexity:** Depends on the number of scattering events, which is governed by the optical thickness of the medium and `maxDepth`. Each scattering event requires phase function sampling and potentially direct lighting estimation [S3].
*   **Memory Complexity:** Low, primarily storing the current path state and interaction contexts [S3].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance degrades with higher optical thickness due to more scattering events [S1, unofficial].
*   **Strengths:** Robustly handles both surface and volume transport, including volumetric emission, chromatic media, and multiple scattering in participating media [S3]. Handles index-matched surfaces [S2].
*   **Limitations:** Can be slow for dense media due to many scattering events [S1, unofficial]. Non-monochromatic media require separate channel simulation or careful handling to avoid inefficiency [S1, unofficial].
*   **When to use:** When accurate multiple scattering in participating media is required, especially with complex surface interactions [S3].
*   **When not:** When real-time performance is critical and approximations (like diffusion models) are acceptable [S1, unofficial].

## Background
The VolPathIntegrator is described in "Physically Based Rendering: From Theory To Implementation" (PBRT), specifically in Chapter 14 on Volume Scattering Integrators [S3]. It pulls together ideas from multiple scattering, index-matched surfaces, and rescaled path probabilities to robustly handle light transport in participating media [S3]. The method builds on the foundational work of Marschner [S1, unofficial] and the UCSD course notes [S2].

## Sources
[S1] Full Monte-Carlo Volumetric Scattering, https://computergraphics.stackexchange.com/questions/2563/full-monte-carlo-volumetric-scattering, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
