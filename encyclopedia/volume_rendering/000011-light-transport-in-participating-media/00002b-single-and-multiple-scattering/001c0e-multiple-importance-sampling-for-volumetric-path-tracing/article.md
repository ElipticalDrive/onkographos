# Multiple Importance Sampling for Volumetric Path Tracing

Also called: Volumetric MIS, Multiple Importance Sampling (MIS) for Volumometric Path Tracing

## Overview
Multiple Importance Sampling (MIS) for volumetric path tracing reduces variance in estimating direct lighting by combining contributions from phase function sampling and next event estimation (NEE) [S1, S2]. It solves the problem of high variance when sampling light sources in participating media, particularly when paths skip index-matching surfaces or involve delta distributions [S1, S3]. Unlike standard path tracing which may rely on a single sampling strategy, this method uses the power heuristic to balance the probability density functions (PDFs) of both strategies, ensuring robust convergence in heterogeneous and chromatic media [S2, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Current path vertex position $\mathbf{p}$, incoming direction $\boldsymbol{\omega}_i$, current path throughput $\beta$, scattering function value $\hat{f}$, transmittance $T_{ray}$, and light source radiance $L_s$.
*   **Output:** Estimated radiance contribution weighted by MIS weights.

**Data Structures:**
To handle index-matching surfaces that skip scattering events, the integrator caches specific quantities during the main path loop:
1.  `dir_pdf`: The PDF of the latest phase function sampling in solid angle measure ($\text{rad}^{-2}$) [S1, S2].
2.  `nee_p_cache`: The position $\mathbf{p}$ of the last vertex capable of issuing NEE. If the current vertex is on an index-matching surface, this remains the previous valid position [S1, S2].
3.  `multi_trans_pdf`: The product PDF of transmittance sampling through index-matching surfaces since the last phase function sampling [S1, S2].

**Numbered Steps:**
1.  **Initialize Cache:** At the start of the path loop, initialize `dir_pdf = 0`, `nee_p_cache = None`, and `multi_trans_pdf = 1` [S1, S2].
2.  **Update Cache:**
    *   If a phase function sample is taken, update `dir_pdf` with the solid angle PDF and reset `multi_trans_pdf` to 1 [S1].
    *   If the path traverses an index-matching surface without scattering, multiply `multi_trans_pdf` by the transmittance sampling PDF [S1].
    *   If the current vertex is not on an index-matching surface, update `nee_p_cache` to the current position $\mathbf{p}$ [S1].
3.  **Sample Light Source:** When NEE is performed, sample a point on the light source using `Light::SampleLi`, consuming two 2D sample dimensions. Pass `true` for `allowIncompletePDF` to enable MIS [S3]. If no valid sample or zero radiance/PDF, return zero [S3].
4.  **Compute Light PDF:** Calculate the light sampling PDF as the product of the light selection probability $p_l$ and the point sampling PDF $p_{ls}$:
    $$ p_{light} = p_l \cdot p_{ls} $$
    [S3]
5.  **Compute Phase Function PDF:** The PDF for the phase function strategy accounts for the cached directional and transmittance components. Let $p_{phase}$ be the cached `dir_pdf` and $p_{trans}$ be the cached `multi_trans_pdf`. The geometric term $G$ (usually $1/r^2$ or similar projection factor) is included if not already in the PDF definition [S1].
    $$ p_{phase\_total} = p_{phase} \cdot p_{trans} \cdot G $$
    [S1]
    *Note: In ratio tracking implementations, the PDFs for free-flight sampling decisions must also be accumulated [S2].*
6.  **Calculate MIS Weight:** Compute the power heuristic weight $w$ using the squared PDFs [S2]:
    $$ w = \frac{p_{phase\_total}^2}{p_{phase\_total}^2 + p_{light}^2} $$
    [S2]
    *Special Case:* If the light source is a delta distribution or the previous bounce was a delta scattering event, only the unidirectional (phase function) strategy is valid. In this case, $w=1$ (or weights are based solely on unidirectional PDFs for chromatic variance reduction) [S3].
7.  **Accumulate Contribution:** Add the radiance contribution weighted by $w$:
    $$ L_{contrib} = \beta \cdot \hat{f} \cdot T_{ray} \cdot L_s \cdot w $$
    [S3]
    *Note:* $\hat{f}$ includes the cosine factor for surface scattering but does **not** include the scattering coefficient $\sigma_s$ for volumetric scattering, as $\sigma_s$ is already accounted for in $\beta$ [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_light_mis(intr, scene, sampler, beta, f_hat, T_ray, L_s):
    """
    intr: Interaction (surface or volume)
    scene: Scene object
    sampler: Sampler object
    beta: Path throughput
    f_hat: Scattering function value (BSDF or phase function)
    T_ray: Transmittance to light
    L_s: Light radiance
    """
    # 1. Sample Light Source
    u = sampler.Get1D()
    sampled_light = scene.light_sampler.Sample(intr.context, u)
    if not sampled_light:
        return 0.0
    
    light = sampled_light.light
    u_light = sampler.Get2D()
    
    # allowIncompletePDF=True enables MIS
    ls = light.SampleLi(intr.context, u_light, lambda, allow_incomplete_pdf=True)
    if not ls or ls.L == 0 or ls.pdf == 0:
        return 0.0
        
    # 2. Compute Light PDF
    light_pmf = scene.light_sampler.PMF(intr.context, light)
    light_pdf = light_pmf * ls.pdf
    
    # 3. Compute Phase Function PDF (using cached values from main loop)
    # dir_pdf is cached solid angle PDF of last phase sample
    # multi_trans_pdf is cached transmittance PDF product
    # G is geometric term (e.g., 1/distance^2) if not in PDF
    phase_pdf = intr.cached_dir_pdf * intr.cached_multi_trans_pdf * intr.G
    
    # 4. Handle Delta Lights / Specular Bounces
    if light.is_delta() or intr.prev_bounce_was_delta:
        # Only unidirectional strategy is valid
        # Use per-wavelength PDFs for variance reduction in chromatic media
        # For simplicity, assume scalar or averaged here; detailed impl uses per-wavelength
        return beta * f_hat * T_ray * L_s / intr.cached_r_u.Average()
        
    # 5. Compute MIS Weight (Power Heuristic)
    w = (phase_pdf * phase_pdf) / (phase_pdf * phase_pdf + light_pdf * light_pdf)
    
    # 6. Return Weighted Contribution
    # Note: f_hat does not include sigma_s
    return beta * f_hat * T_ray * L_s * w
```

## Subtopics

### [Extended Multiple Importance Sampling](1569-extended-multiple-importance-sampling.md)
Extended MIS combines estimators from different dimensional spaces, such as standard path sampling and extended-space density estimators (e.g., photon mapping). It solves variance reduction when mixing unbiased path contributions with biased density estimates by deriving a balance heuristic that accounts for integration over kernel support.

### [Multi-View Path Tracing with MWIS](1834-multi-view-path-tracing-with-mwis.md)
MVPT accelerates rendering of heterogeneous media across multiple views by reusing light paths. It uses Multi-View Weighted Importance Sampling (MWIS) to reweight samples to a target PDF, reducing variance from non-contributing strategies and Jacobian variations, while maintaining consistency with vanishing bias.

### [Weighted Importance Sampling](1849-weighted-importance-sampling.md)
WIS simulates a target PDF $p$ using samples from a source PDF $q$. It addresses intractable resampling of path prefixes in volumetric rendering by accepting bias for feasibility, unlike Resampled Importance Sampling (RIS).

### [Direct Lighting with MIS in VolPathIntegrator](1877-direct-lighting-with-mis-in-volpathintegrator.md)
This method samples direct illumination in volumetric media by combining unidirectional path sampling and light path sampling. It handles transmittance estimation via ratio tracking for light connection segments, reducing MSE compared to non-MIS approaches in heterogeneous media.

### [Unknown Distance PDF and MIS](9222-unknown-distance-pdf-and-mis.md)
This technique enables MIS for rejection-based trackers (like delta tracking) where the free-path PDF cannot be evaluated exactly. It ensures unknown distance PDFs cancel out in the MIS weight calculation, allowing weights to be computed based solely on directional PDFs.

### Power Heuristic for Light Sampling
A heuristic to balance light sampling vs. BRDF/phase function sampling [S4, unofficial]. The weight is calculated as:
$$ w = \frac{(n_f f_{Pdf})^2}{(n_f f_{Pdf})^2 + (n_g g_{Pdf})^2} $$
where $n_f, n_g$ are sample counts and $f_{Pdf}, g_{Pdf}$ are the respective PDFs [S4, unofficial].

### Multiple Importance Sampling for Phase Function and Next Event Estimation
Combines phase function sampling with NEE by accumulating PDFs for ratio tracking and free-flight sampling [S2]. When evaluating one PDF, the other must also be accumulated. The throughput update divides transmittance by the average PDF over channels:
$$ \text{current\_path\_throughput} *= \frac{\text{transmittance}}{\text{avg}(\text{trans\_dir\_pdf})} $$
[S2]

### Resample Importance Sampling (RIS) for Volumetric Path Tracing
RIS jointly samples lighting distribution and phase function, addressing low sampling quality of MIS for high-frequency products [S5]. It uses a spatial-directional tree to represent incident radiance and combines with MIS depending on phase function sharpness [S5]. Pitfalls include naive distance sampling impacting convergence speed [S5].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per light sample addition, but requires caching and additional PDF evaluations.
*   **Memory Complexity:** $O(1)$ additional state per path vertex for caching `dir_pdf`, `nee_p_cache`, and `multi_trans_pdf` [S1].
*   **Performance:** Significantly reduces variance in scenes with small lights or dense volumes compared to single-strategy sampling [S1, S3].
*   **Strengths:** Robust handling of index-matching surfaces, delta distributions, and chromatic media [S1, S3].
*   **Limitations:** Requires careful caching of PDFs across bounces; naive distance sampling can still impact convergence [S1, S5].

## Background
Originating from the need to combine phase function sampling and next event estimation in volumetric path tracing, this method addresses variance issues in participating media [S1, S2]. It builds on the conservation of energy principles and Beer-Lambert law transmittance [S3]. Key developments include handling index-matching surfaces by caching PDFs [S1] and extending to chromatic media with per-wavelength weights [S3].

## Sources
[S1] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S2] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
[S3] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S4] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
[S5] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
