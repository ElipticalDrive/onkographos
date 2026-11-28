# Amplitude Distribution Estimation

Also called: Analysis Tools for Noise Quality, Noise Amplitude Distribution Estimation, Noise Quality Analysis Tools, Yoon et al. Noise Quality Metrics

## Overview
Amplitude Distribution Estimation is a statistical method used to evaluate the quality of procedural noise functions by comparing the histogram of sampled noise values against a theoretical Gaussian distribution. It solves the problem of verifying whether a noise generator produces values with the correct statistical spread and shape, which is critical for visual realism in volume rendering and optimization procedures. It is distinguished from spatial analysis methods like the [Autocorrelation Periodicity Test](17150-autocorrelation-periodicity-test.md) and [Band-Pass Test for Noise](17151-band-pass-test-for-noise.md) by focusing exclusively on the fidelity of amplitude values rather than spatial frequency characteristics or periodicity [S1, S2].

## How it works
**Inputs:**
- A procedural noise function $N(\mathbf{P})$ capable of evaluating noise at any point $\mathbf{P}$ in the domain.
- A set of query points $\{\mathbf{P}_1, \mathbf{P}_2, \dots, \mathbf{P}_M\}$ sampled from the domain.
- The expected variance $\sigma^2_{expected}$ of the noise function, if known analytically (e.g., for standard Perlin noise). If unknown, the variance is estimated from the samples.
- The number of histogram bins $B$.

**Outputs:**
- A histogram $H$ representing the frequency distribution of the sampled noise values.
- A reference Gaussian curve $G(x)$ defined by the mean $\mu$ and variance $\sigma^2$ (either expected or estimated).

**Data Structures:**
- **Histogram Array**: An array of size $B$ storing integer counts for each bin.
- **Sample Array**: An array of size $M$ storing the scalar noise values $v_i = N(\mathbf{P}_i)$.
- **Gaussian Parameters**: Scalar values for mean $\mu$ and variance $\sigma^2$.

**Steps:**
1. **Sampling**: Evaluate the noise function $N(\mathbf{P})$ at $M$ distinct points $\mathbf{P}_i$. The points should be distributed uniformly across the domain of interest to avoid bias.
   $$ v_i = N(\mathbf{P}_i) \quad \text{for } i = 1 \dots M $$
2. **Variance Estimation (if needed)**: If $\sigma^2_{expected}$ is not provided, calculate the sample variance $\sigma^2_{est}$ from the values $v_i$:
   $$ \mu = \frac{1}{M} \sum_{i=1}^{M} v_i $$
   $$ \sigma^2_{est} = \frac{1}{M-1} \sum_{i=1}^{M} (v_i - \mu)^2 $$
   Let $\sigma^2$ be $\sigma^2_{expected}$ if known, otherwise $\sigma^2_{est}$.
3. **Histogram Construction**: Initialize an array $H$ of size $B$ with zeros. Determine the range $[min\_v, max\_v]$ of the sampled values $v_i$. Map each value $v_i$ to a bin index $b_i$:
   $$ b_i = \left\lfloor \frac{v_i - min\_v}{max\_v - min\_v} \times B \right\rfloor $$
   Clamp $b_i$ to $[0, B-1]$. Increment $H[b_i]$.
4. **Reference Gaussian Calculation**: Define the probability density function (PDF) of a Gaussian distribution with mean $\mu$ and variance $\sigma^2$:
   $$ G(x) = \frac{1}{\sqrt{2\pi\sigma^2}} e^{-\frac{(x-\mu)^2}{2\sigma^2}} $$
   Evaluate $G(x)$ at the center of each histogram bin to generate the reference curve.
5. **Comparison**: Plot the normalized histogram (counts divided by $M \times \text{bin\_width}$) alongside the reference Gaussian curve $G(x)$. Visual inspection or a statistical test (such as the [Chi-Square Goodness-of-Fit Test](17149-chi-square-goodness-of-fit-test-for-noise.md)) determines the quality.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def estimate_amplitude_distribution(noise_func, points, num_bins, expected_variance=None):
    """
    Estimates the amplitude distribution of a noise function.
    
    Args:
        noise_func: Function taking a point P and returning a scalar noise value.
        points: List of points P to sample.
        num_bins: Number of bins for the histogram.
        expected_variance: Optional float. If None, variance is estimated from samples.
        
    Returns:
        histogram: List of counts for each bin.
        gaussian_ref: List of Gaussian PDF values for each bin center.
        bin_centers: List of x-coordinates for the bin centers.
    """
    # Step 1: Sampling
    samples = [noise_func(p) for p in points]
    m = len(samples)
    
    if m == 0:
        return [], [], []

    # Step 2: Variance Estimation
    mean_val = sum(samples) / m
    
    if expected_variance is not None:
        variance = expected_variance
    else:
        # Sample variance
        variance = sum((v - mean_val) ** 2 for v in samples) / (m - 1)
        
    # Guard against zero variance (constant noise)
    if variance == 0:
        variance = 1e-9 

    # Step 3: Histogram Construction
    min_val = min(samples)
    max_val = max(samples)
    
    # Handle edge case where all samples are identical
    if min_val == max_val:
        histogram = [0] * num_bins
        histogram[0] = m
        # Gaussian reference at the constant value
        gaussian_ref = [0] * num_bins
        gaussian_ref[0] = 1.0 / math.sqrt(2 * math.pi * variance)
        bin_centers = [min_val] * num_bins
        return histogram, gaussian_ref, bin_centers

    bin_width = (max_val - min_val) / num_bins
    histogram = [0] * num_bins
    
    for v in samples:
        # Map value to bin index
        bin_idx = int((v - min_val) / bin_width)
        # Clamp to last bin for max_val
        if bin_idx >= num_bins:
            bin_idx = num_bins - 1
        histogram[bin_idx] += 1

    # Step 4: Reference Gaussian Calculation
    gaussian_ref = []
    bin_centers = []
    
    for i in range(num_bins):
        # Center of the bin
        center = min_val + (i + 0.5) * bin_width
        bin_centers.append(center)
        
        # Gaussian PDF
        exponent = -((center - mean_val) ** 2) / (2 * variance)
        pdf_val = (1.0 / math.sqrt(2 * math.pi * variance)) * math.exp(exponent)
        gaussian_ref.append(pdf_val)

    return histogram, gaussian_ref, bin_centers
```

**Parameters:**
- $M$ (Number of samples): Not given in the sources. Typically large enough to form a smooth histogram (e.g., $10^4$ to $10^6$).
- $B$ (Number of bins): Not given in the sources. Typically 32–128 for visual comparison.

**Edge Cases and Pitfalls:**
- **Zero Variance**: If the noise function is constant, variance is zero. The Gaussian PDF becomes undefined. A guard value or special handling is required [G].
- **Boundary Bias**: If the noise function has hard clamping or non-periodic boundaries, samples near the edges may skew the distribution. Uniform sampling over the interior is preferred [G].
- **Anisotropy**: Some noise functions (e.g., standard Perlin) exhibit axis-aligned anisotropy, which may affect the uniformity of samples if the sampling grid is aligned with the axes [S3].

## Subtopics

### [Chi-Square Goodness-of-Fit Test for Noise](17149-chi-square-goodness-of-fit-test-for-noise.md)
A statistical method that quantifies the deviation between the observed histogram and the expected Gaussian distribution using the Chi-Square statistic. It provides a numerical quality score rather than just a visual plot.

### [Autocorrelation Periodicity Test](17150-autocorrelation-periodicity-test.md)
A method that analyzes the spatial correlation of noise samples to detect repeating patterns. It complements amplitude estimation by checking for spatial randomness.

### [Band-Pass Test for Noise](17151-band-pass-test-for-noise.md)
A method that measures the spectral bandwidth of the noise by comparing the noise to a down-up-sampled version. It ensures the noise has the correct frequency content, which is distinct from amplitude distribution.

### [Bartlett’s Method for Power Spectrum Estimation](17459-bartlett-s-method-for-power-spectrum-estimation.md)
A frequency-domain method that averages periodograms to estimate the power spectrum. It is used for spectral analysis rather than amplitude distribution.

## Operating characteristics
- **Time Complexity**: $O(M)$, where $M$ is the number of samples. Each sample requires one noise evaluation and constant-time histogram binning.
- **Memory Complexity**: $O(B + M)$, where $B$ is the number of histogram bins and $M$ is the number of samples (to store the samples if needed for variance calculation).
- **Strengths**: Simple to implement; provides immediate visual feedback on the statistical quality of the noise; effective for detecting non-Gaussian artifacts in noise generators.
- **Limitations**: Does not detect spatial artifacts (periodicity, anisotropy) on its own; requires a large number of samples for accurate histogram estimation; sensitive to the choice of bin count [S1, S2].
- **When to use**: During the development and optimization of new noise functions to ensure they match the expected statistical properties (e.g., Gaussian amplitude distribution) [S2].
- **When not to use**: When spatial coherence or frequency content is the primary concern; use [Band-Pass Test](17151-band-pass-test-for-noise.md) or [Autocorrelation Test](17150-autocorrelation-periodicity-test.md) instead.

## Background
Amplitude distribution estimation is part of a suite of analysis tools presented by Yoon et al. [YLC04, YL08] for measuring the quality of procedural noise within optimization procedures [S2]. These tools include the Chi-Square goodness-of-fit test, autocorrelation periodicity detection, and band-pass tests [S1, S2]. The specific method of estimating amplitude distribution via histogram comparison with a Gaussian reference is described in surveys of procedural noise functions [S1, S2, S3]. It is used to verify that noise functions like Perlin noise, sparse convolution noise, and wavelet noise exhibit the expected statistical properties [S3].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
