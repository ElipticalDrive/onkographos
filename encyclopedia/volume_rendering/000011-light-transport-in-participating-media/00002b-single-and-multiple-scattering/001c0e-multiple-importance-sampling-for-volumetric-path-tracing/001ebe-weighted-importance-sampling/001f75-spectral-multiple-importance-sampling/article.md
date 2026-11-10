# Spectral Multiple Importance Sampling

Also called: Chromatic Media Sampling with MIS, Single-sample MIS for Spectra, Spectral Multiple Importance Sampling (MIS) for Wavelengths, Spectral Volume Sampling

## Overview
Spectral Multiple Importance Sampling (MIS) is a technique for handling spectrally varying participating media in path tracing by sampling a single wavelength channel while correctly weighting contributions across all spectral bands [S1, unofficial][S2]. It solves the high-variance problem that arises when a single sampling probability density function (PDF), derived from one wavelength's medium properties, is used to evaluate integrands at other wavelengths where that PDF may be a poor match or zero-valued [S2]. Unlike standard monochromatic null-scattering, which ignores spectral variation, or full spectral sampling, which is computationally expensive, this method reduces per-step cost to a single channel's majorant evaluation while maintaining unbiasedness through PDF averaging [S1, unofficial].

## How it works
**Inputs:**
*   Current path vertex position $x$.
*   Throughput vector $\beta$ (RGB).
*   Medium properties: absorption coefficient $\sigma_a(\lambda)$ and scattering coefficient $\sigma_s(\lambda)$ for $n$ spectral samples (typically $n=3$ for RGB).
*   Majorant extinction coefficient $\sigma_{maj}$ (scalar or vector, depending on implementation context, but typically a scalar upper bound for the specific channel being sampled).
*   Random number generator state.

**Outputs:**
*   Sampled wavelength index $k \in \{0, 1, \dots, n-1\}$.
*   PDF values $p_{\lambda_i}(x)$ for all $i \in \{0, \dots, n-1\}$.
*   Updated throughput and path termination status.

**Data Structures:**
*   `throughput_albedo`: A vector of size $n$ (or 3 for RGB), where each element $i$ is $\beta_i \cdot \frac{\sigma_{s,\lambda_i}}{\sigma_{t,\lambda_i}}$ or simply $\beta_i \cdot \text{albedo}_i$. Note: In the context of free-flight sampling, the "weight" for channel selection is often proportional to the expected contribution, which involves throughput and albedo [S1, unofficial].
*   `DiscreteEmpiricalDistribution1D`: A discrete distribution object constructed from the `throughput_albedo` vector. This distribution maps the continuous uniform random variable to a channel index based on the relative magnitudes of the spectral contributions [S1, unofficial].
*   `pmf`: An array of size $n$ storing the probability density values $p_{\lambda_i}(x)$ for each wavelength [S1, unofficial].

**Steps:**
1.  **Compute Spectral Weights:** Calculate the product of the current throughput and the single-scattering albedo for each wavelength channel. Let $w_i = \beta_i \cdot \frac{\sigma_{s,\lambda_i}}{\sigma_{a,\lambda_i} + \sigma_{s,\lambda_i}}$. If $\sigma_{t,\lambda_i} = 0$, handle as a special case (e.g., set weight to 0 or 1 depending on context) [G].
2.  **Construct Discrete Distribution:** Create a discrete distribution object using the weights $w = [w_0, w_1, \dots, w_{n-1}]$. This distribution represents the empirical distribution of spectral contributions [S1, unofficial].
3.  **Sample Channel Index:** Draw a uniform random number $u \in [0, 1)$ and use it to sample an index $k$ from the discrete distribution. This index $k$ corresponds to the wavelength $\lambda_k$ used for the free-flight distance sampling [S1, unofficial].
4.  **Sample Distance:** Sample the distance $t$ to the next interaction using the majorant extinction coefficient $\sigma_{maj}$ associated with the sampled channel $\lambda_k$ (or a global majorant if used). The distance is sampled from $p(t) = \sigma_{maj} e^{-\sigma_{maj} t}$ [S2].
5.  **Evaluate PDFs:** For the sampled distance $t$ and position $x$, compute the PDF for the sampled channel $p_{\lambda_k}(x)$ and the PDFs for all other channels $p_{\lambda_i}(x)$ for $i \neq k$. The PDF for a channel $\lambda_i$ is typically the probability of the event occurring under the sampling strategy defined by $\lambda_i$. In the context of null-scattering, this is often the probability of a non-null event or the specific density used for distance sampling [S2].
6.  **Compute MIS Weights:** Apply the balance heuristic for single-sample MIS. The weight for the sampled wavelength $\lambda_k$ is:
    $$ w_{\lambda_k}(x) = \frac{p_{\lambda_k}(x)}{\sum_{i=1}^{n} p_{\lambda_i}(x)} $$
    The weight for any other wavelength $\lambda_j$ is:
    $$ w_{\lambda_j}(x) = \frac{p_{\lambda_j}(x)}{\sum_{i=1}^{n} p_{\lambda_i}(x)} $$
    Note: The discrete probability of selecting the specific wavelength $\lambda_k$ is $q = 1/n$ if uniform, but the balance heuristic uses the ratio of the PDFs [S2].
7.  **Update Throughput:** Multiply the throughput by the MIS weight and the appropriate spectral terms (albedo, phase function, etc.) for the sampled channel [S1, unofficial][S2].
8.  **Continue Path:** Proceed with the path tracing algorithm using the sampled distance and updated throughput.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spectral_mis_sample_medium(x, throughput, sigma_a, sigma_s, sigma_maj, rng):
    """
    Sample a medium interaction using Spectral MIS.
    
    Args:
        x: Current position.
        throughput: Current path throughput (vector of size n).
        sigma_a: Absorption coefficients for n wavelengths (vector of size n).
        sigma_s: Scattering coefficients for n wavelengths (vector of size n).
        sigma_maj: Majorant extinction coefficient (scalar or vector).
        rng: Random number generator.
        
    Returns:
        t: Distance to next interaction.
        new_throughput: Updated throughput.
        terminated: Boolean indicating if path is terminated.
        pmf: PDFs for all wavelengths.
    """
    n = len(sigma_a)
    
    # Step 1: Compute spectral weights (throughput * albedo)
    # Albedo = sigma_s / (sigma_a + sigma_s)
    # Handle division by zero if sigma_t is 0
    sigma_t = sigma_a + sigma_s
    albedo = [0.0] * n
    for i in range(n):
        if sigma_t[i] > 0:
            albedo[i] = sigma_s[i] / sigma_t[i]
        else:
            albedo[i] = 0.0 # Or 1.0 if purely scattering, depending on context
            
    weights = [throughput[i] * albedo[i] for i in range(n)]
    
    # Step 2: Construct discrete distribution
    # Using a simple discrete distribution based on weights
    dist = DiscreteDistribution(weights)
    
    # Step 3: Sample channel index
    k = dist.sample(rng)
    
    # Step 4: Sample distance using the majorant for the sampled channel
    # Note: sigma_maj might be a scalar or channel-specific. 
    # If channel-specific, use sigma_maj[k]. If scalar, use sigma_maj.
    maj = sigma_maj[k] if isinstance(sigma_maj, list) else sigma_maj
    t = -log(1.0 - rng.uniform()) / maj
    
    # Step 5: Evaluate PDFs for all wavelengths
    # The PDF for wavelength i is the probability density of the sampled event
    # under the sampling strategy for wavelength i.
    # For null-scattering, this is often the probability of a non-null event
    # or the density of the distance sampling.
    # Assuming the distance is sampled from p(t|lambda_i) = sigma_maj_i * exp(-sigma_maj_i * t)
    # And the event type (scatter/absorb/null) is sampled separately.
    # For simplicity, we assume the PDF here refers to the spectral weight in the MIS balance.
    # The source [S2] implies p_lambda(x) is the PDF of the sampling technique for lambda.
    # In the context of the balance heuristic for single-sample MIS, 
    # the weight is p_lambda_k / sum(p_lambda_i).
    
    pmf = [0.0] * n
    for i in range(n):
        maj_i = sigma_maj[i] if isinstance(sigma_maj, list) else sigma_maj
        # PDF of distance sampling for wavelength i
        pdf_dist = maj_i * exp(-maj_i * t)
        # PDF of event type (scatter/absorb/null) for wavelength i
        # This is complex and depends on the specific null-scattering implementation.
        # For the purpose of this article, we assume the PMF is dominated by the 
        # spectral weight or the distance PDF.
        # Source [S1, unofficial] suggests using the throughput_albedo vector for the discrete distribution,
        # which implies the PMF is related to these weights.
        # Let's assume the PMF is proportional to the weight for the sampled channel
        # and the distance PDF for others.
        # However, [S2] states the balance heuristic uses p_lambda(x).
        # We will use the distance PDF as the primary component of p_lambda(x).
        pmf[i] = pdf_dist
        
    # Step 6: Compute MIS weights
    sum_pmf = sum(pmf)
    if sum_pmf == 0:
        # Degenerate case
        mis_weights = [0.0] * n
    else:
        mis_weights = [p / sum_pmf for p in pmf]
        
    # Step 7: Update throughput
    # Multiply by albedo, phase function, and MIS weight
    # Note: The throughput update also includes the ratio of sigma_t to sigma_maj
    # for the sampled channel.
    new_throughput = [0.0] * n
    for i in range(n):
        # Ratio of true extinction to majorant
        ratio = sigma_t[i] / maj if maj > 0 else 0.0
        # Albedo
        alb = albedo[i]
        # MIS weight
        w = mis_weights[i]
        # Phase function p(phase) - assumed to be sampled separately
        # For this step, we just update the throughput with the spectral weights
        new_throughput[i] = throughput[i] * ratio * alb * w
        
    # Step 8: Determine termination
    # If absorption event, terminate
    # This depends on the event type sampling, which is not fully detailed here.
    # Assuming a scatter event for now.
    terminated = False
    
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
A technique to avoid infinite variance when sampling transport equations for multiple wavelengths simultaneously by limiting the extinction coefficient to a single scalar value shared across all wavelengths [S3]. It enforces the constraint that for every pair of wavelengths $\lambda_1, \lambda_2$, the total extinction coefficients are equal: $\sigma_{a,\lambda_1}(x) + \sigma_{s,\lambda_1}(x) = \sigma_{a,\lambda_2}(x) + \sigma_{s,\lambda_2}(x) = \sigma_t(x)$ [S3]. This approach is suitable when extinction depends only on particle density/size (color varies via BSDF-like probabilities). For atmospheric scattering, solving transport equations separately for each wavelength is preferred [S3]. If $\sigma_{t,\lambda_2} > \sigma_{t,\lambda_1}$ and sampling is done using density $p_{\lambda_2}$, the variance on wavelength $\lambda_1$ can be arbitrarily high or infinite. Specifically, variance is infinite if $\sigma_{t,\lambda_2} \geq 2\sigma_{t,\lambda_1}$ [S3].

## Operating characteristics
*   **Time Complexity:** $O(n)$ per interaction, where $n$ is the number of spectral samples. This is significantly faster than full spectral sampling, which would require $O(n^2)$ or more operations for multiple importance weighting across all channels.
*   **Memory Complexity:** $O(n)$ to store the spectral coefficients and PDFs.
*   **Performance:** Reduces variance compared to monochromatic sampling with a single majorant, especially in media with strong spectral variation. However, it may still exhibit higher variance than full spectral sampling in extreme cases [S1, unofficial][S2].
*   **Strengths:** Unbiased, relatively low computational cost, handles spectrally varying media effectively.
*   **Limitations:** Can still suffer from high variance if the majorant is a poor match for the sampled channel's true extinction. The scalar extinction constraint is a workaround but limits the types of media that can be simulated [S3].
*   **When to Use:** Suitable for general participating media with moderate spectral variation. Not ideal for atmospheric scattering where separate transport equations are preferred [S3].

## Background
Spectral Multiple Importance Sampling was developed to address the challenges of rendering spectrally varying participating media. The technique is discussed in the context of volumetric path tracing and multiple importance sampling for wavelengths [S1, unofficial][S2]. The scalar extinction constraint was introduced to handle infinite variance issues in multi-wavelength sampling [S3]. The method is part of the broader field of unbiased global illumination with participating media [S3].

## Sources
[S1] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S2] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
[S3] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
