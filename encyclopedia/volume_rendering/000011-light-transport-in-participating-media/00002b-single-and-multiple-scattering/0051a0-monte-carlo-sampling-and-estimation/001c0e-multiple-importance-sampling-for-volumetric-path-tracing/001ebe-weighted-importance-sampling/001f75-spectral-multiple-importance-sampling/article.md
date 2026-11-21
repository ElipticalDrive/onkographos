# Spectral Multiple Importance Sampling

Also called: Chromatic Media Sampling with MIS, Single-sample MIS for Wavelengths, Spectral Volume Sampling

## Overview
Spectral Multiple Importance Sampling (MIS) is a technique for handling spectrally varying participating media in path tracing by sampling a single wavelength channel while correctly weighting contributions across all spectral bands [S1, unofficial][S2]. It solves the high-variance problem that arises when a single sampling probability density function (PDF), derived from one wavelength's medium properties, is used to evaluate integrands at other wavelengths where that PDF may be a poor match or zero-valued [S2].

## How it works
**Inputs:**
*   Current path vertex position $x$.
*   Throughput vector $\beta$ (RGB or $n$ spectral samples) [S1, unofficial][S2].
*   Medium properties: absorption coefficient $\sigma_a(\lambda)$ and scattering coefficient $\sigma_s(\lambda)$ for $n$ spectral samples (typically $n=3$ for RGB) [S1, unofficial][S2].
*   Majorant extinction coefficient $\sigma_{maj}$ (scalar or vector, depending on implementation context, but typically a scalar upper bound for the specific channel being sampled) [S2].
*   Random number generator state.

**Outputs:**
*   Sampled wavelength index $k \in \{0, 1, \dots, n-1\}$ [S1, unofficial].
*   PDF values $p_{\lambda_i}(x)$ for all $i \in \{0, \dots, n-1\}$ [S1, unofficial][S2].
*   Updated throughput and path termination status [S1, unofficial][S2].

**Data Structures:**
*   `throughput_albedo`: A vector of size $n$ (or 3 for RGB), where each element $i$ is $\beta_i \cdot \frac{\sigma_{s,\lambda_i}}{\sigma_{a,\lambda_i} + \sigma_{s,\lambda_i}}$ or simply $\beta_i \cdot \text{albedo}_i$ [S1, unofficial].
*   `DiscreteEmpiricalDistribution1D`: A discrete distribution object constructed from the `throughput_albedo` vector. This distribution maps the continuous uniform random variable to a channel index based on the relative magnitudes of the spectral contributions [S1, unofficial].
*   `pmf`: An array of size $n$ storing the probability density values $p_{\lambda_i}(x)$ for each wavelength [S1, unofficial][S2].

**Steps:**
1.  **Compute Spectral Weights:** Calculate the product of the current throughput and the single-scattering albedo for each wavelength channel. Let $w_i = \beta_i \cdot \frac{\sigma_{s,\lambda_i}}{\sigma_{a,\lambda_i} + \sigma_{s,\lambda_i}}$ [S1, unofficial].
2.  **Construct Discrete Distribution:** Create a discrete distribution object using the weights $w = [w_0, w_1, \dots, w_{n-1}]$ [S1, unofficial].
3.  **Sample Channel Index:** Draw a uniform random number $u \in [0, 1)$ and use it to sample an index $k$ from the discrete distribution [S1, unofficial].
4.  **Sample Distance:** Sample the distance $t$ to the next interaction using the majorant extinction coefficient $\sigma_{maj}$ associated with the sampled channel $\lambda_k$ (or a global majorant if used) [S2].
5.  **Evaluate PDFs:** For the sampled distance $t$ and position $x$, compute the PDF for the sampled channel $p_{\lambda_k}(x)$ and the PDFs for all other channels $p_{\lambda_i}(x)$ for $i \neq k$ [S1, unofficial][S2].
6.  **Compute MIS Weights:** Apply the balance heuristic for single-sample MIS [S2].
7.  **Update Throughput:** Multiply the throughput by the MIS weight and the appropriate spectral terms (albedo, phase function, etc.) for the sampled channel [S1, unofficial][S2].
8.  **Continue Path:** Proceed with the path tracing algorithm using the sampled distance and updated throughput [S1, unofficial].

**Formulas:**
*   Albedo: $\alpha_{\lambda_i} = \frac{\sigma_{s,\lambda_i}}{\sigma_{a,\lambda_i} + \sigma_{s,\lambda_i}}$ [S1, unofficial].
*   Discrete weights: $w_i = \beta_i \cdot \alpha_{\lambda_i}$ [S1, unofficial].
*   MIS weight (balance heuristic): $w_{\lambda_k}(x) = \frac{p_{\lambda_k}(x)}{\sum_{i=1}^{n} p_{\lambda_i}(x)}$ [S2].
*   Discrete probability: $q = 1/n$, where $n$ is the number of spectral samples [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def spectral_mis_sample_medium(x, throughput, sigma_a, sigma_s, sigma_maj, rng):
    """
    Sample a medium interaction using Spectral MIS.
    
    Args:
        x: Current position.
        throughput: Current path throughput (vector of size n).
        sigma_a: Absorption coefficients for n wavelengths (vector of size n).
        sigma_s: Scattering coefficients for n wavelengths (vector of size n).
        sigma_maj: Majorant extinction coefficient (scalar or vector).
        rng: Random number generator with uniform() method.
        
    Returns:
        t: Distance to next interaction.
        new_throughput: Updated throughput.
        terminated: Boolean indicating if path is terminated.
        pmf: PDFs for all wavelengths.
    """
    n = len(sigma_a)
    sigma_t = [sigma_a[i] + sigma_s[i] for i in range(n)]
    albedo = [sigma_s[i] / sigma_t[i] if sigma_t[i] > 0 else 0.0 for i in range(n)]
    weights = [throughput[i] * albedo[i] for i in range(n)]
    
    # Construct discrete distribution from weights
    cum_weights = [0.0] * n
    cum_weights[0] = weights[0]
    for i in range(1, n):
        cum_weights[i] = cum_weights[i-1] + weights[i]
    total_weight = cum_weights[-1]
    
    # Sample channel index k
    u = rng.uniform()
    k = 0
    while k < n - 1 and cum_weights[k] < u * total_weight:
        k += 1
        
    # Sample distance using majorant for sampled channel
    maj_k = sigma_maj[k] if isinstance(sigma_maj, (list, tuple)) else sigma_maj
    t = -math.log(1.0 - rng.uniform()) / maj_k
    
    # Evaluate PDFs for all channels
    pmf = [0.0] * n
    for i in range(n):
        maj_i = sigma_maj[i] if isinstance(sigma_maj, (list, tuple)) else sigma_maj
        pmf[i] = maj_i * math.exp(-maj_i * t)
        
    # Compute MIS weights (balance heuristic)
    sum_pmf = sum(pmf)
    mis_weights = [p / sum_pmf if sum_pmf > 0 else 0.0 for p in pmf]
    
    # Update throughput
    new_throughput = [throughput[i] * mis_weights[i] * albedo[i] * (sigma_t[i] / maj_k if maj_k > 0 else 0.0) for i in range(n)]
    
    terminated = False # Absorption check depends on event sampling, not detailed here
    return t, new_throughput, terminated, pmf
```

**Parameters:**
*   $n$: Number of spectral samples. Typical value: 3 (RGB) [S2].
*   $q$: Discrete probability of sampling a specific wavelength. Value: $1/n$ if uniform [S2].

**Edge Cases and Pitfalls:**
*   **Zero Extinction:** If $\sigma_{t,\lambda_i} = 0$, the albedo is undefined. Handle by setting albedo to 0 or 1 depending on whether the medium is purely scattering or empty [G].
*   **Infinite Variance:** If the majorant for one wavelength is much larger than the true extinction for another, the variance can be high. The scalar extinction constraint (see Subtopics) can mitigate this [S3].
*   **Precision:** Floating-point round-off errors in PDF calculations should be clamped to zero to avoid negative probabilities [S2].

## Subtopics

### [Chromatic Null-Scattering Free-Flight Sampling](6634-chromatic-null-scattering-free-flight-sampling.md)
This method extends null-scattering free-flight sampling to handle spectrally varying (chromatic) participating media by sampling a single color channel (R, G, or B) for distance determination while averaging the probability density functions (PDFs) across all channels to maintain unbiasedness. It solves the problem where a single sampling PDF based on one wavelength is a poor match or invalid for other wavelengths, causing high variance in spectral path tracing. It differs from standard monochromatic null-scattering by introducing channel selection logic and PDF averaging, and from full spectral sampling by reducing the per-step computational cost to a single channel's majorant evaluation.

### Scalar Extinction Constraint
A technique to avoid infinite variance when sampling transport equations for multiple wavelengths simultaneously by limiting the extinction coefficient to a single scalar value shared across all wavelengths [S3]. It enforces the constraint that for every pair of wavelengths $\lambda_1, \lambda_2$, the total extinction coefficients are equal: $\sigma_{a,\lambda_1}(x) + \sigma_{s,\lambda_1}(x) = \sigma_{a,\lambda_2}(x) + \sigma_{s,\lambda_2}(x) = \sigma_t(x)$ [S3]. This approach is suitable when extinction depends only on particle density/size (color varies via BSDF-like probabilities) [S3]. For atmospheric scattering, solving transport equations separately for each wavelength is preferred [S3]. If $\sigma_{t,\lambda_2} > \sigma_{t,\lambda_1}$ and sampling is done using density $p_{\lambda_2}$, the variance on wavelength $\lambda_1$ can be arbitrarily high or infinite [S3]. Specifically, variance is infinite if $\sigma_{t,\lambda_2} \geq 2\sigma_{t,\lambda_1}$ [S3].

## Operating characteristics
*   **Time Complexity:** $O(n)$ per interaction, where $n$ is the number of spectral samples [S2].
*   **Memory Complexity:** $O(n)$ to store the spectral coefficients and PDFs [S2].
*   **Performance:** Reduces variance compared to monochromatic sampling with a single majorant [S2].
*   **Strengths:** Unbiased, handles spectrally varying media effectively [S2].
*   **Limitations:** Can still suffer from high variance if the majorant is a poor match for the sampled channel's true extinction. The scalar extinction constraint is a workaround but limits the types of media that can be simulated [S3].
*   **When to Use:** Suitable for general participating media with moderate spectral variation. Not ideal for atmospheric scattering where separate transport equations are preferred [S3].

## Background
Spectral Multiple Importance Sampling was developed to address the challenges of rendering spectrally varying participating media [S1, unofficial][S2]. The technique is discussed in the context of volumetric path tracing and multiple importance sampling for wavelengths [S1, unofficial][S2]. The scalar extinction constraint was introduced to handle infinite variance issues in multi-wavelength sampling [S3]. The method is part of the broader field of unbiased global illumination with participating media [S3].

## Sources
[S1] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S2] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S3] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
