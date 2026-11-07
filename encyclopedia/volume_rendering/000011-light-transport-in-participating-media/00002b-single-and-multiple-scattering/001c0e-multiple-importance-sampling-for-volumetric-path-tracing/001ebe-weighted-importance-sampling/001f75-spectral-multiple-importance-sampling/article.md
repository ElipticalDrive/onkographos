# Spectral Multiple Importance Sampling

Also called: Chromatic Media Sampling with MIS, Single-sample MIS for Spectra, Spectral Multiple Importance Sampling (MIS) for Wavelengths, Spectral Volume Sampling

## Overview
Spectral Multiple Importance Sampling (MIS) is a technique for handling spectrally varying participating media in volumetric path tracing. It solves the problem where a single sampling PDF, derived from one wavelength (e.g., using a majorant), is a poor match or invalid (zero-valued) for other wavelengths, leading to high variance or incorrect estimators when evaluating the radiance transfer function across the full spectrum [S2]. It sets itself apart from standard monochromatic null-scattering by introducing channel selection logic and PDF averaging, and from full spectral sampling by reducing per-step computational cost to a single channel's majorant evaluation while correcting the bias via MIS weights [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Current path throughput $\beta$, medium properties at the current position (absorption coefficient $\sigma_a(\lambda)$, scattering coefficient $\sigma_s(\lambda)$, majorant $\sigma_{maj}(\lambda)$), and a random number generator.
*   **Output:** A sampled wavelength index $k \in \{0, 1, 2\}$ (representing R, G, B channels), the PDFs $p_{\lambda_i}(x)$ for all wavelengths $i$, and the MIS weights $w_{\lambda_i}(x)$.

**Data Structures:**
*   `throughput_albedo`: A 3-element vector (or array) of floats, where each element corresponds to the product of the current path throughput and the single-scattering albedo for the respective RGB channel.
    *   $T_i = \beta \cdot \frac{\sigma_s(\lambda_i)}{\sigma_a(\lambda_i) + \sigma_s(\lambda_i)}$ for $i \in \{0, 1, 2\}$.
    *   Note: If $\sigma_a + \sigma_s = 0$, the albedo is undefined; typically handled by clamping or treating as pure scatter/absorb based on context [G].
*   `pmf`: A 3-element vector storing the probability mass function values for each channel.

**Steps:**
1.  **Compute Throughput-Albedo Product:** For each of the $n=3$ spectral channels (RGB), calculate the product of the current path throughput $\beta$ and the single-scattering albedo $\alpha(\lambda_i) = \sigma_s(\lambda_i) / (\sigma_a(\lambda_i) + \sigma_s(\lambda_i))$. Let this vector be $v$, where $v_i = \beta \cdot \alpha(\lambda_i)$.
2.  **Construct Discrete Distribution:** Create a discrete empirical distribution from the vector $v$. This distribution represents the relative importance of sampling each wavelength based on the expected contribution to the radiance.
3.  **Sample Channel:** Use a uniform random number $u \in [0, 1)$ to sample an index $k$ from this discrete distribution. This determines the "primary" wavelength $\lambda_k$ used for the physical sampling step (e.g., free-flight distance calculation).
4.  **Evaluate PDFs:** Calculate the probability density $p_{\lambda_i}(x)$ for each wavelength $\lambda_i$ under the sampling strategy defined by the selected channel $k$. In the context of null-scattering with spectral MIS, the sampling PDF for a specific wavelength is typically derived from the majorant or transmittance at that wavelength. However, since only one channel was sampled for the physical step, the PDFs for the other channels must be estimated or computed based on the underlying distribution used in Step 2. Specifically, the source [S1, unofficial] implies using the `pmf` values from the discrete distribution as the basis for the PDFs.
5.  **Compute MIS Weights:** Apply the balance heuristic to compute the MIS weight for the sampled wavelength $\lambda_k$ and potentially for others if the estimator requires it. The weight for the sampled wavelength $\lambda_k$ is:
    $$ w_{\lambda_k}(x) = \frac{p_{\lambda_k}(x)}{\sum_{i=1}^{n} p_{\lambda_i}(x)} $$
    where $p_{\lambda_i}(x)$ is the probability of sampling the configuration given wavelength $\lambda_i$. In the specific implementation described in [S1, unofficial], the `pmf` values from the discrete distribution serve as these probabilities.
6.  **Apply Weight:** Multiply the radiance contribution by the MIS weight $w_{\lambda_k}(x)$ to correct for the bias introduced by sampling only one wavelength for the physical interaction while evaluating the spectrum.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spectral_mis_sample(throughput, sigma_a, sigma_s, rng):
    """
    Performs Spectral Multiple Importance Sampling for wavelength selection.
    
    Args:
        throughput: float, current path throughput.
        sigma_a: list of 3 floats, absorption coefficients for R, G, B.
        sigma_s: list of 3 floats, scattering coefficients for R, G, B.
        rng: Random number generator.
        
    Returns:
        sampled_channel: int, index of the sampled channel (0, 1, or 2).
        pmf: list of 3 floats, probability mass function for each channel.
        mis_weight: float, the MIS weight for the sampled channel.
    """
    n_channels = 3
    # Step 1: Compute throughput * albedo for each channel
    # Albedo = sigma_s / (sigma_a + sigma_s)
    # Handle division by zero if sigma_a + sigma_s == 0
    throughput_albedo = [0.0] * n_channels
    for i in range(n_channels):
        sigma_total = sigma_a[i] + sigma_s[i]
        if sigma_total > 0:
            albedo = sigma_s[i] / sigma_total
            throughput_albedo[i] = throughput * albedo
        else:
            # If no interaction possible, albedo is 0 or undefined. 
            # Assuming 0 contribution if no scattering.
            throughput_albedo[i] = 0.0
            
    # Step 2: Create discrete distribution from throughput_albedo
    # Normalize to sum to 1
    total = sum(throughput_albedo)
    if total > 0:
        pmf = [t / total for t in throughput_albedo]
    else:
        # Fallback to uniform if all contributions are zero
        pmf = [1.0 / n_channels] * n_channels
        
    # Step 3: Sample a channel index
    u = rng.uniform(0.0, 1.0)
    sampled_channel = sample_discrete(pmf, u)
    
    # Step 4 & 5: Compute MIS weight
    # The weight is p_lambda_k / sum(p_lambda_i)
    # Here, p_lambda_i is approximated by the pmf value for channel i
    # Note: In some implementations, the PDF might be the majorant-based PDF.
    # Based on [S1, unofficial], the pmf from the discrete distribution is used.
    p_sampled = pmf[sampled_channel]
    sum_p = sum(pmf)
    
    if sum_p > 0:
        mis_weight = p_sampled / sum_p
    else:
        mis_weight = 0.0
        
    return sampled_channel, pmf, mis_weight

def sample_discrete(pmf, u):
    """
    Samples an index from a discrete probability mass function.
    """
    cumulative = 0.0
    for i, p in enumerate(pmf):
        cumulative += p
        if u < cumulative:
            return i
    return len(pmf) - 1 # Fallback to last index
```

**Parameters:**
*   $n$: Number of spectral samples. Typically 3 for RGB [S2].
*   $q$: Discrete probability of sampling a specific wavelength. Uniform at $1/n$ [S2].

**Edge Cases and Pitfalls:**
*   **Zero Coefficients:** If $\sigma_a + \sigma_s = 0$ for all channels, the throughput-albedo vector is zero. A fallback (e.g., uniform sampling) is required [G].
*   **Precision:** Floating-point round-off errors in the calculation of albedo or throughput-albedo should be clamped to zero to avoid negative probabilities [S2].
*   **Invalid PDFs:** Using a single wavelength's PDF for other wavelengths can result in zero-valued PDFs, leading to infinite weights or NaNs if not handled by MIS [S2].

## Subtopics

### [Chromatic Null-Scattering Free-Flight Sampling](6634-chromatic-null-scattering-free-flight-sampling.md)
Extends null-scattering to handle chromatic media by sampling a single color channel for distance determination while averaging PDFs across all channels. Chosen when computational cost of full spectral sampling is prohibitive but variance from monochromatic sampling is unacceptable.

## Operating characteristics
*   **Time Complexity:** $O(n)$ per step, where $n$ is the number of spectral channels (typically 3). This is comparable to monochromatic sampling.
*   **Memory Complexity:** $O(n)$ to store the PMF and PDFs.
*   **Strengths:** Reduces variance in spectrally varying media compared to using a single wavelength's PDF for all channels. Maintains low computational cost by only performing one physical sampling step (e.g., free-flight) per interaction.
*   **Limitations:** Introduces bias if not implemented correctly with MIS. Requires careful handling of PDFs for unsampled wavelengths.
*   **When to Use:** When rendering spectrally varying participating media and standard monochromatic sampling leads to high variance or incorrect results due to mismatched PDFs.
*   **When Not to Use:** When the medium is spectrally uniform (monochromatic), as standard sampling is sufficient and simpler.

## Background
The problem of spectrally varying medium properties causing variance in path tracing was identified in the context of volume scattering integrators [S2]. The solution using single-sample MIS with a balance heuristic was proposed to correct the bias introduced by sampling based on one wavelength while evaluating the spectrum at others [S2]. The specific implementation using a discrete empirical distribution based on throughput and albedo is described in an unofficial volumetric path tracer repository [S1, unofficial].

## Sources
[S1] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S2] 14.2 Volume Scattering Integrators, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/Volume_Scattering_Integrators, fetched 2026-10-29
