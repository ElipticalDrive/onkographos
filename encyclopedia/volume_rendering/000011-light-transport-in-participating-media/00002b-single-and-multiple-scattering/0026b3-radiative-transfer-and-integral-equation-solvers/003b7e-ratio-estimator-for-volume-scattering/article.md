# Ratio Estimator for Volume Scattering

Also called: Ratio Estimator for Single Scattering, Ratio term estimator, Single-Scattering Ratio Estimator, Unoccluded/Occluded ratio estimator

## Overview
The Ratio Estimator is a Monte Carlo integration technique for volumetric light transport that separates the radiance integral into an analytically computed unoccluded term and a numerically estimated ratio of occluded to unoccluded radiance [S1]. It solves the problem of accurate shadowing in participating media by allowing existing sampling schemes (such as distance or equidistant sampling) to estimate visibility without bias, asymptotically converging to the correct solution even in multiple scattering scenarios [S1]. Unlike approximate moment-based estimators, this method provides an unbiased estimate by explicitly canceling the transmittance component through the ratio of two Monte Carlo estimates [S1].

## How it works
The method computes the single-scattering radiance contribution by splitting the integral into two distinct parts: an analytic term for the unoccluded medium and a numerical ratio term for the occlusion effects [S1].

**Inputs and Outputs**
*   **Inputs:**
    *   $\omega_0$: Camera ray direction.
    *   $\omega_i$: Light direction.
    *   $L_i$: Incident light radiance (assumed constant or sampled).
    *   $L_{\text{s}}$: Surface radiance at the interaction point.
    *   $\sigma_a$: Absorption coefficient.
    *   $\sigma_s$: Scattering coefficient.
    *   $\sigma_t = \sigma_a + \sigma_s$: Extinction coefficient.
    *   $f(\omega_i, \omega_0)$: Phase function.
    *   $t_S$: Start distance of the medium segment along the ray.
    *   $t_{SC}$: Distance to the closest surface intersection or medium exit.
    *   $N_{\text{sample}}$: Number of samples for the numerical integration.
    *   $S(x)$: Importance sampling function transforming uniform samples to the target distribution.
    *   $\text{PDF}(t')$: Probability density function of the sample distance.
    *   $V(t', \omega_i)$: Binary visibility function (1 if unoccluded, 0 if occluded).
*   **Outputs:**
    *   $L(\omega_i, \omega_0)$: Estimated outgoing radiance.

**Data Structures**
The method relies on standard floating-point accumulators for the numerator and denominator of the ratio estimator. No complex data structures are required beyond the ray parameters and sample values [G].

**Steps**
1.  **Compute Unoccluded Term:** Calculate the analytic unoccluded single-scattering integral $\mathcal{R}_{\text{box}}$ for the box section of the medium, representing the integral over transmittance with light source radiance separated out [S1].
2.  **Generate Samples:** Generate $N_{\text{sample}}$ sample distances $t'$ using the importance sampling function $S(x)$, which transforms uniform random variables $\xi$ into the target distribution via $t' = S(\xi \cdot \frac{k}{N_{\text{sample}}-1})$ [S1].
3.  **Compute Ratio Numerator:** For each sample $k$, compute the weighted visibility. The numerator sum is:
    $$ \text{Num} = \sum_{k=0}^{N_{\text{sample}}} \frac{e^{-\sigma_t(t' + d_m(t'))} V(t', \omega_i)}{\text{PDF}(t')} $$
    where $d_m(t')$ is the distance to the medium boundary from the sample point, and $V(t', \omega_i)$ is the binary visibility function [S1].
4.  **Compute Ratio Denominator:** For each sample $k$, compute the weight ignoring visibility. The denominator sum is:
    $$ \text{Den} = \sum_{k=0}^{N_{\text{sample}}} \frac{e^{-\sigma_t(t' + d_m(t'))}}{\text{PDF}(t')} $$
    This term represents the unoccluded estimate used for normalization [S1].
5.  **Calculate Visibility Ratio:** Compute the estimated visibility ratio $\hat{V}$:
    $$ \hat{V}(\omega_i, \omega_0) = \frac{\text{Num}}{\text{Den}} $$
    If the section is completely unoccluded, $\hat{V}$ converges to 1 with a single sample [S1].
6.  **Combine Terms:** Compute the final radiance by multiplying the analytic unoccluded term by the visibility ratio and the light radiance:
    $$ L(\omega_i, \omega_0) = L_{\text{s}}(\omega_i, \omega_0) e^{-\sigma_t t_{\text{SC}}} + \sigma_s f(\omega_i, \omega_0) \mathcal{R}_{\text{box}}(\omega_i, \omega_0, t_{\text{S}}) \hat{V}(\omega_i, \omega_0) L_i $$
    where $L_{\text{s}}$ is surface radiance [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ratio_estimator_volume_scattering(
    omega_0, omega_i, L_i, L_s, sigma_a, sigma_s, f_phase, 
    t_S, t_SC, N_sample, S_func, pdf_func, visibility_func
):
    """
    Computes single-scattering radiance using the Ratio Estimator.
    
    Args:
        omega_0: Camera ray direction.
        omega_i: Light direction.
        L_i: Incident light radiance.
        L_s: Surface radiance at the interaction point.
        sigma_a: Absorption coefficient.
        sigma_s: Scattering coefficient.
        f_phase: Phase function f(omega_i, omega_0).
        t_S: Start distance of the medium segment.
        t_SC: Distance to closest surface or medium exit.
        N_sample: Number of samples for ratio estimation.
        S_func: Importance sampling function S(x) returning distance t'.
        pdf_func: Probability density function PDF(t').
        visibility_func: Function V(t_prime, omega_i) returning 1.0 or 0.0.
        
    Returns:
        L: Estimated radiance.
    """
    sigma_t = sigma_a + sigma_s
    
    # Step 1: Compute analytic unoccluded term R_box
    # The exact analytic formula for R_box depends on the medium geometry (e.g., box intersection).
    # Not given in the sources.
    R_box = 0.0
    
    # Step 2 & 3 & 4: Compute Numerator and Denominator of the ratio
    numerator = 0.0
    denominator = 0.0
    
    # Guard against division by zero if N_sample is 1 [G]
    denom_idx = N_sample - 1 if N_sample > 1 else 1
    
    for k in range(N_sample):
        # Generate sample distance t' from uniform distribution
        t_prime = S_func(k / denom_idx)
        
        # Calculate distance to medium boundary from sample point [G]
        d_m = max(0.0, t_SC - t_prime)
        
        # Calculate transmittance from sample point to boundary
        transmittance = math.exp(-sigma_t * (t_prime + d_m))
        
        # Get PDF value for importance sampling
        p_val = pdf_func(t_prime)
        
        # Avoid division by zero
        if p_val == 0:
            continue
            
        weight = transmittance / p_val
        
        # Step 3: Add to numerator with visibility
        V = visibility_func(t_prime, omega_i)
        numerator += weight * V
        
        # Step 4: Add to denominator (ignoring visibility)
        denominator += weight
        
    # Step 5: Calculate visibility ratio
    if denominator == 0:
        V_hat = 0.0
    else:
        V_hat = numerator / denominator
        
    # Step 6: Combine terms
    # Surface radiance term (unoccluded surface contribution)
    L_surface = L_s * math.exp(-sigma_t * t_SC)
    
    # Volumetric scattering term
    L_volume = sigma_s * f_phase * R_box * V_hat * L_i
    
    return L_surface + L_volume
```

**Parameters**
*   $N_{\text{sample}}$: The number of samples used for the numerical integration of the ratio term. Higher values reduce variance but increase computational cost [S1].

**Edge Cases and Pitfalls**
*   **Progressive Rendering Bias:** In progressive rendering, it is critical to accumulate three separate surfaces: the estimated unoccluded term, the analytic unoccluded term, and the occluded term. Failure to do so causes the denominator in the estimator to not cancel properly, leading to bias [S1].
*   **Unoccluded Convergence:** If the medium section is completely unoccluded, the estimator converges to the correct solution with a single sample because the visibility ratio $\hat{V}$ equals 1 [S1].
*   **Multiple Scattering:** While derived for single scattering, the method generalizes to multiple scattering by evaluating both unoccluded and occluded light transport components stochastically after each scattering event [S1].

## Subtopics

### Moment-Based Approximate Estimator
An approximate estimator for single scattering that uses a heuristic formula based on projection on moments to estimate visibility [S1]. It computes the unoccluded single-scattering term analytically and then applies the heuristic to estimate the ratio between unoccluded and occluded radiance [S1]. Unlike the sampling-based Ratio Estimator, which asymptotically converges to the correct solution, the Moment-Based Approximate Estimator provides an approximate solution [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $N_{\text{sample}}$ for the ratio estimation part. The analytic term is computed in constant time relative to sampling [G].
*   **Memory Complexity:** Constant, requiring only accumulators for the numerator and denominator [G].
*   **Strengths:**
    *   Asymptotically converges to the correct solution, even in multiple scattering cases [S1].
    *   Can be easily combined with existing sampling schemes such as equidistant and distance sampling algorithms [S1].
    *   Separates the complex occlusion problem from the analytic integration, allowing reuse of unoccluded integrators [S1].
*   **Limitations:**
    *   Requires careful implementation in progressive rendering to avoid bias [S1].
    *   Performance depends on the efficiency of the visibility function and the importance sampling distribution [S1].
*   **When to Use:** Use when unbiased single-scattering results are required in participating media, especially when integrating with existing Monte Carlo path tracers [G].
*   **When Not to Use:** Use the Moment-Based Approximate Estimator if performance is critical and approximate results are acceptable, or if the scene is static and precomputed [G].

## Background
The Ratio Estimator for Volume Scattering was introduced in the context of collimated whole volume light scattering in homogeneous finite media [S1]. It builds upon previous approaches that used ratio estimators for shadowing BRDFs or approximate filtering techniques based on Fourier coefficients and moments [S1]. The work demonstrates that the ratio estimator, previously considered an approximation, actually asymptotically converges to the correct solution of the integral [S1]. It generalizes the technique to multiple scattering and progressive rendering, providing a robust framework for volumetric light transport [S1].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
