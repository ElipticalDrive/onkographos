# Volumetric Path Tracing with BSSRDF and Medium Sampling

Also called: BSSRDF integration, Path Tracing with Subsurface Scattering and Participating Media, Subsurface Scattering Integration in Volumetric Path Tracing, Subsurface scattering in VolPathIntegrator, VolPathIntegrator with BSSRDF

## Overview
This method extends volumetric path tracing to handle subsurface scattering by integrating BSSRDF (Bidirectional Surface Scattering Reflectance Distribution Function) probes directly into the path traversal loop, allowing light to enter a material, scatter internally, and exit at a different surface point before continuing as a standard surface interaction [S1]. It solves the problem of rendering translucent materials (like skin, wax, or marble) within complex participating media environments where standard surface-only BSSRDF sampling fails to account for attenuation by the surrounding medium or internal volume effects [S1]. Unlike pure volume integrators that treat the subsurface layer as a homogeneous volume [S2, unofficial], this method bridges surface geometry and participating media by treating the subsurface exit as a surface scattering event, distinct from [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) which focuses on homogeneous volume multiple scattering without explicit surface BSSRDF probes.

## How it works
**Inputs and Outputs:**
*   **Input:** Current path state (`beta`, `r_u`, `ray`, `depth`), surface intersection `isect` with an attached `BSSRDF`, scene geometry for intersection tests, sampler for random numbers, and `MediumProperties` for the current volume.
*   **Output:** Updated path contribution `L`, updated path state (`beta`, `r_u`, `ray`), and termination flags.

**Data Structures:**
*   `BSSRDFProbeSegment`: Contains entry point `p0` and exit point `p1` on the surface.
*   `WeightedReservoirSampler<T>`: A reservoir sampling structure to select a random intersection from a stream of candidates with equal weight.
*   `BSSRDFSample`: Contains the exit point interaction `ssi`, the scattering probability `Sp`, the PDF `pdf`, the outgoing direction `wo`, and the BSDF `Sw` at the exit point.
*   `MediumInteraction`: Contains position `p`, incoming direction, time, medium reference, and phase function.

**Parameters:**
*   `maxDepth`: Maximum path depth (typical value: 10-20).
*   `regularize`: Boolean flag to regularize specular BSDFs to prevent infinite throughput spikes (typical: true).
*   `lambda`: Wavelength or spectral sample index.

**Algorithm Steps:**

1.  **Retrieve BSSRDF:** Get the BSSRDF from the current intersection `isect`. If no BSSRDF exists or it is not a transmission type, skip subsurface scattering.
2.  **Sample Probe Segment:** Generate random numbers `uc` (1D) and `up` (2D). Call `bssrdf.SampleSp(uc, up)` to get a `BSSRDFProbeSegment` from entry `p0` to exit `p1`. If no segment is returned, break the subsurface loop.
3.  **Reservoir Sampling for Exit Point:**
    *   Initialize a `WeightedReservoirSampler<SubsurfaceInteraction>` with a seed derived from `MixBits(FloatToBits(sampler.Get1D()))`.
    *   Create a base interaction at `probeSeg->p0`.
    *   Loop: Spawn a ray from `base` to `probeSeg->p1`. Intersect against scene geometry.
    *   If an intersection `si` is found and `si->intr.material == isect.material`, add `SubsurfaceInteraction(si->intr)` to the reservoir with weight 1.0. Update `base` to `si->intr`.
    *   If no intersection is found, break the loop.
    *   If the reservoir has no samples, break the subsurface loop.
4.  **Convert to BSSRDFSample:** Get the sampled `SubsurfaceInteraction` (`ssi`) from the reservoir. Call `bssrdf.ProbeIntersectionToSample(ssi, scratchBuffer)` to get `bssrdfSample`. If `bssrdfSample.Sp` or `bssrdfSample.pdf` is invalid, break.
5.  **Update Path State:**
    *   Calculate PDF: `pdf = interactionSampler.SampleProbability() * bssrdfSample.pdf[0]`.
    *   Update throughput: `beta *= bssrdfSample.Sp / pdf`.
    *   Update rescaled probability: `r_u *= bssrdfSample.pdf / bssrdfSample.pdf[0]`.
    *   Set surface interaction `pi = ssi`. Set outgoing direction `pi.wo = bssrdfSample.wo`.
    *   Set `prevIntrContext = LightSampleContext(pi)`.
6.  **Regularize BSDF (Optional):** If `regularize` is true, call `Sw.Regularize()` on the BSDF from `bssrdfSample`. Set `anyNonSpecularBounces = true`.
7.  **Sample Direct Illumination:** Call `SampleLd(pi, &Sw, lambda, sampler, beta, r_u)` to add attenuated direct illumination to `L`.
8.  **Sample Indirect Subsurface Scattering:**
    *   Generate `u` (1D) and `v` (2D).
    *   Call `Sw.Sample_f(pi.wo, u, v)` to get `bs`. If no sample, break.
    *   Update throughput: `beta *= bs->f * AbsDot(bs->wi, pi.shading.n) / bs->pdf`.
    *   Update rescaled probability: `r_l = r_u / bs->pdf`.
    *   Set `specularBounce = bs->IsSpecular()`.
    *   Spawn new ray: `ray = RayDifferential(pi.SpawnRay(bs->wi))`.
9.  **Russian Roulette:** If `depth > 1`, calculate `rrBeta = beta * etaScale / r_u.Average()`. If `rrBeta.MaxComponentValue() < 1`, generate `uRR`. If `uRR < 1 - rrBeta.MaxComponentValue()`, terminate. Else, `beta /= 1 - q`.
10. **Sample Participating Medium:**
    *   Determine `tMax` from the next surface intersection or infinity.
    *   Initialize RNG with two 1D samples.
    *   Call `SampleT_maj` with a lambda function that handles medium events.
    *   Inside the lambda:
        *   Compute probabilities: `pAbsorb = sigma_a / sigma_maj`, `pScatter = sigma_s / sigma_maj`, `pNull = 1 - pAbsorb - pScatter`.
        *   Sample mode `m` using `um`.
        *   **Mode 0 (Absorption):** Set `terminated = true`, return false.
        *   **Mode 1 (Scattering):**
            *   If `depth >= maxDepth`, set `terminated = true`, return false.
            *   Update `beta` and `r_u` with transmittance and scattering PDF.
            *   If valid, add emission: `L += betap * sigma_a * Le / r_e.Average()` where `betap = beta * T_maj / pdf` and `r_e = r_u * sigma_maj * T_maj / pdf`. Note: Do not cancel `T_maj` factors before MIS weighting [S1].
            *   Sample direct lighting: `L += SampleLd(intr, nullptr, lambda, sampler, beta, r_u)`.
            *   Sample phase function: `ps = intr.phase.Sample_p(-ray.d, u)`. If valid, update `beta`, `r_l`, `ray`, and set `scattered = true`.
        *   **Mode 2 (Null Scattering):**
            *   Compute `sigma_n = ClampZero(sigma_maj - sigma_a - sigma_s)`.
            *   Update `beta` and `r_u` with null scattering PDF. If `pdf == 0`, set `beta = 0`.
            *   Return `beta && r_u`.
    *   After lambda: If `terminated` or invalid `beta`/`r_u`, return `L`. If `scattered`, continue to next path vertex. Else, update `beta`, `r_u`, `r_l` by `T_maj / T_maj[0]`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_path_tracing_with_bssrdf(isect, beta, r_u, ray, depth, max_depth, sampler, scene, lambda_idx):
    L = SampledSpectrum(0.0)
    specular_bounce = False
    any_non_specular_bounces = False
    eta_scale = 1.0
    
    # ... [BSDF Sampling for Volumetric Paths logic to get bs and update ray] ...
    
    # Account for attenuated subsurface scattering
    bssrdf = isect.get_bssrdf(ray, lambda_idx)
    if bssrdf and bs.is_transmission():
        # Sample BSSRDF probe segment
        uc = sampler.get_1d()
        up = sampler.get_2d()
        probe_seg = bssrdf.sample_sp(uc, up)
        if not probe_seg:
            return L, beta, r_u, ray, terminated
        
        # Reservoir sampling for exit point
        seed = mix_bits(float_to_bits(sampler.get_1d()))
        interaction_sampler = WeightedReservoirSampler[SubsurfaceInteraction](seed)
        base = Interaction(probe_seg.p0, ray.time, Medium())
        
        while True:
            r = base.spawn_ray_to(probe_seg.p1)
            si = scene.intersect(r, 1.0)
            if not si:
                break
            base = si.intr
            if si.intr.material == isect.material:
                interaction_sampler.add(SubsurfaceInteraction(si.intr), 1.0)
        
        if not interaction_sampler.has_sample():
            return L, beta, r_u, ray, terminated
            
        # Convert to BSSRDFSample
        ssi = interaction_sampler.get_sample()
        bssrdf_sample = bssrdf.probe_intersection_to_sample(ssi, scratch_buffer)
        if not bssrdf_sample.Sp or not bssrdf_sample.pdf:
            return L, beta, r_u, ray, terminated
            
        # Update path state
        pdf = interaction_sampler.sample_probability() * bssrdf_sample.pdf[0]
        beta *= bssrdf_sample.Sp / pdf
        r_u *= bssrdf_sample.pdf / bssrdf_sample.pdf[0]
        
        pi = ssi
        pi.wo = bssrdf_sample.wo
        prev_intr_context = LightSampleContext(pi)
        
        # Regularize BSDF
        Sw = bssrdf_sample.Sw
        any_non_specular_bounces = True
        if regularize:
            Sw.regularize()
            
        # Direct illumination
        L += sample_ld(pi, Sw, lambda_idx, sampler, beta, r_u)
        
        # Indirect subsurface scattering
        u = sampler.get_1d()
        bs = Sw.sample_f(pi.wo, u, sampler.get_2d())
        if not bs:
            return L, beta, r_u, ray, terminated
            
        beta *= bs.f * abs_dot(bs.wi, pi.shading.n) / bs.pdf
        r_l = r_u / bs.pdf
        specular_bounce = bs.is_specular()
        ray = RayDifferential(pi.spawn_ray(bs.wi))
        
    # Russian Roulette
    rr_beta = beta * eta_scale / r_u.average()
    u_rr = sampler.get_1d()
    if rr_beta.max_component_value() < 1 and depth > 1:
        q = max(0.0, 1.0 - rr_beta.max_component_value())
        if u_rr < q:
            return L, beta, r_u, ray, True
        beta /= 1.0 - q
        
    # Sample Participating Medium
    t_max = si.t_hit if si else Infinity
    hash0 = hash(sampler.get_1d())
    hash1 = hash(sampler.get_1d())
    rng = RNG(hash0, hash1)
    scattered = False
    terminated = False
    
    def medium_lambda(p, mp, sigma_maj, T_maj):
        nonlocal scattered, terminated, beta, r_u, r_l, L
        
        # Emission
        if depth < max_depth and mp.Le:
            pdf_em = sigma_maj[0] * T_maj[0]
            betap = beta * T_maj / pdf_em
            r_e = r_u * sigma_maj * T_maj / pdf_em
            if r_e:
                # MIS weight denominator is average of r_e
                L += betap * mp.sigma_a * mp.Le / r_e.average()
                
        # Sample mode
        p_absorb = mp.sigma_a[0] / sigma_maj[0]
        p_scatter = mp.sigma_s[0] / sigma_maj[0]
        p_null = max(0.0, 1.0 - p_absorb - p_scatter)
        um = rng.uniform()
        mode = sample_discrete([p_absorb, p_scatter, p_null], um)
        
        if mode == 0: # Absorption
            terminated = True
            return False
        elif mode == 1: # Scattering
            if depth >= max_depth:
                terminated = True
                return False
            pdf_sc = T_maj[0] * mp.sigma_s[0]
            beta *= T_maj * mp.sigma_s / pdf_sc
            r_u *= T_maj * mp.sigma_s / pdf_sc
            
            if beta and r_u:
                intr = MediumInteraction(p, -ray.d, ray.time, ray.medium, mp.phase)
                L += sample_ld(intr, None, lambda_idx, sampler, beta, r_u)
                
                u_phase = sampler.get_2d()
                ps = intr.phase.sample_p(-ray.d, u_phase)
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
        else: # Null Scattering
            sigma_n = clamp_zero(sigma_maj - mp.sigma_a - mp.sigma_s)
            pdf_null = T_maj[0] * sigma_n[0]
            beta *= T_maj * sigma_n / pdf_null
            if pdf_null == 0:
                beta = SampledSpectrum(0.0)
            r_u *= T_maj * sigma_n / pdf_null
            r_l *= T_maj * sigma_maj / pdf_null
            return beta and r_u
            
    sample_t_maj(ray, t_max, sampler.get_1d(), rng, lambda_idx, medium_lambda)
    
    if terminated or not beta or not r_u:
        return L, beta, r_u, ray, terminated
    if scattered:
        return L, beta, r_u, ray, False # Continue path
    else:
        beta *= T_maj / T_maj[0]
        r_u *= T_maj / T_maj[0]
        r_l *= T_maj / T_maj[0]
        return L, beta, r_u, ray, False
```

## Subtopics

### [BSDF Sampling for Volumetric Paths](1995-bsdf-sampling-for-volumetric-paths.md)
BSDF Sampling for Volumetric Paths handles the surface interaction component of path tracing within participating media, calculating indirect lighting contributions based on surface BRDFs and sampling strategies. It determines the new direction of a volumetric path upon hitting a surface, updating throughput and rescaled probabilities to account for the surface scattering event. Unlike pure volume integrators, this method bridges surface geometry and participating media by treating surface interactions as scattering events that redirect rays back into the medium or to other surfaces.

### [Reference Path Tracing](3170-reference-path-tracing.md)
Reference Path Tracing is an unbiased volumetric path tracer used as a ground-truth reference renderer for subsurface scattering and participating media. It solves the problem of accurate light transport in high-curvature surfaces and complex media by treating the subsurface layer as a volume, performing full random walks from entry to exit points. It sets itself apart from diffusion-based models by explicitly tracking geometric features for accurate shadowing and handling zero- and single-scatter events, while differing from sibling methods like BSDF Sampling for Volumetric Paths by relying on a pure random walk with Next-Event Estimation (NEE) rather than specialized BSDF sampling strategies.

### [Monte Carlo estimation of in-scattered radiance](10983-monte-carlo-estimation-of-in-scattered-radiance.md)
This method estimates the in-scattered radiance $L_{\text{ins}}(x, \boldsymbol{\omega})$ at a specific point $x$ and direction $\boldsymbol{\omega}$, accounting for light scattered into $\boldsymbol{\omega}$ from all other directions. It solves the integral term of the Radiative Transfer Equation (RTE) by sampling directions according to the phase function. It differs from direct transmission estimation by focusing on the volumetric scattering contribution, and from reference path tracing by providing a specific estimator for the in-scattering term rather than a full path weight.

### Chromatic Effects with MIS
A technique to handle non-monochromatic media efficiently by using Multiple Importance Sampling (MIS) between RGB channels during distance sampling, avoiding the inefficiency of picking a single wavelength [S2, unofficial]. The probability of picking channel $\lambda$ is $P(\lambda)$, and the probability density for distance $s$ given channel $\lambda$ is $P_\lambda(s) = \sigma_\lambda e^{-s\sigma_\lambda}$, where $\sigma_\lambda$ is the extinction coefficient for channel $\lambda$ [S2, unofficial]. The total probability density is $P(s) = P(\lambda) P_\lambda(s)$ [S2, unofficial]. The MIS weight using the balance heuristic for standard RGB renders is $W_\lambda(s) = \frac{P(\lambda)}{P(\lambda_R) + P(\lambda_G) + P(\lambda_B)}$ [S2, unofficial]. To prevent fireflies from throughput spiking in deep paths with highly tinted extinction, $P(\lambda)$ should be chosen using the current throughput instead of single scatter albedo [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per path vertex, where $N$ is the number of intersections along the BSSRDF probe segment. The reservoir sampling requires intersecting the probe segment against all geometry it passes through.
*   **Memory Complexity:** $O(1)$ additional memory for the reservoir sampler state, assuming the scene intersection structure is pre-built.
*   **Strengths:** Accurately handles subsurface scattering in complex scenes with participating media. Integrates seamlessly with existing path tracing infrastructure. Handles chromatic effects via MIS [S2, unofficial].
*   **Limitations:** Performance depends heavily on the density of geometry along the BSSRDF probe segment. Reservoir sampling can be inefficient if few valid intersections are found. Requires careful handling of MIS weights to avoid bias [S1].
*   **When to use:** When rendering translucent materials (skin, wax, marble) in scenes with fog, smoke, or other participating media.
*   **When not to use:** For purely opaque materials or purely volumetric effects without surface subsurface scattering.

## Background
This method is described in "14.2 Volume Scattering Integrators" from the PBRT book [S1]. It builds upon the SimpleVolPathIntegrator and extends it to handle BSSRDFs. The chromatic MIS variant is described in "Path Traced Subsurface Scattering Model" from Pixar [S2, unofficial].

## Sources
[S1] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S2] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
