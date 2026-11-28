# Direct Lighting with MIS in VolPathIntegrator

Also called: Direct Lighting with MIS

## Overview
Direct Lighting with MIS in VolPathIntegrator is a method to sample direct illumination in volumetric media by combining unidirectional path sampling and light path sampling using Multiple Importance Sampling (MIS) [S1]. It solves the high variance problem in spectral volumetric path tracing by allowing the estimator to leverage the strengths of both sampling strategies—phase function sampling for direction and light source sampling for geometry—while correctly accounting for the different PDFs introduced by ratio tracking in participating media [S1]. Unlike standard MIS which assumes a single sampling distribution, this method evaluates path PDFs at all wavelengths to enable MIS over wavelengths, significantly reducing color noise and variance compared to unsampled spectral approaches [S1].

## How it works
The method estimates the direct lighting contribution at a path vertex $\bar{p}_{n-1}$ by generating two candidate paths: one via unidirectional sampling and one via light path sampling [S1]. It computes the MIS weight for each based on their respective probabilities and combines them [S1].

### Inputs and Outputs
**Inputs:**
*   Path vertex $\bar{p}_{n-1}$ and its throughput $\beta$ [S1].
*   Scattering coefficient $\sigma_s(\bar{p}_{n-1})$ and null scattering coefficient $\sigma_n(\bar{p}_{n-1})$ at the vertex [S1].
*   Majorant scattering coefficient $\sigma_{maj}$ (constant or spatially varying upper bound) [S1].
*   Phase function $p_\omega(\omega)$ [S1].
*   Light source geometry and emission $L_e$ [S1].
*   Wavelengths $\lambda$ (typically spectral samples) [S1].

**Outputs:**
*   Estimated direct lighting contribution $L_{dir}$ [S1].

### Data Structures
*   **Path Vertex:** Contains position $p$, incoming direction $\omega_{in}$, and accumulated throughput $\beta$ [S1].
*   **Spectral Coefficients:** Arrays or vectors for $\sigma_s(\lambda)$, $\sigma_n(\lambda)$, and $\sigma_{maj}(\lambda)$ [S1].
*   **MIS Weights:** Scalar values $w_u$ and $w_l$ computed via the power heuristic [S1].

### Numbered Steps

1.  **Initialize Common Path State:**
    For the current path $\bar{p}_{n-1}$, the throughput $\beta$ and the path PDF $p_u(\bar{p}_{n-1})$ are identical for both sampling strategies up to this vertex [S1].

2.  **Unidirectional Path Sampling ($p_u$):**
    a.  Sample a direction $\omega_n$ from the phase function $p_\omega(\omega_n)$ at $p_{n-1}$ [S1].
    b.  Use delta tracking to find the next real-scattering vertex $p_n$ along $\omega_n$ [S1].
    c.  The probability of a path segment from $p_i$ to $p_j$ with $m$ intermediate null-scattering vertices is:
        $$ p_{null}(p_i, p_j) = \left( \prod_{k=1}^{m} \sigma_n(p_{i+k}) T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) \right) T_{maj}(p_{i+m} \rightarrow p_j) $$
        where the acceptance probability for a null scattering event at $p_{i+k}$ is:
        $$ p_e(p_{i+k}) = \frac{\sigma_n(p_{i+k})}{\sigma_{maj}} $$
        and the majorant PDF for the distance is:
        $$ p_{maj}(p_{i+k-1} \rightarrow p_{i+k}) = \sigma_{maj} T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) $$
    d.  The full unidirectional path PDF is:
        $$ p_u(\bar{p}_n) = p_u(\bar{p}_{n-1}) \frac{\sigma_s(p_{n-1})}{\sigma_{maj}} p_\omega(\omega_n) p_{null}(p_{n-1}, p_n) $$

3.  **Light Path Sampling ($p_l$):**
    a.  Follow the same approach up to the last real-scattering vertex before the light vertex; there, the light selects the direction and ratio tracking gives the transmittance along the last path segment [S1].
    b.  Trace a shadow ray from $p_{n-1}$ towards the light [S1].
    c.  Use ratio tracking to estimate the transmittance along this segment [S1]. The PDF for a path sampled between points $p_i$ and $p_j$ with $m$ intermediate vertices using ratio tracking is:
        $$ p_{ratio}(p_i, p_j) = \left( \prod_{k=1}^{m} T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) \sigma_{maj} \right) T_{maj}(p_{i+m} \rightarrow p_j) $$
    d.  The full light path PDF is:
        $$ p_l(\bar{p}_n) = p_u(\bar{p}_{n-1}) \frac{\sigma_s(p_{n-1})}{\sigma_{maj}} p_{l,\omega}(\omega_n) p_{ratio}(p_{n-1}, p_n) $$
        where $p_{l,\omega}$ is the PDF of the light direction sampling [S1].

4.  **Spectral Evaluation:**
    The VolPathIntegrator samples both types of paths according to the first wavelength $\lambda_1$ but evaluates the PDFs $p_u$ and $p_l$ at all wavelengths $\lambda$ to enable MIS over wavelengths [S1].

5.  **MIS Weighting:**
    Compute weights $w_u(\bar{p}_n)$ and $w_l(\bar{p}'_n)$ using the power heuristic based on the evaluated PDFs at each wavelength [S1].

6.  **Estimator Combination:**
    The final contribution is the sum of the weighted contributions from both strategies:
    $$ L_{dir} = w_u(\bar{p}_n) \frac{\hat{T}(\bar{p}_n) L_e(p_n \rightarrow p_{n-1})}{p_{u, \lambda_1}(\bar{p}_n)} + w_l(\bar{p}'_n) \frac{\hat{T}(\bar{p}'_n) L_e(p' \rightarrow p'_{n-1})}{p_{l, \lambda_1}(\bar{p}'_n)} $$
    where $\hat{T}$ is the estimated transmittance [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_lighting_mis(p_n_minus_1, sigma_s, sigma_n, sigma_maj, phase_func, light_sampler, wavelengths):
    # 1. Common state
    p_u_prev = p_n_minus_1.path_pdf
    
    # 2. Unidirectional path sampling
    omega_u = phase_func.sample(p_n_minus_1.position)
    # Delta tracking finds real-scattering vertex p_n
    p_n, T_hat_u, null_vertices_u = delta_tracking(p_n_minus_1.position, omega_u, sigma_s, sigma_n, sigma_maj)
    
    # Compute p_null(p_{n-1}, p_n)
    p_null_u = 1.0
    for k in range(1, len(null_vertices_u) + 1):
        p_null_u *= sigma_n(null_vertices_u[k-1]) * T_maj(p_n_minus_1.position if k == 1 else null_vertices_u[k-2], null_vertices_u[k-1])
    p_null_u *= T_maj(null_vertices_u[-1] if null_vertices_u else p_n_minus_1.position, p_n)
    
    # Full unidirectional PDF
    p_u_val = p_u_prev * (sigma_s(p_n_minus_1.position) / sigma_maj(p_n_minus_1.position)) * \
              phase_func.pdf(omega_u) * p_null_u
    
    # 3. Light path sampling
    light_sample = light_sampler.sample(p_n_minus_1.position)
    omega_l = light_sample.direction
    L_e = light_sample.emission
    p_l_omega = light_sample.pdf
    
    # Ratio tracking for transmittance and PDF
    t_l, null_vertices_l = ratio_tracking(p_n_minus_1.position, omega_l, sigma_maj)
    p_ratio_l = 1.0
    for k in range(1, len(null_vertices_l) + 1):
        p_ratio_l *= T_maj(p_n_minus_1.position if k == 1 else null_vertices_l[k-2], null_vertices_l[k-1]) * sigma_maj(null_vertices_l[k-1])
    p_ratio_l *= t_l
    
    # Full light path PDF
    p_l_val = p_u_prev * (sigma_s(p_n_minus_1.position) / sigma_maj(p_n_minus_1.position)) * \
              p_l_omega * p_ratio_l
    
    # 4 & 5. Spectral MIS weighting
    total_contribution = 0.0
    for lam in wavelengths:
        p_u_lam = p_u_val[lam]
        p_l_lam = p_l_val[lam]
        denom = p_u_lam**2 + p_l_lam**2
        w_u_lam = p_u_lam**2 / denom if denom > 0 else 0.0
        w_l_lam = p_l_lam**2 / denom if denom > 0 else 0.0
        
        # 6. Estimator combination
        # Denominator uses lambda_1 (first wavelength) as per source
        p_u_lam1 = p_u_val[wavelengths[0]]
        p_l_lam1 = p_l_val[wavelengths[0]]
        
        contrib_u = w_u_lam * (T_hat_u * L_e) / p_u_lam1
        contrib_l = w_l_lam * (t_l * L_e) / p_l_lam1
        total_contribution += contrib_u + contrib_l
        
    return total_contribution

def delta_tracking(start, direction, sigma_s, sigma_n, sigma_maj):
    # Standard delta tracking: sample distance from majorant, accept with prob sigma_s/sigma_maj
    # Returns (vertex, transmittance, null_vertices)
    pass # Sources do not give the delta tracking implementation details.

def ratio_tracking(start, direction, sigma_maj):
    # Standard ratio tracking: sample distance from majorant, accumulate transmittance
    # Returns (transmittance, null_vertices)
    pass # Sources do not give the ratio tracking implementation details.
```

### Parameters
*   **$\sigma_{maj}$**: Majorant scattering coefficient, an upper bound for $\sigma_s + \sigma_a$ [S1].
*   **Power Heuristic Exponent**: Typically 2 [G].

### Edge Cases and Pitfalls
*   **Redundant Computation**: Paths share the same vertices for all of $\bar{p}_{n-1}$, so the transmittance factors and PDFs share common factors, requiring careful cancellation or computation to avoid redundancy [S1].
*   **Precision**: Products of transmittances in delta and ratio tracking can underflow; log-space arithmetic is recommended [G].
*   **Boundary Conditions**: $\sigma_{maj}$ must be correctly computed at boundaries of heterogeneous media [G].

## Subtopics

### [Spectral+NEE MIS (Shadow Connection Routine)](1907-spectral-nee-mis-shadow-connection-routine.md)
Uses ratio tracking to handle null scattering events for robust connections through dense or thin media while maintaining color fidelity.

### [Spectral MIS with Unidirectional Path Tracing](1964-spectral-mis-with-unidirectional-path-tracing.md)
Samples a single random color channel to reduce computational time and increase sampled distances.

## Operating characteristics
*   **Time Complexity**: Proportional to the number of scattering events found via delta tracking and the cost of light sampling. Ratio tracking adds overhead for shadow ray transmittance estimation.
*   **Memory Complexity**: Low, as it does not store large path histories beyond the current vertex.
*   **Performance**: Reduces MSE by a factor of 149 in tested scenes compared to non-spectral MIS [S1].
*   **Strengths**: Significantly reduces variance in chromatic participating media. Handles heterogeneous media robustly.
*   **Limitations**: Requires evaluation of PDFs at all wavelengths, increasing computational cost per path vertex.

## Background
This method is described in the context of the VolPathIntegrator in the "Physically Based Rendering" (PBRT) book, specifically in Section 14.2 "Volume Scattering Integrators" [S1]. It builds on the principles of Multiple Importance Sampling introduced in earlier chapters and extends them to the complexities of volumetric path tracing with spectral rendering.

## Sources
[S1] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
