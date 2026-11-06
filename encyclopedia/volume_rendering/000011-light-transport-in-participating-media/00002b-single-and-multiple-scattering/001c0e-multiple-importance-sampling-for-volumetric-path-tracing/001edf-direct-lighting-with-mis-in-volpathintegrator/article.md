# Direct Lighting with MIS in VolPathIntegrator

Also called: Direct Lighting with MIS

## Overview
Direct Lighting with MIS in VolPathIntegrator is a method to sample direct illumination in volumetric media by combining unidirectional path sampling and light path sampling using Multiple Importance Sampling (MIS) [S1]. It solves the high variance problem in spectral volumetric path tracing by allowing the estimator to choose between sampling the phase function to find the next vertex or sampling the light source to connect directly, weighting the contributions via the power heuristic [S1]. Unlike standard spectral MIS which may suffer from high variance in heterogeneous media, this method specifically handles the transmittance estimation via ratio tracking for the light connection segment, reducing Mean Squared Error (MSE) significantly compared to non-MIS approaches [S1].

## How it works
The method estimates direct illumination at a path vertex $p_{n-1}$ by generating two candidate paths: one via unidirectional sampling and one via light path sampling. Both paths share the same geometry up to $p_{n-1}$, allowing for efficient computation and cancellation of common factors in the MIS weights.

### Inputs and Outputs
*   **Input**: The current path $\bar{p}_{n-1}$ ending at vertex $p_{n-1}$, the scattering coefficient $\sigma_s(p_{n-1})$, the null scattering coefficient $\sigma_n(p_{n-1})$, the majorant coefficient $\sigma_{maj}$, and the phase function $p_\omega$.
*   **Output**: The estimated direct lighting contribution, computed as the sum of the weighted contributions from the unidirectional path $\bar{p}_n$ and the light path $\bar{p}'_n$.

### Data Structures
The implementation requires storing the path vertices and the accumulated throughput. Crucially, because the paths share vertices up to $p_{n-1}$, the transmittance factors and PDFs share common factors. The algorithm must compute these shared components once to avoid redundancy and precision loss [S1].

### Steps
1.  **Compute Shared Path State**: Calculate the path throughput weight $\beta$ and the path PDF $p_u(\bar{p}_{n-1})$ for the existing path $\bar{p}_{n-1}$. These are identical for both sampling strategies up to this vertex [S1].
2.  **Unidirectional Path Sampling**:
    *   Use delta tracking to find the next real-scattering vertex $p_n$ from $p_{n-1}$.
    *   Sample the phase function $p_\omega(\omega_n)$ to determine the direction.
    *   Compute the probability of the null-scattering segment $p_{null}(p_{n-1}, p_n)$ using the product of null scattering probabilities and majorant transmittances [S1].
    *   Compute the full unidirectional path PDF $p_u(\bar{p}_n)$ [S1].
3.  **Light Path Sampling**:
    *   Sample a point on a light source to define the direction $\omega_n$ and the light vertex $p_n$ (denoted $p'_n$ in the light path context).
    *   Use ratio tracking to estimate the transmittance along the shadow ray from $p_{n-1}$ to $p_n$. This involves accumulating the product of $\sigma_n / \sigma_{maj}$ factors [S1].
    *   Compute the probability of the ratio tracking segment $p_{ratio}(p_{n-1}, p_n)$ [S1].
    *   Compute the full light path PDF $p_l(\bar{p}'_n)$ [S1].
4.  **Evaluate MIS Weights**:
    *   The VolPathIntegrator samples both types of paths according to the first wavelength $\lambda_1$ but evaluates the PDFs at all wavelengths to enable spectral MIS [S1].
    *   Compute the MIS weights $w_u(\bar{p}_n)$ and $w_l(\bar{p}'_n)$ using the power heuristic based on the evaluated PDFs $p_{u, \lambda_1}$ and $p_{l, \lambda_1}$ [S1].
5.  **Combine Contributions**:
    *   Sum the weighted contributions of both paths. The contribution of a path is the product of the weight, the estimated transmittance $\hat{T}$, and the emitted radiance $L_e$, divided by the PDF [S1].

### Formulas
The probability of a null-scattering vertex at $p_{i+k}$ is:
$$ p_e(p_{i+k}) = \frac{\sigma_n(p_{i+k})}{\sigma_{maj}} $$

The majorant PDF for a segment is:
$$ p_{maj}(p_{i+k-1} \rightarrow p_{i+k}) = \sigma_{maj} T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) $$

The probability of a path segment with $m$ intermediate null-scattering vertices is:
$$ p_{null}(p_i, p_j) = \left( \prod_{k=1}^{m} \sigma_n(p_{i+k}) T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) \right) T_{maj}(p_{i+m} \rightarrow p_j) $$

The full unidirectional path PDF is:
$$ p_u(\bar{p}_n) = p_u(\bar{p}_{n-1}) \frac{\sigma_s(p_{n-1})}{\sigma_{maj}} p_\omega(\omega_n) p_{null}(p_{n-1}, p_n) $$

The probability of the ratio tracking segment is:
$$ p_{ratio}(p_i, p_j) = \left( \prod_{k=1}^{m} T_{maj}(p_{i+k-1} \rightarrow p_{i+k}) \sigma_{maj} \right) T_{maj}(p_{i+m} \rightarrow p_j) $$

The full light path PDF is:
$$ p_l(\bar{p}_n) = p_u(\bar{p}_{n-1}) \frac{\sigma_s(p_{n-1})}{\sigma_{maj}} p_{l,\omega}(\omega_n) p_{ratio}(p_{n-1}, p_n) $$

The final MIS estimator combines the two samples:
$$ w_u(\bar{p}_n) \frac{\hat{T}(\bar{p}_n) L_e(p_n \rightarrow p_{n-1})}{p_{u, \lambda_1}(\bar{p}_n)} + w_l(\bar{p}'_n) \frac{\hat{T}(\bar{p}'_n) L_e(p' \rightarrow p'_{n-1})}{p_{l, \lambda_1}(\bar{p}'_n)} $$

### Parameters
*   $\sigma_{maj}$: The majorant coefficient, used for delta tracking and ratio tracking. Not given in the sources.
*   $\lambda_1$: The first wavelength used for sampling the path type. Not given in the sources.

### Edge Cases and Pitfalls
*   **Redundancy**: Because the paths share the same vertices for all of $\bar{p}_{n-1}$, the transmittance factors and PDFs share common factors. The implementation must carefully cancel or compute these common factors to avoid redundancy and numerical instability [S1].
*   **Spectral Evaluation**: The PDFs must be evaluated at all wavelengths, not just the sampling wavelength $\lambda_1$, to enable effective spectral MIS [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_lighting_mis(path_p_n_minus_1, sigma_s, sigma_n, sigma_maj, phase_func, light_sampler):
    # 1. Compute shared path state
    # p_u(p_{n-1}) and beta are already computed from previous steps
    p_u_prev = path_p_n_minus_1.pdf
    beta_prev = path_p_n_minus_1.throughput
    
    # 2. Unidirectional Path Sampling
    # Sample next vertex using delta tracking
    p_n, omega_n = delta_tracking_sample(path_p_n_minus_1.position, sigma_n, sigma_maj)
    
    # Compute null scattering probability for the segment
    # p_null(p_{n-1}, p_n) involves product of sigma_n and T_maj
    p_null = compute_null_scattering_prob(path_p_n_minus_1.position, p_n, sigma_n, sigma_maj)
    
    # Compute unidirectional PDF
    # p_u(p_n) = p_u(p_{n-1}) * (sigma_s / sigma_maj) * p_omega(omega_n) * p_null
    p_u = p_u_prev * (sigma_s(path_p_n_minus_1.position) / sigma_maj) * \
          phase_func(omega_n) * p_null
    
    # Estimate transmittance and radiance for unidirectional path
    T_hat_u = estimate_transmittance(path_p_n_minus_1.position, p_n, sigma_n, sigma_maj)
    L_e_u = light_sampler.eval_radiance(p_n, path_p_n_minus_1.position)
    
    # 3. Light Path Sampling
    # Sample light source
    p_prime_n, omega_prime_n = light_sampler.sample(path_p_n_minus_1.position)
    
    # Compute ratio tracking probability for the segment
    # p_ratio(p_{n-1}, p_n) involves product of T_maj and sigma_maj
    p_ratio = compute_ratio_tracking_prob(path_p_n_minus_1.position, p_prime_n, sigma_maj)
    
    # Compute light path PDF
    # p_l(p'_n) = p_u(p_{n-1}) * (sigma_s / sigma_maj) * p_{l,omega}(omega'_n) * p_ratio
    # Note: p_{l,omega} is the PDF of the light sampling direction
    p_l = p_u_prev * (sigma_s(path_p_n_minus_1.position) / sigma_maj) * \
          light_sampler.direction_pdf(omega_prime_n) * p_ratio
    
    # Estimate transmittance and radiance for light path
    T_hat_l = estimate_transmittance(path_p_n_minus_1.position, p_prime_n, sigma_n, sigma_maj)
    L_e_l = light_sampler.eval_radiance(p_prime_n, path_p_n_minus_1.position)
    
    # 4. Evaluate MIS Weights
    # Evaluate PDFs at all wavelengths for spectral MIS
    # w_u and w_l are computed using the power heuristic
    w_u = compute_mis_weight(p_u, p_l)
    w_l = compute_mis_weight(p_l, p_u)
    
    # 5. Combine Contributions
    # Note: The formula uses p_{u, lambda_1} and p_{l, lambda_1}
    # In implementation, these are the PDFs evaluated at the sampling wavelength
    contribution_u = w_u * (T_hat_u * L_e_u) / p_u
    contribution_l = w_l * (T_hat_l * L_e_l) / p_l
    
    return contribution_u + contribution_l

def compute_null_scattering_prob(p_i, p_j, sigma_n, sigma_maj):
    # Simulate delta tracking steps to find intermediate null vertices
    # This is a simplified representation; actual implementation requires
    # stepping through the medium
    current_pos = p_i
    prob = 1.0
    while current_pos < p_j:
        # Sample distance
        dist = sample_distance(current_pos, p_j, sigma_maj)
        current_pos += dist
        
        # Check if real scattering occurred (simplified)
        # If null, multiply by sigma_n / sigma_maj and T_maj
        # If real, break loop (handled by outer function)
        prob *= (sigma_n(current_pos) / sigma_maj) * exp(-sigma_maj * dist)
        
        if current_pos >= p_j:
            break
            
    # Final segment to p_j
    dist = p_j - current_pos
    prob *= exp(-sigma_maj * dist)
    return prob

def compute_ratio_tracking_prob(p_i, p_j, sigma_maj):
    # Ratio tracking accumulates product of T_maj and sigma_maj
    # This is a simplified representation
    current_pos = p_i
    prob = 1.0
    while current_pos < p_j:
        dist = sample_distance(current_pos, p_j, sigma_maj)
        current_pos += dist
        prob *= exp(-sigma_maj * dist) * sigma_maj
        
    # Final segment
    dist = p_j - current_pos
    prob *= exp(-sigma_maj * dist)
    return prob

def estimate_transmittance(p_i, p_j, sigma_n, sigma_maj):
    # Estimate transmittance using ratio tracking
    # Accumulate product of sigma_n / sigma_maj
    current_pos = p_i
    T = 1.0
    while current_pos < p_j:
        dist = sample_distance(current_pos, p_j, sigma_maj)
        current_pos += dist
        T *= (sigma_n(current_pos) / sigma_maj) * exp(-sigma_maj * dist)
        
    dist = p_j - current_pos
    T *= exp(-sigma_maj * dist)
    return T
```

## Subtopics

### [Spectral+NEE MIS (Shadow Connection Routine)](1907-spectral-nee-mis-shadow-connection-routine.md)
Spectral+NEE MIS is a shadow connection routine used in volumetric path tracing to estimate direct illumination in chromatic participating media. It reduces variance by combining unidirectional path sampling and next-event estimation (NEE) using Multiple Importance Sampling (MIS) across spectral samples. Unlike standard spectral MIS which may suffer from high variance in heterogeneous media, this method uses ratio tracking to handle null scattering events, allowing robust connections through dense or thin media while maintaining color fidelity.

### [Spectral MIS with Unidirectional Path Tracing](1964-spectral-mis-with-unidirectional-path-tracing.md)
Spectral MIS with Unidirectional Path Tracing is a method for estimating direct illumination in spectrally varying participating media by combining multiple per-channel sampling techniques via Multiple Importance Sampling (MIS). It addresses the high variance and computational cost of standard spectral tracking, which requires a conservative majorant for all color components, by allowing each color channel to be sampled independently while combining the results. Unlike [Spectral+NEE MIS (Shadow Connection Routine)](1907-spectral-nee-mis-shadow-connection-routine.md), which adds a shadow connection routine for next-event estimation, this method focuses on the core path construction and MIS weighting for unidirectional paths.

## Operating characteristics
*   **Time Complexity**: The method requires two path sampling passes (unidirectional and light path) per vertex, doubling the sampling cost for direct lighting compared to a single strategy. However, the variance reduction often allows for fewer samples to achieve the same image quality.
*   **Memory Complexity**: Requires storing the path vertices and intermediate transmittance/PDF values. The shared path state up to $p_{n-1}$ allows for some memory reuse.
*   **Measured Performance**: In the scene from [S1], MSE is reduced by a factor of 149 with spectral MIS compared to non-spectral MIS.
*   **Strengths**: Significantly reduces variance in heterogeneous media and spectral rendering. Handles the complexity of ratio tracking and delta tracking integration.
*   **Limitations**: Increased computational cost per sample due to dual sampling. Requires careful implementation to handle shared factors and spectral evaluation.
*   **When to Use**: When high-quality spectral volumetric rendering is required, especially in heterogeneous media where single-strategy sampling leads to high variance.
*   **When Not to Use**: When computational resources are extremely limited and lower quality is acceptable, or in homogeneous media where single-strategy sampling might be sufficient.

## Background
This method is described in "14.2 Volume Scattering Integrators" from the Physically Based Rendering (pbr-book.org) 4th edition [S1]. It is part of the VolPathIntegrator, which extends the PathIntegrator to handle participating media. The use of MIS in volumetric path tracing is a key advancement to handle the complexity of light transport in media, where both the phase function and light sources contribute to the sampling distribution.

## Sources
[S1] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
