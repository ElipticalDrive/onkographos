# Volumetric Path Tracing

Also called: Simple Volumetric Path Tracing, SimpleVolPathIntegrator, VolpathSimpleIntegrator, volpathsimple-basic, volpathsimple-drt

## Overview
Volumetric Path Tracing simulates light transport in participating media by extending classical path tracing to allow rays to enter volumetric regions, sample scattering events within the medium, and continue random walks after each scattering interaction to support multiple scattering [S1, unofficial][S4]. It solves the problem of computing ground-truth radiance in scenes with participating media (fog, smoke, clouds) by modeling color-dependent absorption and scattering, creating colored shadows [S1, unofficial]. Unlike [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md), which explicitly handles surface intersections and light emission, this method uses simple delta tracking and does not perform explicit light sampling or handle surface scattering [S4]. It differs from [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by relying on delta tracking for distance sampling rather than explicit phase function sampling for direction changes in its basic form [S4].

## How it works
**Inputs:** A camera ray $r$ (origin $o$, direction $d$), a scene with a single participating medium contained within a convex bounding volume, and infinite light sources. The medium is defined by absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and total extinction $\sigma_t = \sigma_a + \sigma_s$ (which may be spectrally varying, i.e., vectors per channel) [S1, S3, unofficial][S4]. A maximum path depth `maxDepth` [S4].

**Outputs:** The estimated radiance $L$ along the ray [S4].

**Data Structures:**
*   `Ray`: Struct with origin $o$ and normalized direction $d$ [S2, unofficial].
*   `MediumProperties`: Contains $\sigma_a$, $\sigma_s$, $\sigma_t$, and a phase function object [S4].
*   `RNG`: Random number generator initialized from hashed sampler values [S4].

**Parameters:**
*   `maxDepth`: Integer, maximum path length (e.g., 64 in Mitsuba implementation [S3, unofficial], or 10 in academic examples [S1, unofficial]).
*   `sigma_maj`: Majorant extinction coefficient, typically $\max(\sigma_t)$ over the medium [S4].

**Steps:**
1.  **Initialization:** Initialize radiance accumulator $L = 0$, path throughput $\beta = 1.0$, and depth $depth = 0$ [S4]. Terminate secondary wavelengths if spectral rendering is not supported [S4].
2.  **Loop:** While true:
    a. **Surface Intersection:** Find the first intersection $si$ of the current ray with the scene geometry (bounding volume or surfaces) [S4]. Let $t_{max}$ be the distance to this intersection (or infinity if no intersection) [S4].
    b. **Medium Sampling (Delta Tracking):** If the ray is inside a medium:
        i. Initialize a local RNG using two hashed samples from the main sampler: `hash0 = Hash(sampler.Get1D())`, `hash1 = Hash(sampler.Get1D())`, `RNG rng(hash0, hash1)` [S4].
        ii. Sample a uniform random number $u$ for distance and $u_{mode}$ for event type from the main sampler [S4].
        iii. Perform delta tracking using `SampleT_maj(ray, t_max, u, rng, lambda, lambda_function)` [S4]. The lambda function is called for each potential interaction step:
            1.  Compute medium event probabilities for the current step:
                $$p_{Absorb} = \sigma_a[0] / \sigma_{maj}[0]$$
                $$p_{Scatter} = \sigma_s[0] / \sigma_{maj}[0]$$
                $$p_{Null} = \max(0, 1 - p_{Absorb} - p_{Scatter})$$
                [S4]
            2.  Sample the event mode using discrete sampling with probabilities $\{p_{Absorb}, p_{Scatter}, p_{Null}\}$ and sample $u_{mode}$ [S4].
            3.  **If mode is 0 (Absorption):** Add emission from the medium at the current point: $L += \beta \times mp.Le$. Set `terminated = true` and return false from the lambda [S4].
            4.  **If mode is 1 (Scattering):**
                *   If `depth >= maxDepth`, set `terminated = true` and return false [S4].
                *   Increment `depth` [S4].
                *   Sample the phase function for the scattering event using two uniform samples from the RNG: $u = \{rng.Uniform<Float>(), rng.Uniform<Float>()\}$ [S4].
                *   If phase function sampling fails (`!ps`), set `terminated = true` and return false [S4].
                *   Update path throughput: $\beta = \beta \times \frac{ps.p}{ps.pdf}$ [S4].
                *   Update ray origin to intersection point $p$ and direction to scattered direction $ps.wi$. Set `scattered = true` and return false from the lambda [S4].
            5.  **If mode is 2 (Null Scattering):** Generate a new $u_{mode}$ from the RNG (`uMode = rng.Uniform<Float>()`) and return true from the lambda to continue delta tracking [S4].
    c. **Post-Sampling Check:**
        i. If `terminated` is true, return accumulated radiance $L$ [S4].
        ii. If `scattered` is true, continue the main while loop to process the new ray [S4].
    d. **Handle Unscattered Ray:** If the ray survived without scattering (no medium or no interaction):
        i. If a surface intersection $si$ exists, add emission from the surface: $L += \beta \times si.intr.Le(-ray.d, lambda)$ [S4].
        ii. If no surface intersection exists, add emission from all infinite lights: $L += \beta \times light.Le(ray, lambda)$ for each light, then return $L$ [S4].
    e. **Handle Surface Intersection:**
        i. Get the BSDF from the surface intersection [S4].
        ii. If no BSDF, skip the intersection and continue the loop [S4].
        iii. If BSDF exists and returns a valid sample, issue an error because this integrator does not support surface scattering [S4].
        iv. Otherwise, break the loop (ray terminates at surface) [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_path_tracing(ray, scene, max_depth):
    L = Spectrum(0.0)
    beta = 1.0
    depth = 0
    
    # Terminate secondary wavelengths if not supported
    lambda_sample.terminate_secondary()
    
    while True:
        si = scene.intersect(ray)
        t_max = si.t_hit if si else Infinity
        
        scattered = False
        terminated = False
        
        if ray.medium:
            # Initialize RNG
            hash0 = hash(sampler.get_1d())
            hash1 = hash(sampler.get_1d())
            rng = RNG(hash0, hash1)
            
            u = sampler.get_1d()
            u_mode = sampler.get_1d()
            
            # Delta tracking
            def delta_tracking_lambda(p, mp, sigma_maj, T_maj):
                nonlocal scattered, terminated, depth, beta, ray
                
                # Probabilities
                p_absorb = mp.sigma_a[0] / sigma_maj[0]
                p_scatter = mp.sigma_s[0] / sigma_maj[0]
                p_null = max(0.0, 1.0 - p_absorb - p_scatter)
                
                mode = sample_discrete([p_absorb, p_scatter, p_null], u_mode)
                
                if mode == 0: # Absorption
                    L += beta * mp.Le
                    terminated = True
                    return False
                elif mode == 1: # Scattering
                    if depth >= max_depth:
                        terminated = True
                        return False
                    depth += 1
                    
                    u_phase = [rng.uniform(), rng.uniform()]
                    ps = mp.phase.sample_p(-ray.d, u_phase)
                    
                    if not ps:
                        terminated = True
                        return False
                    
                    beta *= ps.p / ps.pdf
                    ray.o = p
                    ray.d = ps.wi
                    scattered = True
                    return False
                else: # Null scattering
                    u_mode = rng.uniform()
                    return True
            
            sample_t_maj(ray, t_max, u, rng, lambda_sample, delta_tracking_lambda)
        
        if terminated:
            return L
        
        if scattered:
            continue
        
        # Handle unscattered ray
        if si:
            L += beta * si.intr.Le(-ray.d, lambda_sample)
        else:
            for light in scene.infinite_lights:
                L += beta * light.Le(ray, lambda_sample)
            return L
        
        # Handle surface
        bsdf = si.intr.get_bsdf(ray, lambda_sample, camera, sampler)
        if not bsdf:
            si.intr.skip_intersection(ray, si.t_hit)
            continue
        else:
            # This integrator does not support surface scattering
            raise Error("SimpleVolPathIntegrator doesn't support surface scattering.")
```

## Subtopics

### [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md)
Multiple Scattering Volume Path Tracing with Phase Function Sampling is a path tracing technique for simulating light transport in participating media by sampling distances from an exponential distribution, scattering according to a phase function, and handling absorption via Russian roulette or weight factors. It solves the problem of computing ground-truth radiance in scenes with multiple homogeneous volumes by explicitly sampling the phase function for direction changes and supporting index-matched surfaces. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md), which uses simple delta tracking and no explicit light sampling, this method handles multiple scattering events and tracks rescaled path probabilities to avoid floating-point overflow. It differs from [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md) by focusing on phase function sampling for direction rather than transmittance sampling for distance.

### [Volumetric Path Tracing with Spectral Integration](2188-volumetric-path-tracing-with-spectral-integration.md)
A path tracing technique that handles ray marching through participating media (homogeneous volumes) with absorption and scattering, using spectral wavelength sampling for dispersion effects like rainbow refraction. It differs from standard RGB path tracing by performing spectral Multiple Importance Sampling (MIS) for channel selection based on throughput, rather than sampling in RGB space directly.

### [Volumetric Path Tracing Estimator Derivation](3830-volumetric-path-tracing-estimator-derivation.md)
This article derives the simplified Monte Carlo estimator for the generalized path contribution function $\hat{P}$ in volumetric path tracing. It solves the problem of expressing the radiance contribution of a sampled path in terms of the specific sampling probabilities used during generation (distance, event type, and direction). It sets itself apart from its siblings by providing the foundational algebraic simplification that reduces the raw path integral estimator to a product of phase function ratios and emission, assuming delta tracking with a single majorant. This derivation underpins the basic volumetric path tracer described in [Volumetric Path Tracing](1863-volumetric-path-tracing.md).

### [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md)
Volumetric Path Tracing with Transmittance Sampling is a unidirectional path tracing method that computes single-scattering radiance in participating media by sampling the distance to the next interaction event directly from the transmittance distribution. It differs from its parent, [Volumetric Path Tracing](1863-volumetric-path-tracing.md), by explicitly handling surface intersections and light emission rather than relying solely on delta tracking for volume scattering. Unlike [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md), this method does not handle multiple scattering events within the volume.

### [Unbiased Monte Carlo Estimator for Radiance and Radiance Derivatives](10971-unbiased-monte-carlo-estimator-for-radiance-and-radiance-der.md)
This method provides an unbiased Monte Carlo estimator for the first-order radiance derivative $\hat{L}^{(1)}$ and the radiance $L^{(1)}$ in participating media, handling volumetric and interfacial scattering events separately based on the free-flight distance relative to the boundary distance $D$. It distinguishes itself from standard volumetric path tracing by explicitly propagating scene derivatives alongside radiance values, enabling the computation of gradients required for differentiable rendering. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md), which computes only radiance, this estimator jointly tracks the sensitivity of the solution to scene parameters.

### Differentiable Volumetric Path Tracing
An unbiased Monte Carlo method analogous to volumetric path tracing (VPT) used to numerically estimate derivatives of the radiative transfer equation (RTE) with respect to arbitrary scene parameterizations [S6, unofficial]. It is used in imaging-inspired inverse rendering examples and design-inspired inverse rendering examples to estimate gradients for optimizing scene parameters [S6, unofficial]. For ideal specular surfaces, discontinuities of radiance $L(x, \omega)$ in $\omega$ must include mirror images of geometric edges, which can be nontrivial to identify [S6, unofficial]. The derivations assume exponential transmittance; supporting media with non-exponential transmittance requires adjustment of the derivations [S6, unofficial].

## Operating characteristics
**Time and Memory Complexity:** The method is $O(N)$ per path vertex for delta tracking steps, where $N$ is the number of steps in the delta tracking loop. The total complexity depends on the optical thickness of the medium and the maximum path depth. Memory usage is low, primarily storing the ray state, throughput, and local RNG state [S4].

**Measured Performance:** In academic implementations, 256 samples per pixel give a reasonably accurate rendering of volumetric models, though variance spikes may occur due to the lack of explicit light sampling [S4]. The Mitsuba 3 implementation with Differential Ratio Tracking (DRT) can reduce runtime from $O(n^2)$ to $O(n)$ by subsampling the DRT method [S3, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple implementation, suitable for debugging and computing ground-truth results for more sophisticated algorithms [S4]. Handles multiple scattering within the volume [S1, unofficial].
*   **Limitations:** High variance compared to integrators that sample lights directly (e.g., VolPathIntegrator has 1,288 times lower MSE in comparable scenes) [S4]. Does not support surface scattering or delta distribution light sources [S4]. Assumes a single medium in a convex bounding volume with a null BSDF at the boundary [S3, unofficial]. Only supports infinite light sources [S3, unofficial].

**When to use:** When debugging more sophisticated volumetric algorithms or when a simple, ground-truth baseline is needed for scenes with participating media and no complex surface interactions [S4].

**When not to use:** When high efficiency is required, or when the scene contains surfaces with BSDFs or discrete light sources [S4].

## Background
Volumetric Path Tracing extends classical path tracing to participating media. The `SimpleVolPathIntegrator` is described in "14.2 Volume Scattering Integrators" of the pbr-book [S4]. It follows the sampling approach described in Section 14.1.2 of the same source. The method is implemented as a Mitsuba 3 integrator plugin in the `VolpathSimpleIntegrator` class, supporting both baseline free-flight sampling and Differential Ratio Tracking (DRT) [S3, unofficial]. Academic implementations have explored colored volumetric shadows and homogeneous/heterogeneous media [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] MajorMeerkatThe3rd/VolumetricPathTracing, https://github.com/MajorMeerkatThe3rd/VolumetricPathTracing, fetched 2026-10-26 (unofficial)
[S3] https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
[S4] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S5] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
[S6] PDF A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
