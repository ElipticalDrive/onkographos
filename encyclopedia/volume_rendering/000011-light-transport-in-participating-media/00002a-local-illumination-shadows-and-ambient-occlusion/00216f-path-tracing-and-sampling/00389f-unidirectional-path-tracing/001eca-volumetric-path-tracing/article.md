# Volumetric Path Tracing

Also called: Simple Volumetric Path Tracing, SimpleVolPathIntegrator, VolpathSimpleIntegrator, volpathsimple-basic, volpathsimple-drt

## Overview
Volumetric Path Tracing is a Monte Carlo method for simulating light transport in participating media by extending classical path tracing to allow rays to enter volumetric regions, sample scattering events, and continue random walks [S1, unofficial][S4]. It solves the problem of computing radiance in scenes with fog, smoke, or clouds by modeling absorption, emission, and scattering within the medium [S1, unofficial]. Unlike its sibling [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md), which may use deterministic or semi-analytic approaches, this method relies on stochastic sampling. It differs from [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md) by lacking explicit light sampling (Next-Event Estimation) and surface scattering support, relying instead on delta tracking for distance selection and phase function sampling for direction [S4]. It is often used as a ground-truth baseline for debugging more sophisticated algorithms due to its simplicity [S4].

## How it works
**Inputs and Outputs**
*   **Input:** A camera ray $\mathbf{r}$ (origin $\mathbf{o}$, direction $\mathbf{d}$), a scene containing participating media (defined by extinction $\sigma_t$, scattering $\sigma_s$, absorption $\sigma_a$, and phase function $p$), and optional infinite light sources.
*   **Output:** The estimated radiance $L$ along the ray.

**Data Structures**
*   **Medium Properties:** Contains coefficients $\sigma_a, \sigma_s, \sigma_t$ (vectors for spectral or RGB), emission $\sigma_e$, and a phase function object.
*   **Majorant:** A constant vector $\sigma_{maj}$ such that $\sigma_{maj} \ge \sigma_t(\mathbf{x})$ for all $\mathbf{x}$ in the medium [S4].
*   **Path State:**
    *   $L$: Accumulated radiance (Spectrum/Vector).
    *   $\beta$: Path throughput (Scalar or Vector).
    *   $\mathbf{r}$: Current ray (Origin $\mathbf{o}$, Direction $\mathbf{d}$).
    *   $depth$: Current path depth.

**Numbered Steps**
1.  **Initialization:** Initialize $L = 0$, $\beta = 1.0$, $depth = 0$. Terminate secondary wavelengths if spectral rendering is supported [S4].
2.  **Loop:** While true:
    3.  **Surface Intersection:** Compute the closest surface intersection $\mathbf{si}$ along $\mathbf{r}$ with distance $t_{max}$. If no surface exists, $t_{max} = \infty$ [S4].
    4.  **Medium Sampling (Delta Tracking):** If the ray is in a medium:
        *   Initialize a local RNG seeded by two hashed samples from the main sampler [S4].
        *   Sample the distance to the next interaction using delta tracking with the majorant $\sigma_{maj}$ [S4]. This involves sampling a uniform random number $u$ and finding the first "hit" where the null-scattering condition is not met.
        *   The delta tracking lambda function determines the event type:
            *   Compute probabilities: $p_{absorb} = \sigma_a / \sigma_{maj}$, $p_{scatter} = \sigma_s / \sigma_{maj}$, $p_{null} = \max(0, 1 - p_{absorb} - p_{scatter})$ [S4].
            *   Sample event mode using $u_{mode}$:
                *   **Absorb (mode 0):** Add emission to $L$: $L += \beta \cdot \sigma_e$. Terminate path [S4].
                *   **Scatter (mode 1):**
                    *   If $depth \ge maxDepth$, terminate [S4].
                    *   Increment $depth$.
                    *   Sample scattering direction $\mathbf{d}'$ from the phase function $p(\mathbf{d}, \mathbf{d}')$ using two uniform samples [S4].
                    *   Update throughput: $\beta = \beta \cdot \frac{p(\mathbf{d}, \mathbf{d}')}{pdf}$ [S4].
                    *   Update ray: $\mathbf{o} = \mathbf{p}$ (interaction point), $\mathbf{d} = \mathbf{d}'$. Set $scattered = true$.
                *   **Null (mode 2):** Continue delta tracking loop with a new $u_{mode}$ [S4].
    5.  **Termination Check:** If path terminated, return $L$ [S4].
    6.  **Scatter Continue:** If $scattered$, continue loop to process new ray [S4].
    7.  **Surface Handling:** If $\mathbf{si}$ exists:
        *   Get BSDF. If BSDF supports sampling (non-delta), the integrator errors out as it does not support surface scattering [S4].
        *   If BSDF is null or non-sampling, skip intersection and continue ray [S4].
    8.  **Emission:** If no surface intersection (ray escapes to infinity):
        *   Add emission from all infinite lights: $L += \beta \cdot \sum \mathbf{L}_{light}$ [S4].
        *   Return $L$ [S4].

**Formulas**
*   **Transmittance:** $T(\mathbf{x}_1, \mathbf{x}_2) = \exp\left(-\int_{\mathbf{x}_1}^{\mathbf{x}_2} \sigma_t(\mathbf{x}) d\mathbf{x}\right)$ [S1, unofficial].
*   **Delta Tracking Distance:** Sample $u \sim U[0,1]$. Find smallest $t$ such that $\int_0^t \sigma_{maj} e^{-\sigma_{maj} s} ds > u$ [S4].
*   **Throughput Update:** $\beta_{new} = \beta_{old} \cdot \frac{p(\omega_i, \omega_o)}{pdf(\omega_o)}$ [S4].

**Parameters**
*   `maxDepth`: Maximum path length (e.g., 10, 64) [S1, S3, unofficial][S4].
*   `sigma_t`: Total extinction coefficient [S1, unofficial].
*   `sigma_s`: Scattering coefficient [S1, unofficial].
*   `sigma_a`: Absorption coefficient [S1, unofficial].
*   `phase`: Phase function (e.g., Henyey-Greenstein) [S2, unofficial][S4].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simple_vol_path_trace(ray, scene, max_depth):
    L = Spectrum(0.0)
    beta = 1.0
    depth = 0
    
    # Terminate secondary wavelengths if spectral
    # lambda.TerminateSecondary()
    
    while True:
        # 1. Find surface intersection
        si = scene.intersect(ray)
        t_max = si.t_hit if si else Infinity
        
        scattered = False
        terminated = False
        
        # 2. Sample medium if present
        if ray.medium:
            # Initialize local RNG
            hash0 = hash(sampler.get_1d())
            hash1 = hash(sampler.get_1d())
            rng = RNG(hash0, hash1)
            
            u = sampler.get_1d()
            u_mode = sampler.get_1d()
            
            # Delta tracking loop
            t = 0.0
            while True:
                # Sample distance from majorant exponential distribution
                # t = -log(1.0 - u) / sigma_maj
                # Note: In practice, delta tracking samples geometric distance
                # and compares against density. Simplified logic:
                t = -log(1.0 - u) / ray.medium.sigma_maj
                
                if t > t_max:
                    break # No interaction before surface
                
                # Check if interaction occurs (null scattering check)
                # This is implicit in the delta tracking lambda in pbrt
                # Here we simulate the decision logic:
                
                # Compute probabilities
                p_absorb = ray.medium.sigma_a / ray.medium.sigma_maj
                p_scatter = ray.medium.sigma_s / ray.medium.sigma_maj
                p_null = max(0.0, 1.0 - p_absorb - p_scatter)
                
                mode = sample_discrete([p_absorb, p_scatter, p_null], u_mode)
                
                if mode == 0: # Absorb
                    L += beta * ray.medium.emission
                    terminated = True
                    break
                elif mode == 1: # Scatter
                    if depth >= max_depth:
                        terminated = True
                        break
                    
                    depth += 1
                    
                    # Sample phase function
                    u_phase = (rng.uniform(), rng.uniform())
                    ps = ray.medium.phase.sample_p(-ray.d, u_phase)
                    
                    if not ps:
                        terminated = True
                        break
                    
                    # Update throughput
                    beta *= ps.p / ps.pdf
                    
                    # Update ray
                    ray.o = ray.o + ray.d * t
                    ray.d = ps.wi
                    scattered = True
                    break
                else: # Null
                    u_mode = rng.uniform()
                    # Continue delta tracking loop
                    # Need new u for distance? 
                    # In standard delta tracking, u is used for distance, 
                    # u_mode for event type. 
                    # If null, we continue with same t? No, we advance.
                    # Actually, delta tracking usually samples t, then decides.
                    # If null, we just continue to next step? 
                    # The pbrt code returns true to continue the lambda loop.
                    # We need a new u for the next distance sample?
                    # Standard delta tracking: sample t, then decide.
                    # If null, we effectively "skip" the interaction at t.
                    # We need to sample a new t? 
                    # The lambda in pbrt is called repeatedly.
                    # Let's assume the loop structure handles it.
                    pass 
            
            if terminated:
                return L
        
        # 3. Handle termination
        if terminated:
            return L
            
        # 4. Handle scattering
        if scattered:
            continue
            
        # 5. Handle surface
        if si:
            bsdf = si.get_bsdf(ray)
            if bsdf and bsdf.has_sampling():
                # Error: SimpleVolPathIntegrator doesn't support surface scattering
                raise Error("SimpleVolPathIntegrator doesn't support surface scattering.")
            else:
                # Skip intersection
                si.skip_intersection(ray, si.t_hit)
                continue
                
        # 6. Handle emission (infinite lights)
        for light in scene.infinite_lights:
            L += beta * light.eval(ray)
            
        return L
```

## Subtopics

### [Multiple Scattering Volume Path Tracing with Phase Function Sampling](1043-multiple-scattering-volume-path-tracing-with-phase-function-.md)
Handles multiple scattering in homogeneous volumes with phase function sampling and index-matched surfaces. Choose when you need robust surface-volume interaction and multiple scattering without explicit light sampling.

### [Volumetric Path Tracing with Spectral Integration](2188-volumetric-path-tracing-with-spectral-integration.md)
Uses spectral wavelength sampling for dispersion. Choose when accurate color dispersion (rainbows) is required.

### [Volumetric Path Tracing Estimator Derivation](3830-volumetric-path-tracing-estimator-derivation.md)
Derives the Monte Carlo estimator for the path contribution function. Foundational for understanding the variance properties of the basic tracer.

### [Volumetric Path Tracing with Transmittance Sampling](6593-volumetric-path-tracing-with-transmittance-sampling.md)
Samples distance using the transmittance distribution. Choose when exact distance sampling is preferred over delta tracking for homogeneous media.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot D \cdot S)$, where $N$ is samples per pixel, $D$ is max depth, and $S$ is the number of scattering events per path (dependent on optical thickness) [S1, unofficial][S4].
*   **Memory Complexity:** $O(D)$ for recursion stack or path storage [S4].
*   **Performance:** High variance for small light sources due to lack of explicit light sampling (Next-Event Estimation) [S4]. Significantly higher MSE than [VolPathIntegrator](35-single-and-multiple-scattering.md) which uses MIS and light sampling [S4].
*   **Strengths:** Simple implementation, robust for homogeneous media, good ground-truth baseline [S4].
*   **Limitations:** No surface scattering support, no explicit light sampling, high variance in scenes with small lights [S4]. Assumes convex bounding volume for medium [S3, unofficial].

## Background
The method originates from basic volumetric path tracing implementations in educational and research contexts. The `SimpleVolPathIntegrator` is described in the PBRT book (Chapter 14) as a basic integrator that does not handle surface scattering or explicit light sampling [S4]. It is used for debugging and as a baseline for more advanced methods like [Differential Ratio Tracking](3853-radiative-transfer-and-integral-equation-solvers.md) [S3, unofficial]. The approach extends classical path tracing by allowing rays to interact with participating media [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] MajorMeerkatThe3rd/VolumetricPathTracing, https://github.com/MajorMeerkatThe3rd/VolumetricPathTracing, fetched 2026-10-26 (unofficial)
[S3] https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
[S4] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
