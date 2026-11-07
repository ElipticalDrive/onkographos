# Multiple Importance Sampling for Volumetric Path Tracing

Also called: Volumetric MIS, Multiple Importance Sampling (MIS) for Volumometric Path Tracing

## Overview
Multiple Importance Sampling (MIS) for Volumetric Path Tracing combines contributions from phase function sampling and next event estimation (NEE) using the power heuristic to reduce variance, particularly in scenes with index-matching surfaces or chromatic media [S1, S2]. Unlike standard path tracing which may suffer from high variance when skipping index-matching surfaces without sampling new directions, this method caches PDFs and transmittance across bounces to correctly weight the two primary sampling strategies [S1, S2]. It builds on [Single and multiple scattering](35-single-and-multiple-scattering.md) by providing a consistent estimator for direct lighting in participating media.

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$, direction $\boldsymbol{\omega}$, scene geometry and media, light sources, random number generator.
*   **Output:** Estimated radiance $L$ at the camera.

**Data Structures**
The integrator maintains cache variables during the main path loop to handle index-matching surfaces:
*   `dir_pdf`: Float (or Spectrum). The PDF of the latest phase function sampling in solid angle measure. Initialized to 0.
*   `nee_p_cache`: Point. The last position $\mathbf{p}$ that can issue NEE. `None` if the current position is on an index-matching surface. Initialized to `None`.
*   `multi_trans_pdf`: Float (or Spectrum). The product PDF of transmittance sampling through index-matching surfaces from the last phase function sampling. Initialized to 1.
*   `current_path_throughput`: Spectrum. Accumulated throughput from camera to current vertex.
*   `r_p`: Float (or Spectrum). Rescaled path probability for unidirectional sampling up to the current vertex.
*   `r_l`: Float (or Spectrum). Rescaled path probability for light sampling.
*   `r_u`: Float (or Spectrum). Rescaled path probability for unidirectional (scattering function) sampling.

**Numbered Steps**

1.  **Initialization**:
    Set `radiance = 0`, `current_path_throughput = 1`, `bounces = 0`, `dir_pdf = 0`, `nee_p_cache = None`, `multi_trans_pdf = 1`.
    [G] Initialize `never_scatter = true` to handle the first bounce correctly.

2.  **Main Loop**:
    While the path is alive:
    a.  **Distance Sampling**: Sample distance $t$ to next interaction using delta tracking or homogeneous sampling. Update `current_path_throughput` by multiplying with transmittance $T(t)$ and dividing by the average PDF of the sampling decision (ratio tracking) [S2].
    b.  **Interaction**: Determine if the interaction is a volume scattering event or a surface hit.
    c.  **Update Caches**:
        *   If a phase function was sampled (direction $\boldsymbol{\omega}'$ selected), update `dir_pdf` with the phase function PDF $p(\boldsymbol{\omega}_{in}, \boldsymbol{\omega}')$.
        *   If the current position is on a non-index-matching surface or in a volume, update `nee_p_cache` to the current position $\mathbf{p}$. If on an index-matching surface, set `nee_p_cache = None`.
        *   If transmittance sampling occurred through index-matching surfaces, update `multi_trans_pdf` by multiplying with the transmittance PDF.
    d.  **Next Event Estimation (NEE)**:
        If `nee_p_cache` is not `None`:
        i.  Sample a point on a light source $\mathbf{p}_L$ using `Light::SampleLi`. This consumes 2D samples. Pass `allowIncompletePDF = true` [S3].
        ii. Compute `lightPDF` = $p_{light\_select} \times p_{point\_on\_light}(\mathbf{p}_L | \mathbf{p}_{cache})$ [S3].
        iii. Compute `scatterPDF` using the cached `dir_pdf` and `multi_trans_pdf`. The directional PDF for the light direction is derived from the cached phase function sampling and transmittance sampling: $p_{phase\_to\_light} = dir\_pdf \times multi\_trans\_pdf \times G(\mathbf{p}_{cache}, \mathbf{p}_L)$, where $G$ is the geometry term [S1].
        iv. Compute MIS weight $w$ using the power heuristic:
        $$ w = \frac{p_{phase}^2}{p_{phase}^2 + p_{nee}^2} $$
        where $p_{phase}$ is the PDF of the phase function sampling strategy (including transmittance) and $p_{nee}$ is the PDF of the NEE strategy [S2].
        v.  Compute contribution:
        $$ L_{contrib} = \text{current\_path\_throughput} \times L_e(\mathbf{p}_L, -\boldsymbol{\omega}_{light}) \times \frac{w}{p_{nee}} $$
        [G] If the light is a delta distribution, $w=1$ and only light sampling is used.
        vi. Add $L_{contrib}$ to `radiance`.

    e.  **Scattering**:
        If the interaction is a volume scattering event:
        i.  Sample outgoing direction $\boldsymbol{\omega}'$ from phase function $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$.
        ii. Update `current_path_throughput` by multiplying with $\sigma_s / p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ [S2].
        iii. Update ray: $\mathbf{o} \leftarrow \mathbf{o} + t\boldsymbol{\omega}$, $\boldsymbol{\omega} \leftarrow \boldsymbol{\omega}'$.
        iv. Increment `bounces`. Set `never_scatter = false`.
        v.  Russian Roulette: Terminate path with probability $1 - \min(1, \text{throughput})$.

    f.  **Surface Hit**:
        If the interaction is a surface hit:
        i.  If the surface is emissive and `never_scatter` is true, add emission directly without MIS [S1].
        ii. If the surface is not index-matching, perform NEE as in step 2d.
        iii. If the surface is index-matching, skip scattering and continue loop (updating caches appropriately) [S1].

3.  **Return**:
    Return `radiance`.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_ld(intr, beta, r_p, light_sampler, scene, sampler):
    """
    Estimates direct lighting at interaction intr using MIS.
    intr: Interaction (surface or volume)
    beta: Path throughput from camera to intr
    r_p: Rescaled path probability for unidirectional sampling
    light_sampler: Light sampler object
    scene: Scene object
    sampler: Random sampler
    """
    # Initialize rescaled path probabilities
    r_l = 0.0
    r_u = 0.0
    
    # Sample light source
    u = sampler.get_1d()
    sampled_light = light_sampler.sample(intr.context, u)
    if not sampled_light:
        return 0.0
    
    light = sampled_light.light
    u_light = sampler.get_2d()
    ls = light.sample_li(intr.context, u_light, lambda, allow_incomplete_pdf=True)
    
    if not ls or ls.L == 0 or ls.pdf == 0:
        return 0.0
        
    light_pdf = sampled_light.p * ls.pdf
    
    # Evaluate BSDF or Phase Function
    wo = intr.wo
    wi = ls.wi
    f_hat = 0.0
    scatter_pdf = 0.0
    
    if intr.is_surface():
        bsdf = intr.bsdf
        f_hat = bsdf.f(wo, wi) * abs_dot(wi, intr.shading_normal)
        scatter_pdf = bsdf.pdf(wo, wi)
    else:
        phase = intr.medium.phase
        f_hat = phase.p(wo, wi)
        scatter_pdf = phase.pdf(wo, wi)
        
    if f_hat == 0:
        return 0.0
        
    # Compute transmittance to light
    T_ray = scene.transmittance(intr.p, ls.p_light)
    if T_ray == 0:
        return 0.0
        
    # Update rescaled path probabilities
    r_l *= r_p * light_pdf
    r_u *= r_p * scatter_pdf
    
    # Return estimate
    if is_delta_light(light.type()):
        # Only light sampling is applicable
        return beta * f_hat * T_ray * ls.L / r_l.average()
    else:
        # MIS with both strategies
        return beta * f_hat * T_ray * ls.L / (r_l + r_u).average()

def vol_path_tracing(screen_pos, rng, scene):
    current_path_throughput = 1.0
    radiance = 0.0
    bounces = 0
    dir_pdf = 0.0  # Solid angle measure
    nee_p_cache = None
    multi_trans_pdf = 1.0
    never_scatter = True
    
    ray = Ray(screen_pos, sample_direction(screen_pos, rng))
    
    while True:
        # Distance sampling (Delta Tracking or Homogeneous)
        t, scatter = sample_distance(ray, scene, rng)
        if not scatter:
            # Hit surface or escaped
            if ray.escaped():
                # Handle infinite lights
                for light in scene.infinite_lights:
                    Le = light.le(ray, lambda)
                    if Le:
                        if bounces == 0 or never_scatter:
                            radiance += current_path_throughput * Le / r_u.average()
                        else:
                            light_pdf = light_sampler.pmf(prev_intr_context, light) * light.pdf_li(prev_intr_context, ray.d, True)
                            r_l *= light_pdf
                            radiance += current_path_throughput * Le / (r_u + r_l).average()
                break
            
            isect = scene.intersect(ray, t)
            if isect.is_emissive():
                Le = isect.le(-ray.d, lambda)
                if Le:
                    if bounces == 0 or never_scatter:
                        radiance += current_path_throughput * Le / r_u.average()
                    else:
                        area_light = Light(isect.area_light)
                        light_pdf = light_sampler.pmf(prev_intr_context, area_light) * area_light.pdf_li(prev_intr_context, ray.d, True)
                        r_l *= light_pdf
                        radiance += current_path_throughput * Le / (r_u + r_l).average()
            break
            
        # Volume scattering
        p = ray.o + t * ray.d
        current_path_throughput *= (transmittance / avg(trans_dir_pdf)) # Ratio tracking
        
        # Update caches
        dir_pdf = phase.pdf(ray.d, new_dir)
        if not is_index_matching_surface(p):
            nee_p_cache = p
        else:
            nee_p_cache = None
        multi_trans_pdf *= trans_pdf
            
        # NEE
        if nee_p_cache is not None:
            # Sample light
            ls = light.sample_li(nee_p_cache, ...)
            if ls:
                light_pdf = ls.p * ls.pdf
                # Phase PDF for light direction
                phase_pdf = dir_pdf * multi_trans_pdf * geometry_term(nee_p_cache, ls.p_light)
                
                w = (phase_pdf**2) / (phase_pdf**2 + light_pdf**2)
                radiance += current_path_throughput * ls.L * w / light_pdf
                
        # Scatter
        new_dir = phase.sample(ray.d, rng)
        current_path_throughput *= sigma_s / phase.pdf(ray.d, new_dir)
        ray = Ray(p, new_dir)
        bounces += 1
        never_scatter = False
        
        # Russian Roulette
        if rng.uniform() > current_path_throughput.max():
            break
```

**Parameters**
*   `dir_pdf`: Cached PDF of the latest phase function sampling in solid angle measure [S2].
*   `nee_p_cache`: Last position $\mathbf{p}$ that can issue NEE [S1].
*   `multi_trans_pdf`: Product PDF of transmittance sampling through index-matching surfaces [S1].
*   `beta`: Path throughput factor [S3].
*   `r_p`: Rescaled path probability for unidirectional sampling [S3].
*   `r_l`: Rescaled path probability for light sampling [S3].
*   `r_u`: Rescaled path probability for unidirectional sampling [S3].

**Edge Cases and Pitfalls**
*   **Index-Matching Surfaces**: Quantities required for MIS (PDFs) may exist several bounces ago if the path skips index-matching surfaces. This is resolved by caching `dir_pdf`, `nee_p_cache`, and `multi_trans_pdf` during the main loop [S1, S2].
*   **Delta Lights**: If the light source is a delta distribution, sampling the BSDF or phase function cannot intersect the light. In this case, only the light sampling technique is applicable, and MIS weights are based solely on the per-wavelength PDFs for the unidirectional sampling strategy to reduce variance in chromatic media [S3].
*   **Delta Bounces**: If the previous scattering event was due to a delta-distribution scattering function, sampling the light is not a useful strategy. The MIS weight is only based on the per-wavelength PDFs for the unidirectional sampling strategy [S3].
*   **$\hat{f}$ Definition**: The variable $\hat{f}$ includes the cosine factor for surface scattering but does NOT include the normal scattering coefficient $\sigma_s$ for volumetric scattering, as $\sigma_s$ is already accounted for in the $\beta$ term [S3].

## Subtopics

### [Extended Multiple Importance Sampling](1569-extended-multiple-importance-sampling.md)
Extended Multiple Importance Sampling (Extended MIS) is a framework for combining estimators of integrals defined on spaces of different dimensions, such as standard path sampling and extended-space density estimators (e.g., photon mapping or beam tracing). It solves the problem of variance reduction in volumetric path tracing when mixing unbiased path contributions with biased density estimates that operate on higher-dimensional spaces. Unlike standard MIS, which requires all strategies to sample from the same domain, Extended MIS derives a balance heuristic that accounts for the additional integration over the kernel support inherent in extended-space techniques. See [Extended Multiple Importance Sampling](1569-extended-multiple-importance-sampling.md).

### [Multi-View Path Tracing with MWIS](1834-multi-view-path-tracing-with-mwis.md)
Multi-View Path Tracing with MWIS (MVPT) accelerates the rendering of heterogeneous participating media by exploiting redundancy across multiple camera views, such as in animated sequences or light fields. It jointly renders a sequence by reusing computation during simulation, specifically by transforming light paths from a base view to other views and generating a shared path suffix. Unlike standard multi-view methods that use unbiased Multiple Importance Sampling (MIS) or discard shifted paths based on strict selection criteria, MVPT uses a self-normalized Multiple Weighted Importance Sampling (MWIS) estimator. This approach reweights samples to match a target probability density function (PDF), reducing variance caused by non-contributing strategies and large Jacobian variations, while remaining consistent with a rapidly vanishing bias. See [Multi-View Path Tracing with MWIS](1834-multi-view-path-tracing-with-mwis.md).

### [Weighted Importance Sampling](1849-weighted-importance-sampling.md)
Weighted Importance Sampling (WIS) is a biased but consistent technique used to simulate the use of a known target probability density function (PDF) $p$ with samples drawn from an arbitrary source PDF $q$. It addresses the problem of evaluating integrals where the ideal sampling distribution is known but difficult or impossible to sample from directly, such as when shifted paths in volumetric rendering do not follow the base PDF of sampling the path from the pixel. It sets itself apart from Resampled Importance Sampling (RIS) by avoiding the intractable resampling of path prefixes required for unbounded paths, accepting bias in exchange for feasibility. See [Weighted Importance Sampling](1849-weighted-importance-sampling.md).

### [Direct Lighting with MIS in VolPathIntegrator](1877-direct-lighting-with-mis-in-volpathintegrator.md)
Direct Lighting with MIS in VolPathIntegrator is a method to sample direct illumination in volumetric media by combining unidirectional path sampling and light path sampling using Multiple Importance Sampling (MIS). It solves the high variance problem in spectral volumetric path tracing by allowing the estimator to choose between sampling the phase function to find the next vertex or sampling the light source to connect directly, weighting the contributions via the power heuristic. Unlike standard spectral MIS which may suffer from high variance in heterogeneous media, this method specifically handles the transmittance estimation via ratio tracking for the light connection segment, reducing Mean Squared Error (MSE) significantly compared to non-MIS approaches. See [Direct Lighting with MIS in VolPathIntegrator](1877-direct-lighting-with-mis-in-volpathintegrator.md).

### Power Heuristic for Light Sampling
A heuristic used to balance the contribution of different sampling strategies (e.g., light sampling vs. BRDF sampling) in path tracing to reduce variance [S4, unofficial]. The weight is computed as:
$$ \text{weight} = \frac{(n_f \cdot f_{\text{Pdf}})^2}{(n_f \cdot f_{\text{Pdf}})^2 + (n_g \cdot g_{\text{Pdf}})^2} $$
where $n_f$ and $n_g$ are the number of samples for each strategy, and $f_{\text{Pdf}}$ and $g_{\text{Pdf}}$ are the respective probability density functions [S4, unofficial].

### Multiple Importance Sampling for Phase Function and Next Event Estimation
Combining phase function sampling with next event estimation (NEE) by accumulating PDFs for ratio tracking (free-flight sampling decisions) and free-flight sampling (real/fake particle events) during NEE. When evaluating the PDF for one, the PDF of the other must also be accumulated [S2]. The path throughput is updated as:
$$ \text{current\_path\_throughput} *= \left( \frac{\text{transmittance}}{\text{avg}(\text{trans\_dir\_pdf})} \right) $$
[S2].

### Resample Importance Sampling (RIS) for Volumetric Path Tracing
Extended Müller et al. to translucent materials. Used a spatial-directional tree to represent the incident radiance distribution and then sample the outgoing direction after scattering considering both the learned lighting distribution and the phase function [S5]. Introduced resample importance sampling (RIS) to jointly sample the lighting distribution and the phase function, as they observed the low sampling quality of MIS for the product of two high-frequency functions [S5]. RIS is combined with MIS, depending on the sharpness of the phase function, to save the computational cost in RIS [S5]. Pitfalls include using naive distance sampling, which has a significant impact on convergence speed [S5]. It can handle any homogeneous participating media, from high scattering to low scattering, from high absorption to low absorption, and from isotropic media to highly anisotropic media [S5].

## Operating characteristics
*   **Time Complexity**: $O(N)$ per path, where $N$ is the number of bounces. MIS adds negligible overhead compared to the cost of NEE and phase function sampling.
*   **Memory Complexity**: $O(1)$ additional memory for caching `dir_pdf`, `nee_p_cache`, and `multi_trans_pdf` per path.
*   **Measured Performance**: Reduces variance significantly in scenes with index-matching surfaces and chromatic media compared to non-MIS approaches [S1, S3].
*   **Scaling**: Scales linearly with the number of light sources if NEE is used for all lights.
*   **Strengths**: Reduces variance in complex scenes with index-matching surfaces and chromatic media. Handles both delta and non-delta lights consistently.
*   **Limitations**: Requires careful caching of PDFs across index-matching surfaces. Computationally more expensive than simple NEE due to the need to compute both PDFs and the MIS weight.
*   **When to Use**: Scenes with index-matching surfaces, chromatic media, or where both phase function sampling and NEE are likely to contribute significantly.
*   **When Not to Use**: Simple scenes with no index-matching surfaces where NEE alone is sufficient.

## Background
Originated from the need to handle index-matching surfaces in volumetric path tracing, where paths skip scattering events without sampling new directions, making standard MIS weights invalid [S1, S2]. Developed as part of the VolPathIntegrator in the PBRT framework [S3]. Used in modern volumetric path tracers to handle complex participating media with refractive boundaries [S5].

## Sources
[S1] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S2] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S4] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
[S5] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
