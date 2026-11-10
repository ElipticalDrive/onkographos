# Weighted Importance Sampling


## Overview
Weighted Importance Sampling (WIS) is a biased but consistent technique used to simulate the use of a known target probability density function (PDF) $p$ with samples drawn from an arbitrary source PDF $q$ [S1]. In volumetric path tracing, it allows the estimator to benefit from a target distribution that is well-matched to the integrand, even when the actual sampling process follows a different source distribution [S1]. It sets itself apart from standard Multiple Importance Sampling (MIS) by not requiring the source PDF to be a mixture component of the target, and it differs from Resampled Importance Sampling (RIS) by avoiding the intractable resampling of path prefixes in unbounded scenarios [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of samples drawn from a source PDF $q(x)$. A target PDF $p(x)$ that is known and desirable for the integrand.
*   **Output:** An unbiased estimate of the integral $\int f(x) dx$ (biased in finite samples, consistent as $N \to \infty$).

**Data Structures:**
No specific complex data structures are required beyond storing the sample $x$, the source weight $q(x)$, and the target weight $p(x)$.

**Method:**
1.  For each sample $x_i$ drawn from the source distribution $q$:
2.  Evaluate the integrand $f(x_i)$.
3.  Evaluate the source PDF $q(x_i)$ and the target PDF $p(x_i)$.
4.  Compute the importance weight $w_i$ as the ratio of the target PDF to the source PDF:
    $$ w_i = \frac{p(x_i)}{q(x_i)} $$
5.  The estimator for the integral is the average of the weighted function values:
    $$ \hat{I} = \frac{1}{N} \sum_{i=1}^{N} w_i f(x_i) = \frac{1}{N} \sum_{i=1}^{N} \frac{p(x_i)}{q(x_i)} f(x_i) $$

**Parameters:**
*   $N$: Number of samples.
*   $p(x)$: Target PDF. Must be non-zero wherever $f(x)q(x)$ is non-zero.
*   $q(x)$: Source PDF. Must be non-zero wherever $p(x)$ is non-zero (support condition).

**Edge Cases and Pitfalls:**
*   **Bias:** WIS is biased for finite $N$ because the weights are not normalized by their sum. It becomes unbiased only in the limit as $N \to \infty$ [S1].
*   **Variance:** If $q(x)$ is very small where $p(x)$ is large, the weight $w_i$ becomes very large, leading to high variance.
*   **Zero Division:** If $q(x_i) = 0$ for any sample where $p(x_i) > 0$, the weight is undefined. The source distribution must cover the support of the target distribution.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def weighted_importance_sampling(samples, source_pdf, target_pdf, integrand):
    """
    Estimates the integral using Weighted Importance Sampling.
    
    Args:
        samples: List of sample points x_i drawn from source distribution q.
        source_pdf: Function q(x) evaluating the source PDF.
        target_pdf: Function p(x) evaluating the target PDF.
        integrand: Function f(x) evaluating the integrand.
        
    Returns:
        Estimate of the integral.
    """
    n = len(samples)
    if n == 0:
        return 0.0
    
    weighted_sum = 0.0
    for x in samples:
        q_val = source_pdf(x)
        p_val = target_pdf(x)
        
        # Guard against division by zero
        if q_val == 0:
            # If source PDF is zero but target is not, this sample is invalid
            # for WIS. In practice, ensure q covers p's support.
            continue
            
        weight = p_val / q_val
        f_val = integrand(x)
        weighted_sum += weight * f_val
        
    return weighted_sum / n
```

## Subtopics

### Self-Normalizing Multi-Strategy Estimator
An estimator that combines MIS and WIS weights to benefit from both techniques [S1]. It allows samples with a low source PDF but a high target PDF to contribute to the estimate without increasing variance excessively [S1]. This approach escapes Veach's theoretical bounds for optimal weights because poor samples (low source PDF) do not increase the variance of the estimator if the target PDF $p$ is well-chosen [S1].

## Operating characteristics
*   **Bias:** WIS is biased for finite sample counts but is consistent (converges to the correct value as $N \to \infty$) [S1].
*   **Variance:** Variance can be high if the source PDF $q$ is a poor match for the target PDF $p$, specifically if $q$ is small where $p$ is large [S1].
*   **Comparison to RIS:** Resampled Importance Sampling (RIS) is a bias-corrected version of WIS [S1]. However, RIS requires resampling path prefixes, which is intractable for unbounded paths in volumetric rendering [S1]. WIS is preferred in this context despite its bias because it avoids the resampling step.
*   **Use Case:** Use WIS when you have a fixed set of samples drawn from a source distribution $q$ and you want to evaluate the integral as if they were drawn from a better target distribution $p$, particularly when resampling is not feasible.

## Background
Weighted Importance Sampling has been used in rendering applications for many years [S1]. It was proposed to address the issue of shifted paths not following the base PDF (the ideal PDF for the integrand) [S1]. Bitterli et al. demonstrated that Resampled Importance Sampling (RIS) is a bias-corrected version of WIS [S1]. The technique is cited with references to [HHM18], [BSW00], and [Kel96] in the context of rendering applications [S1].

## Sources
[S1] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
