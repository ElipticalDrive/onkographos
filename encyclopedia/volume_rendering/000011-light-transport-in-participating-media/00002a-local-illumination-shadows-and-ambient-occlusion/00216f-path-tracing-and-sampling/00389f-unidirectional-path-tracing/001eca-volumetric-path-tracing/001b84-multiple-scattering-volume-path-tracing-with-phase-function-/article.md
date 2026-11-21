# Multiple Scattering Volume Path Tracing with Phase Function Sampling

Also called: VolPathIntegrator, Volumetric Path Integrator

## Overview
A path tracing technique for simulating light transport in participating media where a ray enters a material, applies a BTDF, travels a distance determined by an exponential distribution, scatters according to a phase function, and repeats until exiting the material with another BTDF [S1, unofficial][S2, S3]. It solves the problem of rendering multiple scattering events in homogeneous volumes by importance-sampling the distance to the next interaction and the scattering direction, supporting index-matched surfaces and Russian roulette termination [S2]. Unlike its parent [Volumetric Path Tracing](1863-volumetric-path-tracing.md), which uses simple delta tracking, this method explicitly samples the phase function to handle multiple bounces efficiently [S2]. It differs from [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md) by using majorant-based transmittance sampling to handle inhomogeneous or complex media boundaries robustly [S3].

## How it works
**Inputs:** Camera ray, scene geometry, participating media definitions (scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function), light sources, maximum depth `maxDepth`, Russian roulette depth `rr_depth`, and a sampler for random numbers.
**Outputs:** Radiance $L$ for the pixel.

**Data Structures:**
*   `Ray`: Origin `o`, direction `d`, time `time`, medium pointer.
*   `MediumProperties`: $\sigma_a$ (absorption), $\sigma_s$ (scattering), $\sigma_t = \sigma_a + \sigma_s$ (extinction), phase function, emission $L_e$.
*   `SampledSpectrum`: Represents spectral data (e.g., RGB or sampled wavelengths).
*   `PhaseFunctionSample`: Contains direction `wi`, PDF `pdf`, and phase function value `p`.
*   `BSDFSample`: Contains direction `wi`, PDF `pdf`, BSDF value `f`, and flags (specular, transmission).

**Parameters:**
*   `maxDepth`: Integer limiting the maximum number of bounces/scattering events. If -1, infinite bounces are allowed (though Russian roulette terminates them) [S2, S3]. `maxDepth=2` corresponds to single scattering [S2].
*   `rr_depth`: Controls when Russian roulette begins. RR starts when a light path has `rr_depth + 1` vertices [S2].
*   `regularize`: Boolean flag; if true, BSDFs are regularized after non-specular bounces to prevent numerical instability [S3].
*   `lightSampleStrategy`: String specifying the light sampling strategy (default "bvh") [S3].

**Steps:**

1.  **Initialization:**
    Initialize state variables:
    *   $L = 0$ (Radiance)
    *   $\beta = 1$ (Path throughput weight)
    *   $r_u = 1$ (Rescaled unidirectional probability)
    *   $r_l = 1$ (Rescaled light-sampling probability)
    *   `specularBounce` = false
    *   `anyNonSpecularBounces` = false
    *   `depth` = 0
    *   `etaScale` = 1 (Scalar multiplier for transmission refractive index)
    *   `prevIntrContext` = LightSampleContext (empty) [S3]

2.  **Main Loop:**
    While true:
    
    a. **Intersection:**
    Intersect the current ray with the scene geometry to get intersection `si`. Let $t_{max} = si.t_{hit}$ if an intersection exists, otherwise $t_{max} = \infty$ [S3].

    b. **Volumetric Sampling (if in medium):**
    If the ray is inside a medium:
    i.  **RNG Initialization:** Generate two 1D samples from the sampler, hash them to create `hash0` and `hash1`, and initialize a local RNG `rng(hash0, hash1)` [S3].
    ii. **Majorant Sampling:** Sample the majorant transmittance $T_{maj}$ using `SampleT_maj`. This involves a callback that receives point $p$, medium properties $mp$, majorant cross-section $\sigma_{maj}$, and transmittance $T_{maj}$ [S3].
    iii. **Medium Emission:** If `depth < maxDepth` and the medium emits light ($mp.L_e$):
        *   Compute PDF: $pdf = \sigma_{maj}[0] \cdot T_{maj}[0]$ [S3].
        *   Compute $\beta' = \beta \cdot T_{maj} / pdf$ [S3].
        *   Compute rescaled absorption probability: $r_e = r_u \cdot \sigma_{maj} \cdot T_{maj} / pdf$ [S3].
        *   If $r_e$ is non-zero, add to radiance: $L += \beta' \cdot mp.\sigma_a \cdot mp.L_e / r_e.Average()$ [S3].
    iv. **Event Probabilities:**
        Compute probabilities for absorption, scattering, and null scattering:
        $$p_{Absorb} = \frac{\sigma_a[0]}{\sigma_{maj}[0]}$$
        $$p_{Scatter} = \frac{\sigma_s[0]}{\sigma_{maj}[0]}$$
        $$p_{Null} = \max(0, 1 - p_{Absorb} - p_{Scatter})$$ [S3]
    v.  **Sample Event Type:**
        Draw uniform random number $u_m$. Determine mode:
        $$mode = \text{SampleDiscrete}(\{p_{Absorb}, p_{Scatter}, p_{Null}\}, u_m)$$ [S3]
    vi. **Handle Event:**
        *   **Absorption (mode == 0):** Set `terminated = true` and return `false` from the callback to end the path [S3].
        *   **Scattering (mode == 1):**
            *   If `depth >= maxDepth`, set `terminated = true` and return `false` [S3].
            *   Increment `depth` [S3].
            *   Compute PDF: $pdf = T_{maj}[0] \cdot mp.\sigma_s[0]$ [S3].
            *   Update $\beta$ and $r_u$:
                $$\beta \leftarrow \beta \cdot \frac{T_{maj} \cdot mp.\sigma_s}{pdf}$$
                $$r_u \leftarrow r_u \cdot \frac{T_{maj} \cdot mp.\sigma_s}{pdf}$$ [S3]
            *   If $\beta$ and $r_u$ are non-zero:
                *   Create `MediumInteraction` `intr` at point $p$ with direction $-ray.d$ [S3].
                *   Sample direct lighting: $L += \text{SampleLd}(intr, \text{nullptr}, \lambda, \text{sampler}, \beta, r_u)$ [S3].
                *   Sample phase function: Generate 2D sample $u$. Call $ps = intr.phase.Sample_p(-ray.d, u)$ [S3].
                *   If $ps$ is null or $ps.pdf == 0$, set `terminated = true` [S3].
                *   Else:
                    *   Update $\beta$ and $r_l$:
                        $$\beta \leftarrow \beta \cdot \frac{ps.p}{ps.pdf}$$
                        $$r_l \leftarrow \frac{r_u}{ps.pdf}$$ [S3]
                    *   Store `prevIntrContext = LightSampleContext(intr)` [S3].
                    *   Set `scattered = true` [S3].
                    *   Update ray: $ray.o = p$, $ray.d = ps.wi$ [S3].
                    *   Set `specularBounce = false`, `anyNonSpecularBounces = true` [S3].
            *   Return `false` from callback [S3].
        *   **Null Scattering (mode == 2):**
            *   Compute null scattering coefficient: $\sigma_n = \text{ClampZero}(\sigma_{maj} - mp.\sigma_a - mp.\sigma_s)$ [S3].
            *   Compute PDF: $pdf = T_{maj}[0] \cdot \sigma_n[0]$ [S3].
            *   Update $\beta$:
                $$\beta \leftarrow \beta \cdot \frac{T_{maj} \cdot \sigma_n}{pdf}$$
                If $pdf == 0$, set $\beta = 0$ [S3].
            *   Update $r_u$ and $r_l$:
                $$r_u \leftarrow r_u \cdot \frac{T_{maj} \cdot \sigma_n}{pdf}$$
                $$r_l \leftarrow r_l \cdot \frac{T_{maj} \cdot \sigma_{maj}}{pdf}$$ [S3]
            *   Return `beta && r_u` (boolean result of validity) [S3].

    c. **Post-Medium Handling:**
    If `terminated` or $\beta=0$ or $r_u=0$, return $L$ [S3].
    If `scattered`, `continue` to next iteration of the main loop [S3].
    If unscattered (ray passed through medium without interaction):
    $$\beta \leftarrow \beta \cdot \frac{T_{maj}}{T_{maj}[0]}$$
    $$r_u \leftarrow r_u \cdot \frac{T_{maj}}{T_{maj}[0]}$$
    $$r_l \leftarrow r_l \cdot \frac{T_{maj}}{T_{maj}[0]}$$ [S3]

    d. **Surface Intersection Handling:**
    If no intersection (`!si`):
    *   For each infinite light source:
        *   If light emits ($L_e$):
            *   If `depth == 0` or `specularBounce`:
                $$L += \beta \cdot L_e / r_u.Average()$$ [S3]
            *   Else (MIS):
                *   Compute light PDF: $lightPDF = lightSampler.PMF(prevIntrContext, light) \cdot light.PDF_{Li}(prevIntrContext, ray.d, true)$ [S3].
                *   Update $r_l$: $r_l \leftarrow r_l \cdot lightPDF$ [S3].
                *   $$L += \beta \cdot L_e / (r_u + r_l).Average()$$ [S3]
    If intersection exists (`si`):
    *   Get BSDF: $bsdf = isect.GetBSDF(ray, \lambda, camera, scratchBuffer, sampler)$ [S3].
    *   If `!bsdf`, skip intersection and `continue` [S3].
    *   **Visible Surface Initialization:** If `depth == 0` and `visibleSurf` is set:
        *   Define 16 sample arrays `ucRho` and `uRho` (values hardcoded in [S3]).
        *   Compute albedo: $albedo = bsdf.rho(isect.wo, ucRho, uRho)$ [S3].
        *   Set `*visibleSurf = VisibleSurface(isect, albedo, \lambda)` [S3].
    *   **Depth Check:** If `depth >= maxDepth`, return $L$ [S3]. Increment `depth` [S3].
    *   **Regularization:** If `regularize` and `anyNonSpecularBounces`, call `bsdf.Regularize()` [S3].
    *   **Direct Lighting:** If BSDF is non-specular:
        $$L += \text{SampleLd}(isect, &bsdf, \lambda, \text{sampler}, \beta, r_u)$$ [S3]
        Update `prevIntrContext = LightSampleContext(isect)` [S3].
    *   **BSDF Sampling:**
        *   Get outgoing direction $wo = isect.wo$ [S3].
        *   Generate 1D sample $u$ and 2D sample [S3].
        *   Sample BSDF: $bs = bsdf.Sample_f(wo, u, sampler.Get2D())$ [S3].
        *   If `!bs`, break loop [S3].
        *   Update $\beta$ and $r_l$:
            $$\beta \leftarrow \beta \cdot \frac{bs.f \cdot |n \cdot wi|}{bs.pdf}$$ [S3]
            If `bs.pdfIsProportional`:
                $$r_l = \frac{r_u}{bsdf.PDF(wo, bs.wi)}$$
            Else:
                $$r_l = \frac{r_u}{bs.pdf}$$ [S3]
        *   Update state:
            *   `specularBounce = bs.IsSpecular()` [S3].
            *   `anyNonSpecularBounces |= !bs.IsSpecular()` [S3].
            *   If transmission: `etaScale *= Sqr(bs.eta)` [S3].
            *   Spawn new ray: $ray = isect.SpawnRay(ray, bsdf, bs.wi, bs.flags, bs.eta)$ [S3].
    *   **BSSRDF Handling:** If BSSRDF exists and BSDF is transmission:
        *   Sample probe segment: $probeSeg = bssrdf.SampleSp(uc, up)$ [S3].
        *   If `!probeSeg`, break [S3].
        *   Initialize `interactionSampler` with seed [S3].
        *   Intersect ray from $p_0$ to $p_1$. If intersection on same material, add to sampler with weight 1 [S3].
        *   If no sample, break [S3].
        *   Get subsurface interaction $ssi$ and sample $bssrdfSample$ [S3].
        *   Update $\beta$ and $r_u$ using $bssrdfSample.Sp$ and PDF [S3].
        *   Sample BSDF at exit point $pi$: $bs = Sw.Sample_f(pi.wo, u, sampler.Get2D())$ [S3].
        *   If `!bs`, break [S3].
        *   Update $\beta$ and $r_l$:
            $$\beta \leftarrow \beta \cdot \frac{bs.f \cdot |n \cdot wi|}{bs.pdf}$$
            $$r_l = \frac{r_u}{bs.pdf}$$ [S3]
        *   Set `specularBounce`, spawn new ray from $pi$ [S3].

    e. **Russian Roulette:**
    *   Compute $rrBeta = \beta \cdot etaScale / r_u.Average()$ [S3].
    *   Generate $u_{RR}$ [S3].
    *   If $rrBeta.MaxComponentValue() < 1$ and `depth > 1`:
        *   Compute $q = \max(0, 1 - rrBeta.MaxComponentValue())$ [S3].
        *   If $u_{RR} < q$, break [S3].
        *   Else, $\beta /= 1 - q$ [S3].

3.  **Return:** Return $L$ [S3].

**Formulas:**
*   **Extinction:** $\sigma_t = \sigma_s + \sigma_a$ [S1, unofficial].
*   **Rescaled Probabilities:**
    $$r_{u, \lambda, i}(\bar{p}_n) = \frac{p_{u, \lambda, i}(\bar{p}_n)}{p_{path}(\bar{p}_n)}$$
    $$r_{l, \lambda, i}(\bar{p}_n) = \frac{p_{l, \lambda, i}(\bar{p}_n)}{p_{path}(\bar{p}_n)}$$ [S3]
*   **MIS Weight (Unidirectional):**
    $$w_u(\bar{p}_n) = \frac{1}{\frac{1}{m} \sum_{i=1}^m r_{u, \lambda, i}(\bar{p}_n) + \sum_{i=1}^m r_{l, \lambda, i}(\bar{p}_n)}$$ [S3]
*   **Russian Roulette Probability:**
    $$P_{rr} = \min \left( \frac{\text{contrib}(\text{path})}{p(\text{path})}, 0.95 \right)$$ [S2]

**Edge Cases and Pitfalls:**
*   **Double Counting:** Do not apply Beer-Lambert attenuation $e^{-\sigma x}$ explicitly to path weights if the distance was importance-sampled from the exponential distribution, as it is implicitly incorporated in the sampling probability [S1, unofficial][S3].
*   **Zero PDF:** If phase function sampling fails or returns zero PDF, terminate the path [S3]. If null scattering PDF is zero, explicitly set $\beta=0$ to prevent division by zero [S3].
*   **Chromatic Media:** For non-monochromatic coefficients, simulating all channels simultaneously is inefficient. Assign each ray a single wavelength or color channel and calculate scattering separately [S1, unofficial].
*   **Index-Matched Surfaces:** Surfaces with `material_id == -1` are index-matched. Pass through them, update the medium using `update_medium`, and increment bounces [S2].

## Subtopics

### [Next Event Estimation for Volumetric Path Tracing](1125-next-event-estimation-for-volumetric-path-tracing.md)
Adds direct lighting estimation by sampling points on light sources and tracing shadow rays. Solves inefficiency with small/distant lights.

### [Volumetric Path Tracing with BSSRDF and Medium Sampling](1867-volumetric-path-tracing-with-bssrdf-and-medium-sampling.md)
Extends to subsurface scattering by integrating BSSRDF probes into the path traversal loop. Handles translucent materials within media.

## Operating characteristics
*   **Complexity:** Time complexity depends on the number of scattering events, which is driven by the optical thickness of the medium. Memory complexity is linear in path depth.
*   **Performance:** Robust for multiple scattering in homogeneous and inhomogeneous media. Majorant sampling handles complex boundaries but may be less efficient if the majorant is much larger than the actual extinction coefficient [S3].
*   **Strengths:** Handles multiple scattering, index-matched surfaces, and chromatic media. Rescaled probabilities prevent floating-point overflow/underflow [S3].
*   **Limitations:** Can be slow for very dense media due to many scattering events. Russian roulette introduces variance [S2].
*   **When to use:** Ground truth rendering, scenes with complex volumetric interactions, translucent materials.
*   **When not to use:** Real-time rendering requiring approximations, very sparse media where simpler methods suffice.

## Background
Originated from the need to simulate light transport in participating media accurately. The VolPathIntegrator is a key component of modern path tracers like PBRT, handling both surface and volume transport robustly [S3]. It builds on earlier work in Monte Carlo volumetric scattering [S1, unofficial] and extends basic volumetric path tracing [S2] by incorporating rescaled probabilities and MIS for unidirectional and light sampling strategies [S3].

## Sources
[S1] Full Monte-Carlo Volumetric Scattering, https://computergraphics.stackexchange.com/questions/2563/full-monte-carlo-volumetric-scattering, fetched 2026-10-26 (unofficial)
[S2] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
