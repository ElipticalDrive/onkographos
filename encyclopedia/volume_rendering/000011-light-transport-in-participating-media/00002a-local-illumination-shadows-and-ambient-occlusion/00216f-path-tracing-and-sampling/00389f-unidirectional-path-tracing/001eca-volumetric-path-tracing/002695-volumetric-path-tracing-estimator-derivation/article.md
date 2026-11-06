# Volumetric Path Tracing Estimator Derivation

Also called: Generalized Path Contribution Estimator, Monte Carlo Estimator for Volumetric Path Tracing

## Overview
This article derives the simplified Monte Carlo estimator for the generalized path contribution function $\hat{P}$ in volumetric path tracing. It solves the problem of expressing the radiance contribution of a sampled path in terms of the specific sampling probabilities used during generation (distance, event type, and direction). It sets itself apart from its siblings by providing the foundational algebraic simplification that reduces the raw path integral estimator to a product of phase function ratios and emission, assuming delta tracking with a single majorant. This derivation underpins the basic volumetric path tracer described in [Volumetric Path Tracing](1863-volumetric-path-tracing.md).

## How it works
The derivation starts with the general Monte Carlo estimator for a path $\bar{p}_n = (p_0, \dots, p_n)$ with $m$ real-scattering vertices. It simplifies this estimator through three sequential cancellations corresponding to the three sampling operations: distance sampling, event type selection, and direction sampling.

### Inputs and Outputs
*   **Input**: A sampled path $\bar{p}_n$ of $n+1$ vertices ($p_0$ to $p_n$). $p_0$ is the camera position. $p_n$ is the final vertex where absorption occurs. The path contains $m$ real-scattering vertices ($r_1, \dots, r_{m-1}$) and potentially null-scattering vertices.
*   **Output**: The estimated path contribution $\hat{P}(\bar{p}_n)$.

### Data Structures
No complex data structures are required for the estimator itself. The derivation relies on scalar values associated with each vertex $p_i$:
*   $\sigma_{maj}$: The single majorant extinction coefficient used for delta tracking.
*   $\sigma_{\{a,s,n\}}(p_i)$: The extinction coefficient for absorption ($a$), real scattering ($s$), or null scattering ($n$) at vertex $p_i$.
*   $\hat{f}(p_{i+1} \to p_i \to p_{i-1})$: The generalized phase function (or scattering density) at vertex $p_i$, accounting for the incoming direction from $p_{i-1}$ and outgoing direction to $p_{i+1}$.
*   $\hat{L}_e(p_n \to p_{n-1})$: The emitted radiance at the final vertex $p_n$ towards $p_{n-1}$.
*   $p_\omega(\omega_{i+1}|r_i, \omega_i)$: The probability density of sampling direction $\omega_{i+1}$ at real-scattering vertex $r_i$ given incident direction $\omega_i$.

### Numbered Steps

1.  **Initial Estimator Formulation**:
    The general estimator for the path contribution is defined as [S1]:
    $$
    \hat{P}(\bar{p}_n) = \frac{\hat{T}(\bar{p}_n) \hat{L}_e(p_n \to p_{n-1})}{\left(\prod_{i=0}^{n-1} p_{maj}(p_{i+1}|p_i, \omega_i)\right) \left(\prod_{i=1}^{n} p_e(p_i)\right) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i) \hat{G}(r_i \leftrightarrow r_{i+1})\right)}
    $$
    Where:
    *   $\omega_i$ is the direction from $p_i$ to $p_{i+1}$.
    *   $\hat{T}(\bar{p}_n)$ is the transmittance along the path.
    *   $p_{maj}(p_{i+1}|p_i, \omega_i)$ is the probability density of sampling point $p_{i+1}$ along direction $\omega_i$ from $p_i$.
    *   $p_e(p_i)$ is the discrete probability of sampling the scattering event type (absorption, real-scattering, or null-scattering) at $p_i$.
    *   $p_\omega(\omega_{i+1}|r_i, \omega_i)$ is the probability of sampling direction $\omega_{i+1}$ after a regular scattering event at $r_i$.
    *   $\hat{G}(r_i \leftrightarrow r_{i+1})$ accounts for the change of variables from solid angle to path space measure.

2.  **Distance Sampling Simplification**:
    Assume a single majorant $\sigma_{maj}$ is used for delta tracking. The density for sampling a distance $t$ is $p_{maj} = \sigma_{maj} e^{-\sigma_{maj} t}$ [S1].
    The transmittance $\hat{T}$ contains exponential factors $e^{-\sigma_{maj} t}$ that cancel with the exponential parts of $p_{maj}$ [S1].
    Each $p_{maj}$ term leaves a factor of $1/\sigma_{maj}$ in the denominator. There are $n$ such terms (for $i=0$ to $n-1$), resulting in a $(\sigma_{maj})^n$ factor in the denominator [S1].
    The $\hat{G}$ factors also cancel out during the expansion of $\hat{T}$ and the change of variables [S1].
    The estimator becomes [S1]:
    $$
    \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1})\right) \hat{L}_e(p_n \to p_{n-1})}{\left(\sigma_{maj}\right)^n \left(\prod_{i=1}^{n} p_e(p_i)\right) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)}
    $$

3.  **Event Type Simplification**:
    The probabilities $p_e(p_i)$ are defined as the ratio of the specific extinction coefficient to the majorant: $p_e(p_i) = \sigma_{\{a,s,n\}}(p_i) / \sigma_{maj}$ [S1].
    The product $\prod_{i=1}^{n} p_e(p_i)$ contains $n$ factors of $1/\sigma_{maj}$. This cancels the $(\sigma_{maj})^n$ term in the denominator [S1].
    The estimator simplifies to [S1]:
    $$
    \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1})\right) \hat{L}_e(p_n \to p_{n-1})}{\left(\prod_{i=1}^{n} \sigma_{\{a,s,n\}}(p_i)\right) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)}
    $$

4.  **Specific Event Constraints**:
    By the sampling strategy of the basic volumetric path tracer, the first $n-1$ vertices are either real-scattering or null-scattering events. The final vertex $p_n$ is an absorption event [S1].
    Therefore, the product in the denominator splits:
    *   For $i=1$ to $n-1$, $\sigma_{\{a,s,n\}}(p_i)$ is either $\sigma_s(p_i)$ or $\sigma_n(p_i)$.
    *   For $i=n$, $\sigma_{\{a,s,n\}}(p_n)$ is $\sigma_a(p_n)$.
    The estimator becomes [S1]:
    $$
    \hat{P}(\bar{p}_n) = \frac{\left(\prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1})\right) \hat{L}_e(p_n \to p_{n-1})}{\left(\prod_{i=1}^{n-1} \sigma_{\{s,n\}}(p_i)\right) \sigma_a(p_n) \left(\prod_{i=1}^{m-1} p_\omega(\omega_{i+1}|r_i, \omega_i)\right)}
    $$

5.  **Phase Function and Null Scattering Cancellation**:
    Neglecting surface scattering, $\hat{f}$ represents either regular volumetric scattering or null scattering [S1].
    *   For real scattering, $\hat{f}$ includes a factor of $\sigma_s$.
    *   For null scattering, $\hat{f}$ includes a factor of $\sigma_n$ and a Heaviside function which is always 1 given the sampling method [S1].
    These $\sigma_s$ and $\sigma_n$ factors in the numerator cancel the corresponding $\sigma_{\{s,n\}}$ factors in the denominator product $\prod_{i=1}^{n-1} \sigma_{\{s,n\}}(p_i)$ [S1].
    We restrict the remaining product in the numerator to the $m-1$ real-scattering vertices $r_i$ [S1].
    The estimator simplifies to [S1]:
    $$
    \hat{P}(\bar{p}_n) = \left(\prod_{i=1}^{m-1} \frac{p(r_{i-1} \to r_i \to r_{i+1})}{p_\omega(\omega_{i+1}|r_i, \omega_i)}\right) \frac{\hat{L}_e(p_n \to p_{n-1})}{\sigma_a(p_n)}
    $$
    Here, $p(r_{i-1} \to r_i \to r_{i+1})$ denotes the phase function value for the real scattering event at $r_i$.

6.  **Final Emission Cancellation**:
    The path space emission function $\hat{L}_e$ (Equation 14.10 in [S1]) contains a factor of $\sigma_a(p_n)$ [S1].
    This $\sigma_a(p_n)$ cancels with the $\sigma_a(p_n)$ in the denominator [S1].
    The final estimator is [S1]:
    $$
    \hat{P}(\bar{p}_n) = \left(\prod_{i=1}^{m-1} \frac{p(r_{i-1} \to r_i \to r_{i+1})}{p_\omega(\omega_{i+1}|r_i, \omega_i)}\right) \hat{L}_e(p_n \to p_{n-1})
    $$
    The value is the emission at the last vertex scaled by the product of ratios of phase function values to sampling probabilities for all real-scattering events.

### Pseudocode
The following Python-like pseudocode computes the estimator for a single path. It assumes the path has already been sampled and the necessary values are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_path_contribution(path_vertices, real_scattering_indices, phase_functions, sampling_probs, emission_at_end):
    """
    Computes the simplified volumetric path contribution estimator.
    
    Args:
        path_vertices: List of vertex objects. path_vertices[n] is the final absorption vertex.
        real_scattering_indices: List of indices in path_vertices corresponding to real scattering events (r_1 to r_{m-1}).
        phase_functions: List of phase function values p(r_{i-1} -> r_i -> r_{i+1}) for each real scattering vertex.
        sampling_probs: List of direction sampling probabilities p_omega(omega_{i+1}|r_i, omega_i) for each real scattering vertex.
        emission_at_end: The emitted radiance L_e(p_n -> p_{n-1}) at the final vertex.
        
    Returns:
        The estimated path contribution P_hat.
    """
    contribution = emission_at_end
    
    # Iterate over all real-scattering vertices (excluding the first and last in the list of real scatterers if they are boundaries,
    # but based on the formula, we iterate i from 1 to m-1. 
    # In the context of the path, these are the internal real scattering vertices.
    # The formula uses product from i=1 to m-1.
    
    for i in range(len(real_scattering_indices)):
        # Get the phase function value and sampling probability for this real scattering event
        p_val = phase_functions[i]
        prob_val = sampling_probs[i]
        
        # Avoid division by zero
        if prob_val == 0:
            return 0.0
            
        # Multiply by the ratio of phase function to sampling probability
        contribution *= (p_val / prob_val)
        
    return contribution
```

### Parameters and Edge Cases
*   **Majorant**: The derivation assumes a single constant majorant $\sigma_{maj}$ [S1]. If spatially varying majorants are used, the cancellation of exponential terms is more complex and not covered by this specific simplified estimator [S1].
*   **Null Scattering**: The Heaviside function for null scattering is assumed to be 1 [S1]. This holds because vertices are sampled with path tracing, ensuring valid null scattering events.
*   **Surface Scattering**: The derivation explicitly neglects surface scattering [S1]. If surface scattering is present, additional terms and probabilities must be included.
*   **Zero Probability**: If any direction sampling probability $p_\omega$ is zero, the estimator is undefined (or zero contribution if the path was not sampled with non-zero probability).

## Operating characteristics
*   **Complexity**: The estimator computation is $O(m)$, where $m$ is the number of real-scattering vertices in the path.
*   **Strengths**: The simplified form is computationally efficient and numerically stable, as it avoids large transmittance values and small probabilities by canceling them out analytically. It directly relates the path contribution to the phase function and sampling strategy.
*   **Limitations**: It is specific to the basic volumetric path tracing algorithm using delta tracking with a single majorant and no explicit light sampling. It does not account for surface scattering or multiple importance sampling with other strategies.
*   **When to use**: Use this estimator when implementing the basic volumetric path tracer for debugging or ground-truth reference, as described in [Volumetric Path Tracing](1863-volumetric-path-tracing.md).

## Background
This derivation is presented in the context of the path integral formulation of light transport for participating media. It is based on the work described in [S1], which details the Monte Carlo estimator for volumetric path tracing. The simplification relies on the properties of delta tracking and the specific sampling strategy of the basic volumetric path tracer.

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
