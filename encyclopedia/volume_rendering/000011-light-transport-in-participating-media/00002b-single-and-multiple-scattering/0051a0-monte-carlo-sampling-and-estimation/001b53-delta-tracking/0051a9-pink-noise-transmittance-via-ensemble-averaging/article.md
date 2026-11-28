# Pink Noise Transmittance via Ensemble Averaging

Also called: Delta Tracking for Pink Noise, Stochastic Delta Tracking

## Overview
This method computes the transmittance of participating media characterized by 1/f (pink) fractal noise by interpreting the macroscopic transmittance as an ensemble average of random homogeneous realizations. It allows the use of classical, unbiased delta tracking for non-exponential media, whereas standard delta tracking relies on transmittance being exponential ($T_r(a+b) = T_r(a)T_r(b)$) [S1]. It is distinct from regular tracking (which is slow) and raymarching (which is biased) by providing an unbiased estimator that can be significantly faster for specific noise distributions [S1].

## How it works
**Inputs:**
*   Ray segment length $t$ (distance along the ray).
*   Noise color parameter $\beta$ (must be 1 for pink noise).
*   Probability density function (PDF) of the extinction coefficient scaling factor, $\text{pdf}(\bar{\sigma}_\mu)$.
*   Number of ensemble samples $N$ (optional, for Monte Carlo approximation).

**Outputs:**
*   Estimated transmittance $f_{\text{pink}}(t)$.

**Data Structures:**
*   The method relies on sampling from the distribution $\text{pdf}(\bar{\sigma}_\mu)$. For a Gamma distributed pink noise medium, this is $\Gamma(\alpha, \alpha)$ where $\alpha = C^{-\beta-1}$ [S1]. The exact layout of the PDF/CDF evaluators is implementation-dependent but must support sampling and evaluation [S1].

**Steps:**
1.  **Check Noise Color:** Verify that the medium is characterized by pink noise ($\beta = 1$). If $\beta \neq 1$ (e.g., blue noise where $\beta = -1$), the scale-invariant property does not hold, and this method is not applicable [S1].
2.  **Sample Scaling Factor:** Draw a sample $\bar{\sigma}_{\mu}$ from the distribution $\text{pdf}(\bar{\sigma}_\mu)$ [S1].
3.  **Compute Homogeneous Transmittance:** Calculate the transmittance for a homogeneous medium with extinction coefficient scaled by $\bar{\sigma}_{\mu}$. This is given by $e^{-t\bar{\sigma}_{\mu}}$ [S1].
4.  **Ensemble Average (Monte Carlo):** If using the Monte Carlo approximation, repeat steps 2-3 for $N$ samples and average the results:
    $$ f_{\text{pink}}(t) \approx \frac{1}{N} \sum_{i=1}^{N} e^{-t\bar{\sigma}_{\mu_i}} $$
    where $\bar{\sigma}_{\mu_i}$ are drawn proportional to $\text{pdf}(\bar{\sigma}_\mu)$ [S1].

**Formulas:**
The exact transmittance for pink noise is the characteristic function of the extinction coefficient distribution:
$$ f_{\text{pink}}(t) = \int_0^\infty e^{-t\bar{\sigma}_\mu} \text{pdf}(\bar{\sigma}_\mu) \, d\bar{\sigma}_\mu = \varphi_{\bar{\sigma}_\mu}(t) \quad [S1] $$

For the Gamma model with $\beta=1$, the PDF is $\text{pdf}(\sigma_\mu) = \Gamma(\mu_\sigma, \alpha)$ where $\alpha = C^{-\beta-1}$ [S1]. The resulting transmittance function is:
$$ f(t) = \left(1 + \frac{t}{\alpha}\right)^{-\alpha} \quad [S1] $$

**Parameters:**
*   $\beta = 1$: Defines pink noise. This value ensures the distribution $\text{pdf}(\sigma_\mu)$ is independent of distance $t$ [S1].
*   $N$: Number of samples for Monte Carlo estimation. Not given in the sources.

**Pitfalls:**
*   **Restricted Applicability:** This method is strictly restricted to media characterized by 1/f (pink) fractal noise. It does not work for other noise colors like blue noise ($\beta = -1$) because they lack the scale-invariant property where $\text{pdf}(\sigma_\mu)$ is independent of $t$ [S1].
*   **Delta Tracking Requirement:** While the transmittance is non-exponential, the method enables the use of *unbiased* delta tracking by treating the heterogeneity as a random scaling of a homogeneous medium. Standard delta tracking cannot be applied directly to the heterogeneous medium because its transmittance is not exponential [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pink_noise_transmittance(t, beta, pdf_sigma_mu, N=1):
    """
    Compute transmittance for pink noise media using ensemble averaging.
    
    Args:
        t: Distance along the ray.
        beta: Noise color parameter. Must be 1 for pink noise.
        pdf_sigma_mu: A callable or sampler that draws samples from the 
                      distribution pdf(sigma_mu).
        N: Number of ensemble samples.
    
    Returns:
        Estimated transmittance.
    """
    if beta != 1:
        raise ValueError("Method restricted to pink noise (beta=1).")
    
    total_transmittance = 0.0
    
    for _ in range(N):
        # Sample a scaling factor from the distribution
        sigma_mu = pdf_sigma_mu.sample()
        
        # Compute homogeneous transmittance with scaled extinction
        # T = exp(-t * sigma_mu)
        transmittance = math.exp(-t * sigma_mu)
        
        total_transmittance += transmittance
        
    if N > 0:
        return total_transmittance / N
    else:
        return 0.0
```

## Subtopics

### Gamma Distributed Pink Noise
A specific instance where the PDF is Gamma-distributed, leading to a closed-form transmittance $(1 + t/\alpha)^{-\alpha}$. It is chosen when the medium's heterogeneity follows this specific statistical model. [Link](35-single-and-multiple-scattering.md)

### Stochastic Delta Tracking
The broader technique of using delta tracking on media with random properties. Pink noise is a key case where this yields unbiased results efficiently. [Link](35-single-and-multiple-scattering.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of ensemble samples. Each sample involves a PDF draw and an exponential evaluation.
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample.
*   **Measured Performance:** In a cloud with Gamma distributed pink noise, delta tracking rendered 23x faster than regular tracking while maintaining comparable noise levels and unbiased results [S1].
*   **Strengths:** Unbiased estimation; enables fast delta tracking for non-exponential media; simple implementation if PDF sampling is available.
*   **Limitations:** Only applicable to pink noise ($\beta=1$); requires sampling from the specific PDF of the extinction coefficient distribution; variance depends on $N$.
*   **When to use:** When rendering participating media with 1/f fractal noise characteristics and unbiased results are required.
*   **When not to use:** For media with other noise colors (e.g., blue noise) or when exponential transmittance is sufficient.

## Background
The method originates from the work on radiative transfer frameworks for non-exponential media [S1]. It leverages the mathematical property that the transmittance of pink noise media is the characteristic function of the extinction coefficient distribution. This allows the problem to be decomposed into an ensemble of homogeneous problems, which can be solved efficiently using established unbiased techniques like delta tracking [S1]. The approach connects to fractal literature via the Hurst parameter $H = (\beta - 1)/2$ [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
