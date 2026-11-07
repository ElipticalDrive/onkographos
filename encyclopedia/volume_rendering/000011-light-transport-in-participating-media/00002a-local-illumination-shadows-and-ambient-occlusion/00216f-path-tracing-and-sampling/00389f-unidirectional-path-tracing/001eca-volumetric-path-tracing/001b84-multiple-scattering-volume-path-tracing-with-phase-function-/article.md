# Multiple Scattering Volume Path Tracing with Phase Function Sampling

Also called: VolPathIntegrator, Volumetric Path Integrator

## Overview
Multiple Scattering Volume Path Tracing with Phase Function Sampling is a path tracing technique for simulating light transport in participating media where a ray enters a material, applies a BTDF, travels a distance determined by an exponential distribution, scatters according to a phase function, and repeats until exiting the material with another BTDF [S1, unofficial][S2]. It solves the problem of computing ground-truth radiance in scenes with multiple homogeneous volumes by explicitly sampling the phase function for direction changes and handling absorption via Russian roulette or weight factors [S2]. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md), which uses simple delta tracking and no explicit light sampling, this method handles multiple scattering events and supports index-matched surfaces [S2]. It differs from [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md) by focusing on phase function sampling for direction rather than transmittance sampling for distance [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray, scene geometry, participating media properties ($\sigma_a, \sigma_s$), phase function $\rho$, maximum depth `maxDepth`, Russian roulette depth `rr_depth`.
*   **Output:** Radiance $L$ accumulated along the path.

**Data Structures:**
*   **Medium Properties:** $\sigma_a$ (absorption coefficient), $\sigma_s$ (scattering coefficient), $\sigma_t = \sigma_a + \sigma_s$ (extinction coefficient), phase function $\rho$.
*   **State Variables:**
    *   `beta`: Path throughput weight (SampledSpectrum).
    *   `r_u`: Rescaled unidirectional path probability (SampledSpectrum).
    *   `r_l`: Rescaled light-sampling path probability (SampledSpectrum).
    *   `depth`: Current path depth (integer).
    *   `specularBounce`: Boolean flag for specular surface interactions.
    *   `anyNonSpecularBounces`: Boolean flag indicating if a non-specular bounce has occurred.
    *   `etaScale`: Scalar multiplier for transmission refractive index effects (Float).
    *   `prevIntrContext`: Light sample context for MIS (LightSampleContext).

**Formulas:**
*   **Distance Sampling:** $x = -(\ln \xi)/\sigma_t$, where $\xi$ is a uniform random number [S1, unofficial].
*   **Scattering Coefficient:** $\sigma = \sigma_s + \sigma_a$ [S1, unofficial].
*   **Rescaled Path Probabilities:**
    $$r_{u, \lambda, i}(\bar{p}_n) = \frac{p_{u, \lambda, i}(\bar{p}_n)}{p_{path}(\bar{p}_n)}$$
    $$r_{l, \lambda, i}(\bar{p}_n) = \frac{p_{l, \lambda, i}(\bar{p}_n)}{p_{path}(\bar{p}_n)}$$
    where $p_{path}$ is the probability for sampling the current path [S3].
*   **MIS Weight (Unidirectional):**
    $$w_u(\bar{p}_n) = \frac{1}{\frac{1}{m} \sum_{i=1}^m r_{u, \lambda, i}(\bar{p}_n) + \sum_{i=1}^m r_{l, \lambda, i}(\bar{p}_n)}$$
    [S3].
*   **Russian Roulette Probability:**
    $$P_{rr} = \min \left( \frac{\text{contrib}(\text{path})}{p(\text{path})}, 0.95 \right)$$
    [S2].

**Parameters:**
*   `maxDepth`: Integer limiting maximum bounces. If -1, infinite bounces. `maxDepth=2` corresponds to single scattering [S2, S3].
*   `rr_depth`: Russian roulette starts when path has `rr_depth + 1` vertices [S2].
*   `regularize`: Boolean; if true, BSDFs are regularized after non-specular bounces [S3].
*   `lightSampleStrategy`: String for light sampling (default "bvh") [S3].

**Algorithm Steps:**
1.  **Initialization:** Initialize `L=0`, `beta=1`, `r_u=1`, `r_l=1`, `depth=0`, `etaScale=1`, `specularBounce=false`, `anyNonSpecularBounces=false` [S3].
2.  **Ray Intersection:** Intersect ray with scene. If no intersection, treat as infinite light source [S3].
3.  **Medium Sampling:** If ray is in a medium:
    *   Generate RNG from sampler hashes [S3].
    *   Sample majorant transmittance $T_{maj}$ [S3].
    *   Compute event probabilities: $p_{Absorb} = \sigma_a/\sigma_{maj}$, $p_{Scatter} = \sigma_s/\sigma_{maj}$, $p_{Null} = \max(0, 1 - p_{Absorb} - p_{Scatter})$ [S3].
    *   Sample event type `mode` using uniform random `um` [S3].
    *   **Absorption (mode 0):** Set `terminated=true`, return `L` [S3].
    *   **Scattering (mode 1):**
        *   If `depth >= maxDepth`, terminate [S3].
        *   Update `beta *= T_{maj} * \sigma_s / pdf` and `r_u *= T_{maj} * \sigma_s / pdf` [S3].
        *   If `beta` and `r_u` are non-zero, sample direct lighting `SampleLd` [S3].
        *   Sample new direction from phase function: `ps = phase.Sample_p(-ray.d, u)` [S3].
        *   If `ps` is null or `pdf==0`, terminate [S3].
        *   Update `beta *= ps->p / ps->pdf`, `r_l = r_u / ps->pdf` [S3].
        *   Update ray origin/direction, set `scattered=true` [S3].
    *   **Null Scattering (mode 2):**
        *   Compute $\sigma_n = \max(0, \sigma_{maj} - \sigma_a - \sigma_s)$ [S3].
        *   Update `beta`, `r_u`, `r_l` using $\sigma_n$ and $\sigma_{maj}$ [S3].
        *   If `pdf==0`, set `beta=0` [S3].
    *   **Post-Medium:** If terminated or `beta/r_u` zero, return `L`. If scattered, continue loop. If unscattered, update `beta, r_u, r_l *= T_{maj} / T_{maj}[0]` [S3].
4.  **Surface Interaction:**
    *   Get BSDF. If null, skip intersection [S3].
    *   If `depth==0`, estimate albedo using 16 pre-defined samples and store in `visibleSurf` [S3].
    *   If `depth >= maxDepth`, return `L` [S3].
    *   If `regularize` and `anyNonSpecularBounces`, call `bsdf.Regularize()` [S3].
    *   If BSDF is non-specular, sample direct lighting `SampleLd` [S3].
    *   Sample BSDF direction `bs = bsdf.Sample_f(wo, u, sampler.Get2D())` [S3].
    *   If `bs` is null, break loop [S3].
    *   Update `beta *= bs->f * |n \cdot wi| / bs->pdf` [S3].
    *   Update `r_l`: if `bs->pdfIsProportional`, `r_l = r_u / bsdf.PDF(wo, bs->wi)`, else `r_l = r_u / bs->pdf` [S3].
    *   Update state: `specularBounce`, `anyNonSpecularBounces`, `etaScale` (if transmission) [S3].
    *   Spawn new ray [S3].
    *   **BSSRDF:** If BSSRDF exists and transmission, sample probe segment, find exit point, sample BSDF at exit, update `beta`, `r_l`, spawn new ray [S3].
5.  **Russian Roulette:**
    *   Compute `rrBeta = beta * etaScale / r_u.Average()` [S3].
    *   If `rrBeta.MaxComponentValue() < 1` and `depth > 1`:
        *   $q = \max(0, 1 - rrBeta.MaxComponentValue())$ [S3].
        *   If random `uRR < q`, break [S3].
        *   Else, `beta /= 1 - q` [S3].
6.  **Return:** Return `L` [S3].

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
        scattered = False
        terminated = False

        if ray.medium:
            tMax = si.tHit if si else Infinity
            hash0 = Hash(sampler.Get1D())
            hash1 = Hash(sampler.Get1D())
            rng = RNG(hash0, hash1)
            
            # Sample majorant transmittance
            T_maj = SampleT_maj(ray, tMax, sampler.Get1D(), rng, lambda, 
                lambda p, mp, sigma_maj, T_maj: HandleMediumEvent(p, mp, sigma_maj, T_maj, beta, r_u, r_l, depth, maxDepth, lambda, sampler, L))
            
            if terminated or not beta or not r_u:
                return L
            
            if scattered:
                continue
            
            # Unscattered ray update
            beta *= T_maj / T_maj[0]
            r_u *= T_maj / T_maj[0]
            r_l *= T_maj / T_maj[0]

        if not si:
            # Infinite lights
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

        if depth >= maxDepth:
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
            
            Sw = bssrdfSample.Sw
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

        depth += 1

    return L

def HandleMediumEvent(p, mp, sigma_maj, T_maj, beta, r_u, r_l, depth, maxDepth, lambda, sampler, L):
    if depth < maxDepth and mp.Le:
        pdf = sigma_maj[0] * T_maj[0]
        betap = beta * T_maj / pdf
        r_e = r_u * sigma_maj * T_maj / pdf
        if r_e:
            L += betap * mp.sigma_a * mp.Le / r_e.Average()

    pAbsorb = mp.sigma_a[0] / sigma_maj[0]
    pScatter = mp.sigma_s[0] / sigma_maj[0]
    pNull = max(0.0, 1.0 - pAbsorb - pScatter)
    
    um = rng.Uniform()
    mode = SampleDiscrete([pAbsorb, pScatter, pNull], um)
    
    if mode == 0:
        terminated = True
        return False
    elif mode == 1:
        if depth >= maxDepth:
            terminated = True
            return False
        pdf = T_maj[0] * mp.sigma_s[0]
        beta *= T_maj * mp.sigma_s / pdf
        r_u *= T_maj * mp.sigma_s / pdf
        
        if beta and r_u:
            intr = MediumInteraction(p, -ray.d, ray.time, ray.medium, mp.phase)
            L += SampleLd(intr, nullptr, lambda, sampler, beta, r_u)
            
            u = sampler.Get2D()
            ps = intr.phase.Sample_p(-ray.d, u)
            if not ps or ps.pdf == 0:
                terminated = True
            else:
                beta *= ps.p / ps.pdf
                r_l = r_u / ps.pdf
                prevIntrContext = LightSampleContext(intr)
                scattered = True
                ray.o = p
                ray.d = ps.wi
                specularBounce = False
                anyNonSpecularBounces = True
        return False
    else:
        sigma_n = ClampZero(sigma_maj - mp.sigma_a - mp.sigma_s)
        pdf = T_maj[0] * sigma_n[0]
        beta *= T_maj * sigma_n / pdf
        if pdf == 0:
            beta = SampledSpectrum(0.0)
        r_u *= T_maj * sigma_n / pdf
        r_l *= T_maj * sigma_maj / pdf
        return beta and r_u
```

**Edge Cases and Pitfalls:**
*   **Double Counting:** Do not apply Beer-Lambert law explicitly to path weights if distance was importance-sampled from the exponential distribution, as attenuation is implicit in the sampling [S1, unofficial][S3].
*   **Zero PDF:** If phase function sampling fails or returns zero PDF, terminate the path [S3]. If null scattering PDF is zero, set `beta=0` [S3]. If BSDF sampling fails, break the loop [S3].
*   **Chromatic Media:** For non-monochromatic coefficients, assign each ray a single wavelength or color channel to simulate separately, as different wavelengths have different scattering strengths [S1, unofficial].
*   **Index-Matched Surfaces:** Surfaces with `material_id == -1` are passed through, counting as one bounce, and the medium is updated based on ray direction and geometry normal [S2].

## Subtopics

### [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md)
Adds direct lighting estimation by sampling light sources and tracing shadow rays, solving inefficiency with small/distant lights.

### [Volumetric Path Tracing with BSSRDF and Medium Sampling](1867-volumetric-path-tracing-with-bssrdf-and-medium-sampling.md)
Extends to subsurface scattering on surfaces coupled with participating media, sampling BSSRDF probe segments.

## Operating characteristics
*   **Time Complexity:** Linear in the number of scattering events and bounces. Complexity increases with optical depth and `maxDepth`.
*   **Memory Complexity:** Low, primarily storing path state variables.
*   **Performance:** Robust for multiple scattering but can be slow for dense media due to many null scattering events or long path lengths [S3].
*   **Strengths:** Handles multiple scattering, index-matched surfaces, and chromatic media accurately [S3].
*   **Limitations:** Inefficient for small/distant lights without NEE; high variance in dense media [S1, unofficial][S3].
*   **When to Use:** Ground-truth rendering, scenes with complex volumetric interactions, index-matched boundaries [S2, S3].
*   **When Not to Use:** Real-time rendering, scenes with very sparse lighting where NEE is preferred, or when diffusion approximations are sufficient [S1, unofficial].

## Background
Originated from Monte Carlo path tracing techniques for participating media, popularized by course notes from Steve Marschner and implemented in modern renderers like PBRT as `VolPathIntegrator` [S1, unofficial][S3]. It addresses the limitations of simpler methods like Dipole or diffusion models by providing a full Monte Carlo simulation of light transport [S1, unofficial].

## Sources
[S1] Full Monte-Carlo Volumetric Scattering, https://computergraphics.stackexchange.com/questions/2563/full-monte-carlo-volumetric-scattering, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
