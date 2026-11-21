# Direct Lighting with MIS in VolPathIntegrator

Also called: Direct Lighting with MIS

## Overview
Direct Lighting with MIS in VolPathIntegrator is a method to sample direct illumination in volumetric media by combining unidirectional path sampling and light path sampling using Multiple Importance Sampling (MIS) [S1]. It solves the high variance problem in spectral volumetric path tracing by allowing the estimator to leverage the strengths of both sampling strategies—phase function sampling for direction and light source sampling for geometry—while correctly accounting for the different PDFs introduced by ratio tracking in participating media [S1]. Unlike standard MIS which assumes a single sampling distribution, this method evaluates path PDFs at all wavelengths to enable MIS over wavelengths, significantly reducing color noise and variance compared to unsampled spectral approaches [S1].

## How it works
The method estimates the direct lighting contribution at a path vertex $p_{n-1}$ by generating two candidate paths: one via unidirectional sampling and one via light path sampling. It computes the MIS weight for each based on their respective probabilities and combines them.

### Inputs and Outputs
**Inputs:**
*   Path vertex $p_{n-1}$ and its throughput $\beta$.
*   Scattering coefficient $\sigma_s(p_{n-1})$ and null scattering coefficient $\sigma_n(p_{n-1})$ at the vertex.
*   Majorant scattering coefficient $\sigma_{maj}$ (constant or spatially varying upper bound).
*   Phase function $p_\omega(\omega)$.
*   Light source geometry and emission $L_e$.
*   Wavelengths $\lambda$ (typically spectral samples).

**Outputs:**
*   Estimated direct lighting contribution $L_{dir}$.

### Data Structures
*   **Path Vertex:** Contains position $p$, incoming direction $\omega_{in}$, and accumulated throughput $\beta$.
*   **Spectral Coefficients:** Arrays or vectors for $\sigma_s(\lambda)$, $\sigma_n(\lambda)$, and $\sigma_{maj}(\lambda)$.
*   **MIS Weights:** Scalar values $w_u$ and $w_l$ computed via the power heuristic.

### Numbered Steps

1.  **Initialize Common Path State:**
    For the current path $\bar{p}_{n-1}$, the throughput $\beta$ and the path PDF $p_u(\bar{p}_{n-1})$ are identical for both sampling strategies up to this vertex [S1].

2.  **Unidirectional Path Sampling ($p_u$):**
    a.  Sample a direction $\omega_n$ from the phase function $p_\omega(\omega_n)$ at $p_{n-1}$ [S1].
    b.  Use **delta tracking** to find the next real-scattering vertex $p_n$ along $\omega_n$. This involves sampling null-scattering events.
    c.  The probability of a path segment from $p_i$ to $p_j$ with $m$ intermediate null-scattering vertices is:
        $$ p_{null}(p_i, p_j) = \left( \prod_{k=1}^{m} \sigma_n(p_{i+k}) T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) \right) T_{maj}(p_{i+m} \rightarrow p_j) $$
        where the acceptance probability for a null scattering event at $p_{i+k}$ is:
        $$ p_e(p_{i+k}) = \frac{\sigma_n(p_{i+k})}{\sigma_{maj}} $$
        and the majorant PDF for the distance is:
        $$ p_{maj}(p_{i+k-1} \rightarrow p_{i+k}) = \sigma_{maj} T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) $$
    d.  The full unidirectional path PDF is:
        $$ p_u(\bar{p}_n) = p_u(\bar{p}_{n-1}) \frac{\sigma_s(p_{n-1})}{\sigma_{maj}} p_\omega(\omega_n) p_{null}(p_{n-1}, p_n) $$

3.  **Light Path Sampling ($p_l$):**
    a.  Sample a point on a light source and compute the direction $\omega_{light}$ towards $p_{n-1}$ [S1].
    b.  Trace a shadow ray from $p_{n-1}$ towards the light.
    c.  Use **ratio tracking** to estimate the transmittance along this segment. The PDF for a path sampled between points $p_i$ and $p_j$ with $m$ intermediate vertices using ratio tracking is:
        $$ p_{ratio}(p_i, p_j) = \left( \prod_{k=1}^{m} T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) \sigma_{maj} \right) T_{maj}(p_{i+m} \rightarrow p_j) $$
    d.  The full light path PDF is:
        $$ p_l(\bar{p}_n) = p_u(\bar{p}_{n-1}) \frac{\sigma_s(p_{n-1})}{\sigma_{maj}} p_{l,\omega}(\omega_n) p_{ratio}(p_{n-1}, p_n) $$
        where $p_{l,\omega}$ is the PDF of the light direction sampling.

4.  **Spectral Evaluation:**
    The VolPathIntegrator samples both types of paths according to the first wavelength $\lambda_1$ but evaluates the PDFs $p_u$ and $p_l$ at all wavelengths $\lambda$ to enable MIS over wavelengths [S1].

5.  **MIS Weighting:**
    Compute weights $w_u(\bar{p}_n)$ and $w_l(\bar{p}'_n)$ using the power heuristic (typically $\frac{w^2}{w_u^2 + w_l^2}$) based on the evaluated PDFs at each wavelength [S1].

6.  **Estimator Combination:**
    The final contribution is the sum of the weighted contributions from both strategies:
    $$ L_{dir} = w_u(\bar{p}_n) \frac{\hat{T}(\bar{p}_n) L_e(p_n \rightarrow p_{n-1})}{p_{u, \lambda_1}(\bar{p}_n)} + w_l(\bar{p}'_n) \frac{\hat{T}(\bar{p}'_n) L_e(p' \rightarrow p'_{n-1})}{p_{l, \lambda_1}(\bar{p}'_n)} $$
    where $\hat{T}$ is the estimated transmittance [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_lighting_mis(p_n_minus_1, sigma_s, sigma_n, sigma_maj, phase_func, light_sampler):
    """
    Computes direct lighting contribution at p_n_minus_1 using MIS.
    """
    # 1. Common state
    beta = p_n_minus_1.throughput
    p_u_prev = p_n_minus_1.path_pdf # PDF up to p_{n-1}
    
    # 2. Unidirectional Sampling
    # Sample direction from phase function
    omega_u = phase_func.sample(p_n_minus_1.position)
    
    # Delta tracking to find next vertex p_n
    p_n, t_u, null_count = delta_tracking(p_n_minus_1.position, omega_u, sigma_s, sigma_n, sigma_maj)
    
    # Calculate unidirectional PDF components
    # p_e = sigma_n / sigma_maj
    # p_maj = sigma_maj * T_maj
    # p_null = product(sigma_n * T_maj) * T_maj_last
    # Simplified: The text defines p_null explicitly. We compute it here.
    p_null_u = compute_null_pdf(p_n_minus_1.position, p_n, sigma_n, sigma_maj)
    
    # Full unidirectional PDF
    # p_u(p_n) = p_u(p_{n-1}) * (sigma_s / sigma_maj) * p_omega(omega_n) * p_null(p_{n-1}, p_n)
    p_u_val = p_u_prev * (sigma_s(p_n_minus_1.position) / sigma_maj(p_n_minus_1.position)) * \
              phase_func.pdf(omega_u) * p_null_u
    
    # 3. Light Path Sampling
    # Sample light direction
    light_sample = light_sampler.sample(p_n_minus_1.position)
    omega_l = light_sample.direction
    L_e = light_sample.emission
    p_l_omega = light_sample.pdf
    
    # Ratio tracking for transmittance and PDF
    # p_ratio = product(T_maj * sigma_maj) * T_maj_last
    t_l, p_ratio_l = ratio_tracking(p_n_minus_1.position, omega_l, sigma_maj)
    
    # Full light path PDF
    # p_l(p_n) = p_u(p_{n-1}) * (sigma_s / sigma_maj) * p_{l,omega}(omega_n) * p_ratio(p_{n-1}, p_n)
    p_l_val = p_u_prev * (sigma_s(p_n_minus_1.position) / sigma_maj(p_n_minus_1.position)) * \
              p_l_omega * p_ratio_l
    
    # 4. Spectral MIS
    # Evaluate PDFs at all wavelengths lambda
    # w_u, w_l are computed using power heuristic: w_u = p_u^2 / (p_u^2 + p_l^2)
    # Note: The text implies MIS over wavelengths.
    total_contribution = 0
    for lambda_idx in wavelengths:
        p_u_lambda = p_u_val[lambda_idx]
        p_l_lambda = p_l_val[lambda_idx]
        
        # Avoid division by zero
        denom = p_u_lambda**2 + p_l_lambda**2
        if denom == 0:
            w_u_lambda = 0
            w_l_lambda = 0
        else:
            w_u_lambda = p_u_lambda**2 / denom
            w_l_lambda = p_l_lambda**2 / denom
            
        # Contribution from unidirectional path
        # Term: w_u * (T_hat * L_e) / p_{u, lambda_1}
        # Note: The denominator uses lambda_1 (the sampled wavelength), but weights are spectral.
        # The text formula: w_u(p_n) * (T_hat(p_n) L_e(...)) / p_{u, lambda_1}(p_n)
        # This implies the weight w_u might be aggregated or the formula is per-wavelength.
        # Based on "MIS over wavelengths", we likely sum the weighted contributions.
        
        # Re-reading formula: w_u(p_n) [T_hat(p_n) L_e / p_{u, lambda_1}(p_n)]
        # It seems w_u is a scalar weight for the whole path, but evaluated per wavelength?
        # "evaluates these PDFs at all wavelengths so that MIS over wavelengths can be used"
        # Standard spectral MIS sums the contributions.
        
        contrib_u = w_u_lambda * (t_u * L_e) / p_u_lambda # Using p_u_lambda for consistency in spectral MIS
        contrib_l = w_l_lambda * (t_l * L_e) / p_l_lambda
        
        total_contribution += contrib_u + contrib_l
        
    return total_contribution

def compute_null_pdf(start, end, sigma_n, sigma_maj):
    """
    Computes p_null(p_i, p_j) for delta tracking.
    """
    # This requires iterating through null scattering events.
    # For implementation simplicity, we assume delta_tracking returns the necessary components.
    # p_null = (prod sigma_n * T_maj) * T_maj_last
    pass
```

### Parameters
*   **$\sigma_{maj}$**: Majorant scattering coefficient. Must be $\ge \sigma_s + \sigma_a$ everywhere. Typical values depend on the scene's maximum density.
*   **Power Heuristic Exponent**: Typically 2.

### Edge Cases and Pitfalls
*   **Redundant Computation**: Paths share the same vertices for all of $\bar{p}_{n-1}$, so transmittance factors and PDFs share common factors. Implementations must carefully cancel or compute these to avoid redundancy and precision loss [S1].
*   **Precision**: Ratio tracking and delta tracking involve products of transmittances which can underflow. Use log-space arithmetic if necessary.
*   **Boundary Conditions**: Ensure $\sigma_{maj}$ is correctly computed at boundaries of heterogeneous media.

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
