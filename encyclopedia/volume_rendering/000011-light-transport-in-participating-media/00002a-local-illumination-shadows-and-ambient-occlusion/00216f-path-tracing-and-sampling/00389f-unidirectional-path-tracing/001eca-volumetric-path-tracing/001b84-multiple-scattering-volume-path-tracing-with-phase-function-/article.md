# Multiple Scattering Volume Path Tracing with Phase Function Sampling

Also called: VolPathIntegrator, Volumetric Path Integrator

## Overview
Multiple Scattering Volume Path Tracing with Phase Function Sampling is a path tracing technique for simulating light transport in participating media by sampling distances from an exponential distribution, scattering according to a phase function, and handling absorption via Russian roulette or weight factors [S1, unofficial][S2]. It solves the problem of computing ground-truth radiance in scenes with multiple homogeneous volumes by explicitly sampling the phase function for direction changes and supporting index-matched surfaces [S2]. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md), which uses simple delta tracking and no explicit light sampling, this method handles multiple scattering events and tracks rescaled path probabilities to avoid floating-point overflow [S3]. It differs from [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md) by focusing on phase function sampling for direction rather than transmittance sampling for distance [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray, scene geometry, participating media properties ($\sigma_a, \sigma_s$), phase function $\rho$, maximum depth `maxDepth`, Russian roulette depth `rr_depth` [S2, S3].
*   **Output:** Radiance $L$ accumulated along the path [S3].

**Data Structures:**
*   **Medium Properties:** $\sigma_a$ (absorption coefficient), $\sigma_s$ (scattering coefficient), $\sigma_t = \sigma_a + \sigma_s$ (extinction coefficient), phase function $\rho$ [S1, unofficial][S3].
*   **State Variables:**
    *   `beta`: Path throughput weight (SampledSpectrum) [S3].
    *   `r_u`: Rescaled unidirectional path probability (SampledSpectrum) [S3].
    *   `r_l`: Rescaled light-sampling path probability (SampledSpectrum) [S3].
    *   `depth`: Current path depth (integer) [S3].
    *   `specularBounce`: Boolean flag for specular surface interactions [S3].
    *   `anyNonSpecularBounces`: Boolean flag indicating if a non-specular bounce has occurred [S3].
    *   `etaScale`: Scalar multiplier for transmission refractive index effects (Float) [S3].
    *   `prevIntrContext`: Light sample context for MIS (LightSampleContext) [S3].

**Formulas:**
*   **Distance Sampling:** $x = -(\ln \xi)/\sigma_t$, where $\xi$ is a uniform random number [S1, unofficial].
*   **Extinction Coefficient:** $\sigma_t = \sigma_s + \sigma_a$ [S1, unofficial].
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
    *   **Absorption (mode 0):** Set `terminated=true`, return `false` to end the callback [S3].
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
        *   Update `beta *= T_{maj} * \sigma_n / pdf`, `r_u *= T_{maj} * \sigma_n / pdf`, `r_l *= T_{maj} * \sigma_{maj} / pdf` [S3].
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
    *   **BSSRDF:** If BSSRDF exists and bounce is transmission, sample probe segment, find exit point, sample BSDF at exit, update `beta`, `r_l`, spawn new ray [S3].
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
def vol_path_integrator_li(ray, lambda_wl, sampler, scratch_buffer, visible_surf, max_depth, rr_depth, regularize, lights, light_sampler):
    L = Spectrum(0.0)
    beta = Spectrum(1.0)
    r_u = Spectrum(1.0)
    r_l = Spectrum(1.0)
    specular_bounce = False
    any_non_specular_bounces = False
    depth = 0
    eta_scale = 1.0
    prev_intr_context = LightSampleContext()

    while True:
        si = intersect_scene(ray)
        scattered = False
        terminated = False

        if ray.medium is not None:
            t_max = si.t_hit if si else float('inf')
            hash0 = hash(sampler.get_1d())
            hash1 = hash(sampler.get_1d())
            rng = RNG(hash0, hash1)

            T_maj = sample_majorant_transmittance(ray, t_max, sampler.get_1d(), rng, lambda_wl,
                lambda p, mp, sigma_maj, T_maj: handle_medium_event(p, mp, sigma_maj, T_maj, beta, r_u, r_l, depth, max_depth, lambda_wl, sampler, L))

            if terminated or not beta or not r_u:
                return L

            if scattered:
                continue

            beta *= T_maj / T_maj[0]
            r_u *= T_maj / T_maj[0]
            r_l *= T_maj / T_maj[0]

        if not si:
            for light in infinite_lights:
                Le = light.Le(ray, lambda_wl)
                if Le:
                    if depth == 0 or specular_bounce:
                        L += beta * Le / r_u.Average()
                    else:
                        light_pdf = light_sampler.PMF(prev_intr_context, light) * light.PDF_Li(prev_intr_context, ray.d, True)
                        r_l *= light_pdf
                        L += beta * Le / (r_u + r_l).Average()
            continue

        bsdf = si.GetBSDF(ray, lambda_wl, camera, scratch_buffer, sampler)
        if not bsdf:
            si.SkipIntersection(ray, si.t_hit)
            continue

        if depth == 0 and visible_surf:
            n_rho_samples = 16
            uc_rho = [0.75741637, 0.37870818, 0.7083487, 0.18935409, 0.9149363, 0.35417435, 0.5990858, 0.09467703, 0.8578725, 0.45746812, 0.686759, 0.17708716, 0.9674518, 0.2995429, 0.5083201, 0.047338516]
            u_rho = [Point2f(0.855985, 0.570367), Point2f(0.381823, 0.851844), Point2f(0.285328, 0.764262), Point2f(0.733380, 0.114073), Point2f(0.542663, 0.344465), Point2f(0.127274, 0.414848), Point2f(0.964700, 0.947162), Point2f(0.594089, 0.643463), Point2f(0.095109, 0.170369), Point2f(0.825444, 0.263359), Point2f(0.65, 0.152852), Point2f(0.180888, 0.214174), Point2f(0.898579, 0.503897)]
            albedo = bsdf.rho(si.wo, uc_rho, u_rho)
            visible_surf = VisibleSurface(si, albedo, lambda_wl)

        if depth >= max_depth:
            return L

        if regularize and any_non_specular_bounces:
            bsdf.Regularize()

        if is_non_specular(bsdf.Flags()):
            L += SampleLd(si, bsdf, lambda_wl, sampler, beta, r_u)
            prev_intr_context = LightSampleContext(si)

        wo = si.wo
        u = sampler.get_1d()
        bs = bsdf.Sample_f(wo, u, sampler.Get2D())
        if not bs:
            break

        beta *= bs.f * abs_dot(bs.wi, si.shading.n) / bs.pdf
        if bs.pdf_is_proportional:
            r_l = r_u / bsdf.PDF(wo, bs.wi)
        else:
            r_l = r_u / bs.pdf

        specular_bounce = bs.IsSpecular()
        any_non_specular_bounces |= not bs.IsSpecular()
        if bs.IsTransmission():
            eta_scale *= bs.eta ** 2

        ray = si.SpawnRay(ray, bsdf, bs.wi, bs.flags, bs.eta)

        bssrdf = si.GetBSSRDF(ray, lambda_wl, camera, scratch_buffer)
        if bssrdf and bs.IsTransmission():
            uc = sampler.get_1d()
            up = sampler.Get2D()
            probe_seg = bssrdf.SampleSp(uc, up)
            if not probe_seg:
                break

            seed = mix_bits(float_to_bits(sampler.get_1d()))
            interaction_sampler = WeightedReservoirSampler(seed)

            base = Interaction(probe_seg.p0, ray.time, Medium())
            while True:
                r = base.SpawnRayTo(probe_seg.p1)
                si_sub = intersect_scene(r, 1)
                if not si_sub:
                    break
                base = si_sub.intr
                if si_sub.intr.material == si.material:
                    interaction_sampler.add(SubsurfaceInteraction(si_sub.intr), 1.0)

            if not interaction_sampler.has_sample():
                break

            ssi = interaction_sampler.get_sample()
            bssrdf_sample = bssrdf.ProbeIntersectionToSample(ssi, scratch_buffer)
            if not bssrdf_sample.Sp or not bssrdf_sample.pdf:
                break

            pdf = interaction_sampler.sample_probability() * bssrdf_sample.pdf[0]
            beta *= bssrdf_sample.Sp / pdf
            r_u *= bssrdf_sample.Sp / pdf
            r_l = r_u / pdf
            specular_bounce = False
            any_non_specular_bounces = True

            Sw = bssrdf_sample.Sw
            if regularize:
                Sw.Regularize()

            L += SampleLd(pi, Sw, lambda_wl, sampler, beta, r_u)

            u = sampler.get_1d()
            bs = Sw.Sample_f(pi.wo, u, sampler.Get2D())
            if not bs:
                break
            beta *= bs.f * abs_dot(bs.wi, pi.shading.n) / bs.pdf
            r_l = r_u / bs.pdf
            specular_bounce = bs.IsSpecular()
            ray = RayDifferential(pi.SpawnRay(bs.wi))

        rr_beta = beta * eta_scale / r_u.Average()
        u_rr = sampler.get_1d()
        if rr_beta.max_component_value() < 1 and depth > 1:
            q = max(0.0, 1.0 - rr_beta.max_component_value())
            if u_rr < q:
                break
            beta /= 1.0 - q

        depth += 1

    return L

def handle_medium_event(p, mp, sigma_maj, T_maj, beta, r_u, r_l, depth, max_depth, lambda_wl, sampler, L):
    if depth < max_depth and mp.Le:
        pdf = sigma_maj[0] * T_maj[0]
        betap = beta * T_maj / pdf
        r_e = r_u * sigma_maj * T_maj / pdf
        if r_e:
            L += betap * mp.sigma_a * mp.Le / r_e.Average()

    p_absorb = mp.sigma_a[0] / sigma_maj[0]
    p_scatter = mp.sigma_s[0] / sigma_maj[0]
    p_null = max(0.0, 1.0 - p_absorb - p_scatter)
    
    um = rng.uniform()
    mode = sample_discrete([p_absorb, p_scatter, p_null], um)
    
    if mode == 0:
        terminated = True
        return False
    elif mode == 1:
        if depth >= max_depth:
            terminated = True
            return False
        pdf = T_maj[0] * mp.sigma_s[0]
        beta *= T_maj * mp.sigma_s / pdf
        r_u *= T_maj * mp.sigma_s / pdf
        
        if beta and r_u:
            intr = MediumInteraction(p, -ray.d, ray.time, ray.medium, mp.phase)
            L += SampleLd(intr, None, lambda_wl, sampler, beta, r_u)
            
            u = sampler.Get2D()
            ps = intr.phase.Sample_p(-ray.d, u)
            if not ps or ps.pdf == 0:
                terminated = True
            else:
                beta *= ps.p / ps.pdf
                r_l = r_u / ps.pdf
                prev_intr_context = LightSampleContext(intr)
                scattered = True
                ray.o = p
                ray.d = ps.wi
                specular_bounce = False
                any_non_specular_bounces = True
        return False
    else:
        sigma_n = clamp_zero(sigma_maj - mp.sigma_a - mp.sigma_s)
        pdf = T_maj[0] * sigma_n[0]
        beta *= T_maj * sigma_n / pdf
        if pdf == 0:
            beta = Spectrum(0.0)
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
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Robust for multiple scattering but can be slow for dense media due to many null scattering events or long path lengths [S3].
*   **Strengths:** Handles multiple scattering, index-matched surfaces, and chromatic media accurately [S3].
*   **When to Use:** Ground-truth rendering, scenes with complex volumetric interactions, index-matched boundaries [S2, S3].
*   **When Not to Use:** Not given in the sources.

## Background
Originated from Monte Carlo path tracing techniques for participating media, popularized by course notes from Steve Marschner and implemented in modern renderers like PBRT as `VolPathIntegrator` [S1, unofficial][S3]. It addresses the limitations of simpler methods like Dipole or diffusion models by providing a full Monte Carlo simulation of light transport [S1, unofficial].

## Sources
[S1] Full Monte-Carlo Volumetric Scattering, https://computergraphics.stackexchange.com/questions/2563/full-monte-carlo-volumetric-scattering, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
